---
description: Measure and diagnose LCP (Largest Contentful Paint) / web performance using Playwright — mobile emulation, network/CPU throttling, resource blocking, LCP breakdown, CPU profiling. Use when the user wants to audit load speed / LCP / Core Web Vitals for a specific URL.
---

You are performing an LCP/performance audit for URL: `$ARGUMENTS`

If the user didn't provide a URL, ask for it before starting.

All code below runs through the `browser_run_code_unsafe` tool from the Playwright MCP server bundled with this plugin (written as `async (page) => { ... }`). If that tool isn't available, tell the user the `playwright` MCP server (declared in this plugin's `.mcp.json`) may need to be approved/enabled first.

## Reference audit techniques

These are available techniques to use as needed, not a mandatory ordered checklist. Assess the actual situation of the page being audited to decide which steps to run, which to skip, or in what order:

- **Measure baseline** — median of 3-4 runs, under both conditions: Slow 4G + CPU throttle 4x (to surface the bottleneck clearly) and Regular 4G (to know the real-world experience). Usually worth doing first to have a comparison point, but can be skipped if the user already provided baseline numbers.
- **Block suspect resource groups** (third-party, by group) — always check that `lcpUrl` is the same element before/after comparing LCP. Skip this if there's no suspicious third-party, or if the breakdown already points straight to the cause.
- **LCP breakdown** (section 4) — use to confirm whether the cause is `resourceLoadTime` (heavy image/bandwidth) or `elementRenderDelay` (JS delaying render) when you need to distinguish between these two very different causes. Can be run first if you want a quick overview before digging into individual resources.
- **CPU Profiling** (section 6) — use when you suspect JS is blocking the main thread and need real CPU numbers instead of guessing from network behavior. If JS-blocking is the obvious suspicion from the start, this can run before the breakdown.
- **Cross-check PSI field data** (section 7) — usually best done last since it's the metric that decides whether further optimization is even worth it, but can be checked early to gauge severity before investing time in deep investigation.

## Final report

When the investigation is done, write the findings to a markdown file instead of only summarizing in chat. Get today's date with the `date +%F` shell command, then write to `./reports/lcp-audit-<domain>-<date>.md` (create the `reports/` directory if it doesn't exist), using this structure — omit any section you didn't actually run:

```markdown
# LCP Audit Report — <domain>

- URL: <full URL audited>
- Date: <YYYY-MM-DD>
- Conditions: Mobile 390x844 DPR3, Slow 4G + CPU throttle 4x (and/or Regular 4G)

## Baseline
- LCP (median of N runs): <value>
- TBT (median of N runs): <value>
- LCP element: <lcpUrl>

## LCP Breakdown
- TTFB: <value>
- Resource load delay: <value>
- Resource load time: <value>
- Element render delay: <value>

## Resource blocking results
<table or list of what was blocked and the resulting LCP/TBT, only if this step was run>

## CPU profiling results
<top offending scripts by self time, only if this step was run>

## PSI field data
<CrUX field data LCP, only if this step was checked>

## Root cause
<the confirmed root cause(s), referencing which measurement proved it>

## Recommendations
<concrete, actionable fixes, one per root cause>
```

---

## 1. Base setup: mobile emulation + network/CPU throttle

Create a fresh `context` for each measurement (isolates cache/cookies), enable a CDP session to control network/CPU throttling. Always use a new `context.newPage()` — never reuse an existing `page` — to avoid `addInitScript` stacking across runs and to avoid stale cache/cookies skewing the measurement.

```js
async (page) => {
  const URL = '<TARGET_URL>'; // replace with the URL to measure

  const browser = page.context().browser();
  const context = await browser.newContext({
    viewport: { width: 390, height: 844 },       // common mobile screen size
    deviceScaleFactor: 3,                         // high DPR (like a modern Android/iPhone)
    isMobile: true,
    hasTouch: true,
    userAgent: 'Mozilla/5.0 (Linux; Android 12; Pixel 6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Mobile Safari/537.36'
  });
  const p = await context.newPage();
  const cdp = await context.newCDPSession(p);

  await cdp.send('Network.enable');
  await cdp.send('Network.setCacheDisabled', { cacheDisabled: true }); // always disable cache for a "cold load"
  await cdp.send('Network.clearBrowserCache');

  // Profile (a) Slow 4G — harsh, used to surface a clear bottleneck
  await cdp.send('Network.emulateNetworkConditions', {
    offline: false,
    latency: 150,
    downloadThroughput: (1.6 * 1024 * 1024) / 8,
    uploadThroughput: (0.75 * 1024 * 1024) / 8
  });

  // Profile (b) Regular 4G — cross-check against real-world experience:
  // await cdp.send('Network.emulateNetworkConditions', {
  //   offline: false, latency: 20,
  //   downloadThroughput: (4 * 1024 * 1024) / 8,
  //   uploadThroughput: (3 * 1024 * 1024) / 8
  // });

  await cdp.send('Emulation.setCPUThrottlingRate', { rate: 4 }); // 4x = Lighthouse mobile default

  await p.goto(URL, { waitUntil: 'load', timeout: 45000 });
  await p.waitForTimeout(4000); // extra wait so LCP/long tasks have time to fire

  await context.close();
}
```

