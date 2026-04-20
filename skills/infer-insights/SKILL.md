---
name: infer-insights
description: Use when the user asks for LLM observability insights, a health check, a pulse on their agent, or says "what should I know about my data." Also use when invoked via /schedule or /loop for automated daily insights.
allowed-tools:
  - AskUserQuestion
---

# Infer Insights — Proactive Briefing

Your job: surface what's worth knowing about the agent's LLM traffic before
the user has to ask. Check pre-computed anomalies (the Phase 5 hourly cron
does the detection math); interpret via `infer-observability`'s protocols;
hand off to `infer-debug-trace` for deep dives.

## Update Check (non-blocking)

```bash
_INFER_CACHE=~/.infer/last-update-check.json
_NEEDS_CHECK="yes"
if [ -f "$_INFER_CACHE" ]; then
  _CACHE_AGE=$(( $(date +%s) - $(stat -f '%m' "$_INFER_CACHE" 2>/dev/null || stat -c '%Y' "$_INFER_CACHE" 2>/dev/null || echo 0) ))
  [ "$_CACHE_AGE" -lt 21600 ] && _NEEDS_CHECK="no"
fi
if [ "$_NEEDS_CHECK" = "yes" ]; then
  _MCP_LATEST=$(npm view @inferevents/mcp version 2>/dev/null || echo "unknown")
  mkdir -p ~/.infer
  echo "{\"checked\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"mcp_latest\":\"$_MCP_LATEST\"}" > "$_INFER_CACHE"
else
  echo "INFER_CHECK=cached"
fi
```

If an update is available, append `Infer update available — run /infer-upgrade to get the latest.` at the END of your response. Don't block.

## STEP ZERO & STEP ONE: Summary + Insights (fire in parallel)

These two reads are independent — call them in parallel to halve the
wall-clock. You need both before presenting the briefing: the summary gives
project-wide context; insights surface cron-detected anomalies.

### STEP ZERO: `get_project_summary()`

Hourly-compiled wiki: model distribution, weekly spend, 7-day error rate,
active anomaly threads, annotations from previous investigations.

### STEP ONE: `get_insights()`

No severity filter — get everything. Insights include:

- **Evidence keys** — `evidence.provider`, `evidence.model`, optional
  `evidence.trace_id`, plus per-detection fields (e.g. `p95_regression_pct`,
  `error_rate_current`, `cost_ratio`)
- **Thread context** — `thread_id`, `day_count` (how long the issue has
  been ongoing), and any prior annotations
- **Correlation hint** — a pre-built `git log` command that widens 2 hours
  before the anomaly start, ready to copy-paste

**Empty envelope handling.** If `get_insights` returns the empty envelope
("No new insights. Everything looks normal, or there isn't enough data
yet."), **trust it** for interactive health-check queries — the cron runs
hourly and covers 9 detection types across 7-day baselines; empty really
does mean "nothing unusual." Use the compact **`## Infer Pulse`** output
format from the Automated Mode section below, not the full multi-insight
format. Only run the Manual Query Sequence if the user asks for a probe
deeper than the default briefing.

## STEP TWO: Correlate with code changes (for CODE actions)

For insights where `action_type` is "code" (i.e. the cron thinks a code
change is the likely cause), run the provided `correlation_hint`:

```bash
# Example hint from the cron:
git log --since='2026-04-19T13:00:00Z' --until='2026-04-19T15:00:00Z' --oneline
```

If commits appear, present the most recent one as "Likely cause." If none,
say "No related commits found in this window."

## STEP THREE: Present the briefing

Format for each insight:

```
[!!] CRITICAL: openai/gpt-4o p95 up 58% since Wed 14:00 UTC
      n=1247, p_value_adjusted=0.0003, baseline_p95=890ms, current_p95=1410ms
      detected: 2026-04-19 15:00 UTC | thread day 2

      Likely cause: commit abc123 "refactor prompt builder" (3 hours before anomaly)
      → Invoke infer-debug-trace on trace_id=550e8400-... to diagnose?

[!] NOTABLE: new error_type "schema_validation" (8 occurrences last hour)
      trace samples: 3f8a..., b2c1...
      detected: 2026-04-19 15:00 UTC | day 1

      Likely cause: no related commits found
      → Run get_error_spans(error_type="schema_validation") to investigate?
```

Include the **raw `get_insights` tool output verbatim** (Unicode bar charts,
sparklines, structured layouts are part of the product) before adding your
interpretation.

## STEP FOUR: After investigating — annotate_trace

**After investigating any insight** (git log, code check, span inspection),
call `annotate_trace(trace_id, content)` — not `annotate_thread`.

When the insight's `evidence.trace_id` is set (single-trace anomaly, Phase 5
thread-linker routes these so `insight_threads.id = trace_id`), annotate the
trace directly. Otherwise, if the anomaly is aggregate (many traces),
`annotate_thread(thread_id, content)` is the fallback — but note that
`thread_id = trace_id` for single-trace anomalies, so both paths converge.

