---
name: infer-upgrade
description: Use when the user says "update infer", "upgrade infer", invokes /infer-upgrade, or when another Infer skill detects an outdated MCP / skills version. Updates the MCP server and skills to the latest versions.
allowed-tools:
  - AskUserQuestion
---

# Infer Upgrade

Updates the two Infer components to their latest versions:

1. **`@inferevents/mcp`** — the MCP server (npm package)
2. **`infer-events/skills`** — this skills bundle (GitHub repo, pulled via `npx skills add`)

(The legacy `@inferevents/sdk` is deprecated — Phase 8 of the gateway pivot.
If it's still installed, `/infer-setup` walks you through removing it.)

## Step 1: Check current vs latest

```bash
# Latest versions
echo "MCP_LATEST=$(npm view @inferevents/mcp version 2>/dev/null || echo 'not-published')"

# Skills last install date (proxy for installed version — this repo pulls from GitHub HEAD)
echo "SKILLS_MTIME=$(stat -f '%Sm' -t '%Y-%m-%d' ~/.claude/skills/infer-setup/SKILL.md 2>/dev/null || stat -c '%y' ~/.claude/skills/infer-setup/SKILL.md 2>/dev/null | cut -d' ' -f1 || echo 'not-installed')"
```

## Step 2: Show status

```
Infer Update Check
──────────────────────────────────────

MCP (@inferevents/mcp)
  Latest:    [version]
  Status:    [up to date / UPDATE AVAILABLE / not installed]
  Note:      npx always fetches latest on restart (if .mcp.json unpinned)

Skills (infer-events/skills)
  Installed: [date]
  Status:    Re-installing to ensure latest

SDK (@inferevents/sdk)
  Status:    DEPRECATED — gateway-era Infer doesn't use the SDK.
             Run /infer-setup if you still have it installed.
```

## Step 3: Update

Ask via `AskUserQuestion`: "Ready to upgrade? This re-installs the skills
bundle and clears the MCP cache so the next restart picks up the latest
MCP version."

### 3a: Update MCP

The MCP runs via `npx @inferevents/mcp`, which caches. Clear the cache:

```bash
# Prime the cache with the latest version
npx --yes @inferevents/mcp@latest --version 2>/dev/null || true
```

Check `.mcp.json` in the project root (and `~/.claude/.mcp.json` globally).
If `args` contains a pinned version like `@inferevents/mcp@1.0.0`, replace
it with the unpinned form `@inferevents/mcp` so npx always picks up the
latest on restart:

```bash
# Check and fix pinned versions
if [ -f .mcp.json ] && grep -q '@inferevents/mcp@' .mcp.json; then
  sed -i '' 's/@inferevents\/mcp@[0-9.]*/@inferevents\/mcp/g' .mcp.json
  echo "Fixed pinned MCP version in .mcp.json"
fi
```

Tell the user: "Type `/mcp` and restart the `infer` server. No need to
exit Claude Code."

### 3b: Update skills

```bash
npx skills add infer-events/skills
```

This always fetches the latest from GitHub HEAD (no caching).

### 3c: Write cache

```bash
mkdir -p ~/.infer
echo "{\"checked\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"mcp\":\"$(npm view @inferevents/mcp version 2>/dev/null || echo 'unknown')\"}" > ~/.infer/last-update-check.json
```

## Step 4: Re-verify baseURL config

Your LLM client's `baseURL` is unchanged by a skills or MCP upgrade — it
still points at `gateway.infer.events/v1/<PROJECT_ID>/<provider>/...`.
But if you've restructured your project recently (moved files, switched
frameworks, etc.), the old baseURL edit may have been lost.

Run a quick probe to confirm spans are still flowing:

```
list_spans(limit=1, time_window=10m)
```

If a span came back within the last 10 minutes, you're good. If not:
- Make a test LLM call from your app
- Re-run `list_spans(limit=1, time_window=5m)` after the call
- If still nothing: run `/infer-setup` to re-verify the baseURL configuration

## Step 5: Verify MCP is live

After the `/mcp` restart, run a test tool call:

```
get_project_summary()
```

If it returns the compiled wiki (model distribution, weekly spend, error
rate, active threads), the MCP server is live and talking to the API.

## Step 6: Compatibility Matrix

| Skills | MCP | Notes |
|---|---|---|
| 2.0+ | 1.0.0+ | LLM-obs pivot (current gateway-era). Skills auto-loaded on LLM observability questions. |
| 1.x | 0.1.x | Legacy web-analytics era. Strongly recommend upgrading — the 1.x skills assume retired MCP tools (`get_event_counts`, `get_retention`, etc.) |

If you're on skills 1.x + MCP 1.0+: the legacy skills will call retired
MCP tools, which return deprecation envelopes until 2026-06-17. Upgrade
skills immediately.

If you're on skills 2.0+ + MCP 0.1.x: the new skills reference tools
(`list_spans`, `get_trace`, etc.) that don't exist in 0.1.x. Upgrade MCP
first.

## Summary Format

```
Infer Upgrade Complete
──────────────────────────────────────

MCP     1.0.0 → 1.0.1   Updated (after /mcp restart)
Skills  ────────────────  Re-installed from GitHub HEAD
```

If already up to date:

```
Infer — All Up to Date
──────────────────────────────────────

MCP     1.0.1   Latest
Skills  ──────  Re-installed (always pulls latest)
```

## Step 7: Closing

Call `AskUserQuestion`:

> Upgrade complete. What next?
>
> 💡 Tip: After an upgrade, it's worth a quick list_spans probe to confirm nothing silently broke.

Options:
- A) Verify spans are flowing — run `list_spans(limit=1, time_window=5m)`
- B) Run a health check — `/infer-insights`
- C) Check what's new — show me the changelog
- D) Back to work — everything looks good