---

## 2. Measure LCP + Total Blocking Time (TBT)

Attach a `PerformanceObserver` **before the page loads** via `addInitScript`, so no early events are missed.

```js
const initScript = () => {
  window.__perf = { lcp: null, longtasks: [] };

  try {
    const poLcp = new PerformanceObserver((list) => {
      const entries = list.getEntries();
      const last = entries[entries.length - 1]; // LCP candidate can change multiple times, take the last one
      if (last) {
        window.__perf.lcp = {
          renderTime: last.renderTime || last.loadTime, // images use loadTime if renderTime=0 (cross-origin, no Timing-Allow-Origin)
          url: last.url,
          size: last.size
        };
      }
    });
    poLcp.observe({ type: 'largest-contentful-paint', buffered: true });
  } catch (e) {}

  try {
    const poLt = new PerformanceObserver((list) => {
      list.getEntries().forEach(e => window.__perf.longtasks.push({ duration: e.duration }));
    });
    poLt.observe({ type: 'longtask', buffered: true });
  } catch (e) {}
};

await p.addInitScript(initScript);
// ... goto + waitForTimeout as in section 1 ...

const data = await p.evaluate(() => {
  const perf = window.__perf || { lcp: null, longtasks: [] };
  const tbt = perf.longtasks.reduce((sum, t) => sum + t.duration, 0); // TBT = sum of each long task's time over 50ms
  return {
    lcpRenderTime: perf.lcp ? Math.round(perf.lcp.renderTime) : null,
    lcpUrl: perf.lcp ? perf.lcp.url : null,
    totalBlockingTime: Math.round(tbt),
    longtaskCount: perf.longtasks.length
  };
});
```

**Note on noise:** numbers vary quite a lot between runs (same page, same conditions, LCP can swing by several seconds). Run at least 2-4 times per scenario, use the **median**, don't trust a single run. When comparing two scenarios (before/after a fix), **interleave** the run order (A, B, A, B, ...) instead of running all of A then all of B, to avoid noise from the measuring machine's load changing over time.

---

## 3. Block individual resources to find the cause (`page.route`)

Use `page.route(pattern, handler)` to block a group of requests before `goto()`, then compare LCP/TBT against the baseline (nothing blocked).

```js
const PATTERNS_TO_BLOCK = ['**/*judgeme*', '**/*pagefly*']; // Playwright-style glob patterns, adjust to the suspect third-parties on the page

for (const pattern of PATTERNS_TO_BLOCK) {
  await p.route(pattern, route => route.abort());
}
// then p.goto(...)
```

**How to interpret results — important, easy to get wrong:**
- If blocking one resource makes LCP drop sharply, that **doesn't necessarily mean that resource was "heavy"** — LCP might simply have **jumped to measuring a different element** instead (a false signal). Always check the returned `lcpUrl` before/after to confirm you're comparing the same element.
- To know the REAL impact of N individual resources, add a "block everything at once" scenario to see the combined effect — don't jump to conclusions from blocking things one at a time (many individually-harmless-looking third parties can add up to something significant).

### Common pattern reference (Playwright glob vs Chrome DevTools URLPattern)

These are examples from **Shopify** sites specifically (the site this methodology was first developed on) — not a fixed/required list. On a non-Shopify site, most of these won't apply at all. Always inspect the actual page's network requests to find the real third-parties worth investigating, and write a pattern for those instead.

| Target | Playwright (`page.route`) | Chrome DevTools Request Blocking (URLPattern) |
|---|---|---|
| PageFly | `**/*pagefly*` | `*://*/*pagefly*` |
| Judge.me | `**/*judgeme*` | `*://*/*judgeme*` |
| Google GTM/Ads | `**/*googletagmanager*` | `*://*googletagmanager.com/*` |
| Facebook Pixel | `**/*facebook.com*` | `*://*facebook.com/*` |
| Shop Pay/Customer Accounts | `**/shopifycloud/shop-js/**` | `*://*/shopifycloud/shop-js/*` |
| Web Pixels | `**/web-pixels/**` | `*://*/web-pixels/*` |
| Shopify Telemetry | `**/otlp-http-production*/**` | `*://otlp-http-production*/*` |

