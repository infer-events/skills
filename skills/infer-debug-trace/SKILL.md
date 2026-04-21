---
name: infer-debug-trace
description: Use when diagnosing a specific trace (slow, failed, truncated, or anomalous), when invoked with a trace_id from an insight, or as follow-up to get_error_spans / get_trace / list_spans showing a problem trace. Triggers on "diagnose trace", "why was this slow", "why did this fail", "investigate this trace", "debug session X".
allowed-tools:
  - AskUserQuestion
---

# Infer Debug Trace — Live Investigation

Your job: given a problem trace (slow, failed, truncated, anomalous), run
the diagnosis arc, identify the root cause, annotate the finding so future
sessions inherit it. The skill is a live-investigation tool — it does NOT
generate tracking plans; it does NOT read codebase structure. It executes
MCP tool calls against specific traces and follows the investigation
protocols from `infer-observability`.

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

## Entry Contract — Three-Tier Resolution

Before diagnosis starts, resolve *which* trace you're investigating. Use
this order — don't skip tiers.

### Tier 1: Explicit `trace_id`

If the invocation gives you a specific `trace_id` (user pasted it, upstream
skill passed it, an insight had `evidence.trace_id_samples[0]`), jump
straight to **The Investigation Arc** below.

Examples of explicit invocation:
- "diagnose trace 550e8400-e29b-41d4-a716-446655440000"
- `infer-insights` passed `evidence.trace_id_samples[0]` after user approved
  follow-up on an insight

### Tier 2: Sniff recent conversation

If no explicit `trace_id` in the invocation, scan recent conversation
context for `trace_id` values. MCP tool output embeds them in:
- `get_error_spans` result rows
- `get_trace` output (the root `trace_id`)
- `list_spans` result rows
- `get_insights` evidence (`trace_id_samples[]`)
- `annotate_trace` previous calls

If **exactly one** candidate `trace_id` appears in recent context AND it's
semantically connected to what the user asked about ("the top one", "this
slow trace", "the failure I just saw"), use it.

If **multiple** candidates appear (e.g. user ran `get_error_spans` with 10
results), ask via `AskUserQuestion`:

> Which trace should I diagnose?
>
> - A) trace_id=<first-ten-chars-of-first>... (status 500, 1247 ms, 3 minutes ago)
> - B) trace_id=<first-ten-chars-of-second>... (status 429, 852 ms, 8 minutes ago)
> - C) trace_id=<first-ten-chars-of-third>... (status 500, 2103 ms, 12 minutes ago)
> - D) None of these — let me narrow down differently

### Tier 3: Narrow-down subroutine

If no `trace_id` is available anywhere, ask the user about the **symptom**
and run a narrow-down query:

- **Slow** → `list_spans(min_duration_ms=3000, time_window=1h)` — returns
  slow spans; pick the top few
- **Failed** → `get_error_spans(limit=10, time_window=1h)` — returns
  recent failures
- **Specific `error_type`** → `get_error_spans(error_type=<X>, limit=10)`
- **High-token** → `list_spans(time_window=1h)` and sort mentally on
  `gen_ai_usage_input_tokens`
- **User saw something specific** → ask what they saw, then narrow

Present candidates via `AskUserQuestion` with 3–4 options (one trace each,
plus an "other / narrow differently" option).

### Session-Scoped Invocation