Lead with one of the four **tag prefixes** (see `infer-observability` for full
guidance):
- `Root cause:` — confirmed with evidence
- `Observation:` — pattern noticed but not a direct cause
- `Inconclusive:` — investigated, nothing systemic found
- `Pending user:` — blocked on user-side info

Example:
```
annotate_trace(
  trace_id: "550e8400-e29b-41d4-a716-446655440000",
  content: "Root cause: commit abc123 enlarged system prompt from 400 to 1200 tokens; p95 tripled at that timestamp. Fix: revert or cap prompt size."
)
```

Good annotations are specific + actionable. Bad annotations ("looked at it,
seems fine") leave the next agent no better off.

## STEP FIVE: Close with AskUserQuestion

**Branch the options by what the data actually supports.** Don't present
no-ops (e.g. "Investigate the top insight" when `count == 0`).

### If insights exist (count > 0)

```
Use AskUserQuestion. The question is:

> Which insight do you want to dig into?
>
> 💡 Tip: [rotate from the tip list below]
```

Options:
- A) **Investigate the top insight** — invoke `infer-debug-trace` with the
     top insight's `evidence.trace_id` (or narrow-down subroutine if
     aggregate)
- B) **Zoom out** — re-run `get_project_summary()` for the project-wide
     view
- C) **Dig into a specific model / provider** — narrow via
     `get_latency_stats(dimension=model)` or
     `get_cost_stats(dimension=model)`
- D) **Set up daily monitoring** — `/loop` or `/schedule` for recurring
     checks

### If the envelope was empty (count == 0)

```
Use AskUserQuestion. The question is:

> Everything looks normal. What's next?
>
> 💡 Tip: [rotate from the tip list below]
```

Options:
- A) **Set up recurring monitoring** — `/loop` or `/schedule`; highest-
     leverage action on a quiet project
- B) **Probe deeper** — run the Manual Query Sequence Round 1
     (`get_latency_stats` + `get_error_spans` + `get_cost_stats`) to look
     past the cron
- C) **Verify spans are flowing** — `list_spans(limit=5, time_window=1h)`
     to confirm the gateway is still capturing traffic (rule out a silent
     outage)
- D) **Configure missing dimensions** — if `session_id` / `user_id` /
     `metadata.feature` are null on spans, the client isn't propagating
     `x-infer-*` headers; unlock per-session/per-user analytics by invoking
     `infer-setup`

### If the project has < 100 spans total

Use the options from the "When You Don't Have Enough Data" section below.

### Rotating tips (pick one not used in this session)

- `💡 Tip: Set up /loop to run this hourly — you'll catch regressions before they compound`
- `💡 Tip: attempt_count > 1 means the gateway retried; the root cause is upstream-side`
- `💡 Tip: Annotate with a tag prefix (Root cause: / Observation: / Inconclusive: / Pending user:) so future sessions can filter`
- `💡 Tip: Prompt size is the #1 cause of LLM latency — check gen_ai_usage_input_tokens first`
- `💡 Tip: An empty envelope on a project with steady traffic is the goal — means the gateway + cron are both working and nothing is anomalous`

## Manual Query Sequence (when get_insights is empty)