> Newer Chrome DevTools versions validate patterns against the **URLPattern API** — bare globs like `**/*x*` aren't accepted, you need a full `scheme://host/path`.

---

## 4. LCP breakdown into 4 phases (like PageSpeed Insights)

Uses Navigation Timing + Resource Timing + the LCP entry to pinpoint exactly which phase LCP is slow in.

```js
const data = await p.evaluate(() => {
  const perf = window.__perf; // attached in section 2
  const nav = performance.getEntriesByType('navigation')[0];
  const hero = performance.getEntriesByType('resource').find(r => r.name === perf.lcp.url);

  const ttfb = nav.responseStart;
  const resourceLoadDelay = hero.requestStart - ttfb;             // from HTML available to the image STARTING to load
  const resourceLoadTime = hero.responseEnd - hero.requestStart;  // ACTUAL time spent loading the image
  const elementRenderDelay = perf.lcp.renderTime - hero.responseEnd; // from load finished to painted on screen

  return {
    ttfb: Math.round(ttfb),
    resourceLoadDelay: Math.round(resourceLoadDelay),
    resourceLoadTime: Math.round(resourceLoadTime),
    elementRenderDelay: Math.round(elementRenderDelay),
    total: Math.round(perf.lcp.renderTime)
  };
});
```

**How to read the results:**
- High `ttfb` → server is slow to respond with HTML (backend/CDN).
- High `resourceLoadDelay` → the image is discovered too late (missing `fetchpriority="high"`/`loading="eager"`, or another script is blocking discovery).
- High `resourceLoadTime` → the image is heavy, or bandwidth is being shared with many other requests (cross-check section 5).
- High `elementRenderDelay` → some JS is delaying the actual PAINTING of the image even though it already finished loading (e.g. a lazy-load fade-in effect waiting on `requestIdleCallback`, delayed because the main thread is busy).

---

## 5. Measure bandwidth contention (how many requests run concurrently)

```js
const initScript = () => {
  window.__resources = [];
  new PerformanceObserver((list) => {
    list.getEntries().forEach(e => window.__resources.push({
      name: e.name, startTime: e.startTime, responseEnd: e.responseEnd, transferSize: e.transferSize
    }));
  }).observe({ type: 'resource', buffered: true });
};

// ... after load finishes, with hero = the LCP image's resource entry:
const concurrent = resources.filter(r =>
  r.startTime <= hero.responseEnd && r.responseEnd >= hero.startTime && r.name !== hero.name
);
const totalBytesInFlight = concurrent.reduce((s, r) => s + (r.transferSize || 0), 0);
```

Tells you how many other requests + how many bytes were in flight competing for bandwidth while the LCP image was loading.

---

## 6. CPU Profiling — see which script is ACTUALLY consuming CPU

Much more reliable than just blocking individual files and inferring — this is real CPU data.

```js
await cdp.send('Profiler.enable');
await cdp.send('Profiler.setSamplingInterval', { interval: 100 }); // μs
await cdp.send('Profiler.start');

// ... goto + waitForTimeout ...

const { profile } = await cdp.send('Profiler.stop');

const nodeById = new Map(profile.nodes.map(n => [n.id, n]));
const selfTimeByUrl = {};
profile.samples.forEach((nodeId, i) => {
  const dt = profile.timeDeltas[i] || 0;
  const node = nodeById.get(nodeId);
  const url = node?.callFrame?.url || '(native/idle)';
  selfTimeByUrl[url] = (selfTimeByUrl[url] || 0) + dt;
});

const ranked = Object.entries(selfTimeByUrl)
  .map(([url, us]) => ({ url, ms: Math.round(us / 1000) }))
  .sort((a, b) => b.ms - a.ms);
```

Result ranks each script URL by **actual CPU ms spent** — use this to answer precisely "which script is blocking the main thread", instead of guessing by blocking network requests.

---

## 7. Cross-check against real PageSpeed Insights data

`pagespeed.web.dev` gives 2 kinds of data, and it's important to tell them apart when reading a report:

- **Field data (CrUX, 28 days)** — LCP measured from **real users**, stable, trustworthy. This is the number that actually affects SEO/ranking.
- **Lab data (simulated Lighthouse run)** — a single simulated run (usually "Moto G Power + Slow 4G"), **very noisy, can diverge significantly from field data**. Don't panic over a high lab number — always cross-check against field data first.

You can open the report with Playwright to read it directly instead of just looking at a screenshot:

```js
await page.goto('https://pagespeed.web.dev/analysis/https-<domain>/<report-id>?form_factor=mobile');
await page.waitForTimeout(3000);
// use browser_snapshot or browser_find to read the numbers, or click the "LCP breakdown"/"3rd parties" groups to expand details
```
