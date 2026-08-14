# pagespeed-playwright

Claude Code plugin marketplace containing `pagespeed-audit`: a skill for measuring and diagnosing LCP/performance of a web page using Playwright (mobile emulation, network/CPU throttle, resource blocking, LCP breakdown, CPU profiling, PSI cross-check).

## Install

1. Add this repo as a plugin marketplace:

   ```
   /plugin marketplace add Hoangthais034/Pagespeed-diagnose
   ```

2. Install the plugin:

   ```
   /plugin install pagespeed-audit@claude-yho-plugins
   ```

3. Restart Claude Code so the `/pagespeed-audit` command and the `lcp-audit` skill are loaded.

To update to the latest version after the marketplace repo changes:

```
/plugin marketplace update claude-yho-plugins
/plugin update pagespeed-audit@claude-yho-plugins
```

## Usage

- Slash command: `/pagespeed-audit <url>`
- Or just ask in natural language, e.g. "audit LCP for https://example.com" — Claude will pick up the `lcp-audit` skill automatically based on context.

## Structure

- `.claude-plugin/marketplace.json` — marketplace manifest
- `.claude-plugin/plugin.json` — plugin manifest
- `commands/pagespeed-audit.md` — the `/pagespeed-audit` slash command
- `skills/lcp-audit/SKILL.md` — the LCP audit skill
