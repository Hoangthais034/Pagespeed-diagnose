---
description: Measure and diagnose LCP/performance of a web page using Playwright (mobile or desktop emulation, network/CPU throttle, resource blocking, LCP breakdown, CPU profiling)
argument-hint: <url> [mobile|desktop]
---

Invoke the `lcp-audit` skill to perform an LCP/performance audit for: `$ARGUMENTS`

Parse the arguments as `<url> [mobile|desktop]`:
- If no URL was provided, ask for it before starting.
- If no device type was provided, ask the user whether to audit **mobile** or **desktop** before starting — don't assume.
- Use the matching device profile (mobile or desktop) described in the `lcp-audit` skill's base setup section.
