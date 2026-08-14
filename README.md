# pagespeed-playwright

Claude Code plugin marketplace containing `pagespeed-audit`: a skill for measuring and diagnosing LCP/performance of a web page using Playwright (mobile emulation, network/CPU throttle, resource blocking, LCP breakdown, CPU profiling, PSI cross-check).

## Install

Add this repo as a plugin marketplace in Claude Code, then install the `pagespeed-audit` plugin.

## Structure

- `.claude-plugin/marketplace.json` — marketplace manifest
- `.claude-plugin/plugin.json` — plugin manifest
- `skills/lcp-audit/SKILL.md` — the LCP audit skill