When `get_insights` returns empty AND the user wants a probe deeper than the
project summary, run these in order. Bail early if you find something worth
surfacing — don't run the whole sequence if Round 1 produces a finding.

### Round 1: Top-level health (latency + errors + cost)

```
get_latency_stats(time_window=24h)          # p50/p95/p99 per model
get_error_spans(limit=5, time_window=24h)   # recent failures
get_cost_stats(time_window=7d)              # weekly spend by model
```

Look for:
- p95 > 2× typical (rare on quiet data; likely means something's happening)
- Error spans with unfamiliar `error_type`
- Cost spike vs the prior week's total

If something looks off, switch to `infer-observability`'s investigation
protocols.

### Round 2: Dimensional breakdown

If Round 1 was quiet, narrow by dimension:

```
get_latency_stats(dimension=session, time_window=24h)   # per-session latency spread
get_token_usage(dimension=user, time_window=7d)         # per-user token burn
```

Only useful if the user's client propagates `x-infer-session-id` and
`x-infer-user-id`. If dimensions return one aggregate bucket, the client
isn't setting those headers — surface this as a setup issue, not a data
issue.

### Round 3: Deep dive on specific spans

If you've narrowed to a specific session or model:

```
list_spans(session_id=<X>, time_window=24h)             # all spans in that session
get_trace(trace_id=<sample>)                            # turn-level view of one trace
```

At this point you've transitioned from "insights pattern-surface" to
"trace-level diagnosis" — hand off to `infer-debug-trace`.

## Output Format

Present insights in this shape:

```
## Infer Insights — [Date]

**Data window:** [time range] | **Total spans:** [N]

[Raw tool output with bar charts here, verbatim]

### 1. [Headline — the insight in one sentence]

[2-3 sentences: what the cron detected, what it means, suggested action.]

### 2. [Next insight]
...

### 3-5. [Remaining insights]
...

---
**Next check:** [When the next automated run is scheduled, if applicable]
```

Then call `AskUserQuestion` per Step 5 above.

## Automated Mode (via /schedule or /loop)

When triggered by a schedule:

1. Call `get_project_summary()` first — pre-compiled, fast. Health score,
   metrics, active threads, prior annotations in one call.
2. Call `get_insights()` only if the summary suggests active issues (health
   < 8 OR `anomaly_threads.length > 0`). Skip on quiet days.
3. Call `annotate_trace()` (or `annotate_thread()` fallback) if you
   investigate anything.

Only notify the user if:
1. An insight is severity `critical`
2. Error rate spiked > 2× vs the prior period
3. Cost spiked > 3× for any single model
4. A new `error_type` appeared with > 10 occurrences in the hour
5. Zero spans recorded (the gateway might be broken)

For routine checks with no notable findings:
```
## Infer Pulse — [Date]
Spans last 24h: [N] | Error rate: [X%] | Active threads: [N]
All clear. Nothing unusual.
```

## When You Don't Have Enough Data

If the project is new (< 100 spans in 7d):

```
## Infer Insights — [Date]

**Status:** Not enough data yet for meaningful insights.

**What I can tell you:**
- [N] spans since [first span date]
- Models observed: [list]
- Providers: [openai | anthropic | ollama]

**What I need:**
- At least 100 spans for statistically-significant detections
- 24+ hours of data for hour-over-hour comparisons
- 7+ days of data for weekly-baseline detections

**Check back in a few days.**
```

Even with limited data, still call `AskUserQuestion`:

> Not enough data for insights yet, but here's what you can do now.
>
> 💡 Tip: Spans need a few days to accumulate before Phase 5 detections can fire statistically. Set up /loop now and check back.

Options:
- A) Check spans are flowing — run `list_spans(limit=5, time_window=1h)`
- B) Configure session/user headers — unlocks per-session/per-user analysis
- C) Set up daily monitoring — `/loop` or `/schedule`
- D) Generate test traffic — make a few LLM calls to accumulate data

## Cross-References

This skill surfaces patterns. `infer-observability` teaches the *interpretation*
(latency benchmarks, error norms, investigation protocols, anti-patterns).
`infer-debug-trace` executes the *investigation* on a specific trace. Lean on
both liberally.