If the user gives a `session_id` instead of a `trace_id` ("debug session
tg_55555"):

1. Run `list_spans(session_id=<X>, time_window=24h)` — returns all spans
   in that session
2. Filter mentally to **failing spans** (`status_code >= 400`) OR **slow
   spans** (top 5 by `duration_ms`) — these are the candidates
3. Present candidates via `AskUserQuestion` as in Tier 3
4. Once the user picks a trace, execute the full investigation arc on it
5. After investigation, offer to loop: "Investigate another trace in this
   session?" — session-scoped debug often has multiple correlated issues

## The Investigation Arc — 4-Beat Signature

Once you have a `trace_id`, execute these four phases in order. Each phase
maps to one beat of the investigation-protocol signature shared with
`infer-observability`:

### Phase 1: Symptom — Surface the trace shape

Call `get_trace(trace_id)`. This returns:
- `total_duration_ms` — wall-clock time from first span start to last span end
- `spans[]` — the full hierarchy, sorted by start_time with depth indicators
- Any annotations that already exist on this trace

Read the shape:
- **How many spans?** One = single LLM call. Multiple = agent iteration
  chain. 2–5 is typical; >10 suggests a long multi-step agent turn.
- **Is the hierarchy nested?** `parent_span_id` links children to parents.
  Biscuit currently doesn't propagate `parent_span_id`, so multi-iteration
  biscuit turns look like multiple roots (Phase 6 documents this).
- **Are there error spans?** Any span with `status_code >= 400` or
  `stream_truncated=true` is a candidate problem.
- **Is there a clear "slow" span?** Compare each span's `duration_ms`
  against the trace's `total_duration_ms` — the span accounting for most
  of the time is your primary target.

Note any existing annotations — if a prior session already investigated
this trace, you may not need to repeat work. Read their annotation; if the
finding is stale or needs follow-up, continue. Otherwise, hand back to the
user: "This trace was already investigated — <prior annotation>. Is that
still the answer you need?"

### Phase 2: Classify — what kind of problem is this?

Based on Phase 1's shape, route to one of the investigation protocols:

| Observed | Route to |
|---|---|
| Slow trace, all spans `status_code=200` | `infer-observability` § "Diagnosing a slow trace" |
| Failed span(s) with `status_code >= 400` | `infer-observability` § "Diagnosing a new error_type" |
| `attributes.stream_truncated=true` on a streaming span | Mixed — first slow-trace protocol (upstream may have slowed before dropping), then new-error-type if a specific error preceded truncation |
| `attempt_count > 1` on multiple spans | `infer-observability` § "Diagnosing a retry storm" |
| Unexpectedly high `gen_ai_usage_input_tokens` | `infer-observability` § "Diagnosing a cost spike" (even if not a budget issue, prompt size is the lever) |
| Multi-span trace with only one span slow | Slow trace protocol, narrow to that specific span |

If multiple classifications apply (e.g. failed AND slow AND retried —
common for upstream-pressure failures), prioritize in order: retry-storm →
new-error-type → slow-trace. Retry-storm explains the duration; error-type
explains why; slow is usually a downstream symptom.

### Phase 3: Diagnose — execute the protocol

Cross-reference `infer-observability`'s protocol for the routed classification
— don't re-derive the steps; execute them against this trace.

Key tool calls you'll use:
- **`get_span(span_id=X)`** — deep inspection of one span; gives you the full
  `attributes` JSONB, `messages` (if not redacted), upstream headers, and
  the exact error body when present.
  **Large-payload note:** when `gen_ai_usage_input_tokens > 20,000`, the
  response can exceed 100 KB because `messages` includes the full
  accumulated context. The MCP harness saves oversized responses to a file
  and returns the path — use `jq`, Grep, or a small Python script against
  the saved file rather than trying to print the blob inline.
- **Run the `correlation_hint`** (from the insight if you came from
  `infer-insights`, else construct it manually): `git log --since=<anomaly_start - 2h> --until=<anomaly_end + 30m> --oneline`
- **`get_token_usage(dimension=feature, time_window=1h)`** — attributes
  token burn to the feature (if `metadata.feature` is set); useful for cost
  investigations

Execute the steps, take notes on what you find.

Compare findings against the "Likely root causes" list in the protocol:
- If one cause clearly matches the evidence, you have a finding
- If no cause clearly matches, you have an inconclusive investigation
- If evidence points to user-side info (e.g. "check if your rate-limit tier
  changed"), you have a pending-user investigation

### Phase 4: Close the loop — annotate

Call `annotate_trace(trace_id, content)` with content starting in one of
the four tag prefixes:

- **`Root cause:`** — confirmed finding. Include what you found AND the
  suggested fix.
  ```
  Root cause: commit abc123 "refactor prompt builder" enlarged system
  prompt from 400 to 1200 tokens at 14:23 UTC; p95 tripled at that
  timestamp. Fix: revert commit or cap prompt template length.
  ```

- **`Observation:`** — pattern noticed but not directly causal.
  ```
  Observation: attempt_count=3 across all spans in this trace; upstream
  returned 503 on attempts 1 and 2, then 200 on attempt 3. Consistent
  with a transient Ollama outage; no code-side change needed.
  ```

- **`Inconclusive:`** — investigated, no systemic cause.
  ```
  Inconclusive: prompt_tokens=1200 (within 7d p50 of 1400), upstream
  200 on first attempt, status codes all normal. Single slow outlier;
  may be upstream colo variance. Will re-investigate if pattern repeats.
  ```

- **`Pending user:`** — blocked on user-side info.
  ```
  Pending user: asked user to check Ollama Cloud rate-limit tier at
  cloud.ollama.com/dashboard. Awaiting response.
  ```

After annotating, suggest the next user action in your text response:
- Root cause → "Want me to help revert commit abc123?" or "Want me to draft
  a PR that caps prompt size?"
- Observation → "This looks transient; I'll watch `get_insights` over the
  next hour. Anything else?"
- Inconclusive → "Nothing structural found. I'll flag this in annotations
  so future slowness here gets cross-referenced."
- Pending user → "Once you check and let me know, I'll continue the
  investigation."

## Common Failure Modes — Quick Reference

When you're first reading the trace's symptom, this table points you at the
most-likely cause and the first query to run:

| Symptom | Likely cause | First query |
|---|---|---|
| Slow + large `input_tokens` | Prompt regression | `git log --since=<anomaly_start - 2h>` + check prompt files |
| Slow + `finish_reason="length"` | `max_tokens` hit; response was forced-stopped | Check client's `max_tokens` setting; may need raising or prompt tightening |
| Slow + `tool_calls` / `tool_use` content | Agent iteration (long because agent was working, not because LLM was slow) | `get_trace(trace_id)` to see full iteration count |
| Slow + `attempt_count > 1` | Upstream retry; duration includes retry waits | Check `status_code` distribution across attempts |
| Fail + `status_code=429` | Rate limit | Check `attempt_count`; check `x-ratelimit-*` headers in `attributes` via `get_span` |
| Fail + `status_code=503` or `502` | Upstream outage | `get_error_spans(error_type=upstream_error, time_window=1h)` — is it a cluster? |
| Fail + `status_code=400` | Malformed request | Check `messages` payload in `get_span`; likely a code-side schema change |
| `stream_truncated=true` on a streaming span | Mid-stream upstream drop or gateway cap | Check `attributes.upstream_status` + last-chunk timestamp via `get_span` |

## Closing AskUserQuestion

After annotating, close with:

```
Use AskUserQuestion:

> Investigation complete. What's next?
>
> 💡 Tip: [rotate from tip list below]
```

Options:
- A) **Annotate and move on** — record the finding (already done in Phase 4);
  return to the prior conversation
- B) **Investigate similar traces** — `list_spans(time_window=24h)` filtered
  to the same `(provider, model)` to see if this is a pattern
- C) **Check for related insights** — `get_insights()` to see if the cron
  already flagged this
- D) **Set up recurring monitoring** — suggest `/loop` or `/schedule` for
  periodic `get_insights()` checks on this `(provider, model)` pattern.
  Useful when the investigation closed as Observation/Inconclusive and you
  want a safety net for recurrence, or when the user wants automated health
  reports

Rotating tips (pick one not used this session):
- `💡 Tip: Prompt size is the #1 cause of LLM latency — check gen_ai_usage_input_tokens first`
- `💡 Tip: Annotate with a tag prefix (Root cause: / Observation: / Inconclusive: / Pending user:) so future sessions can filter`
- `💡 Tip: attempt_count > 1 means the gateway retried; root cause is upstream-side`
- `💡 Tip: If the trace had a non-null session_id, list_spans(session_id=<X>, time_window=24h) shows the full session — useful for correlated issues across turns`

## What This Skill Is NOT

- **Not a tracking plan.** No proposing events or codebase-driven schema.
  That was the retired `infer-tracking-plan`; the LLM-obs equivalent of
  "what should I track?" is "set `x-infer-metadata` and `x-infer-session-id`
  headers" — covered in `infer-setup`.
- **Not a codebase-reading tool.** The investigation uses MCP tool calls
  against spans/traces + `git log` on the user's actual repo. It doesn't
  propose code changes.
- **Not a proposal generator.** The output is a root-cause annotation, not
  a Markdown tracking-plan document.

## Cross-References

- `infer-observability` — the interpretation guide; this skill cross-references
  its protocols in Phase 3 rather than re-deriving them
- `infer-insights` — the proactive-briefing skill; hands off to this skill
  via `evidence.trace_id_samples[0]`
- `annotate_trace` / `annotate_span` MCP tools — what this skill calls in
  Phase 4 to close the loop
