---
name: infer-observability
description: Use when interpreting Infer MCP tool results, understanding LLM observability data (latency, errors, tokens, cost, traces), diagnosing patterns in agent behavior, or capturing pre-ship baselines before a prompt/model/config change. Auto-loads when user asks about LLM performance, spend, errors, spans, or says they're about to ship/deploy/change something that affects the agent.
allowed-tools:
  - AskUserQuestion
---

# Infer Observability — Agent Interpretation Guide

Your job: interpret MCP tool output like an experienced SRE. Cite numbers
from the user's live data (via `get_latency_stats`, `get_cost_stats`, etc.),
not memorized absolutes that drift. When you diagnose, walk the protocols
below; when you find root causes, annotate the trace so future sessions
inherit the context.

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

If a newer MCP version is available, append at the END of your response:
`Infer update available — run /infer-upgrade to get the latest.`
Do NOT block the workflow.

## What Infer Captures

Every LLM call routed through `gateway.infer.events` becomes one row in
the `spans` table. Key columns:

- **Identity:** `span_id`, `trace_id`, `parent_span_id` — OTEL GenAI shape.
  Agent iterations that share a `trace_id` are one logical turn; `get_trace`
  returns the full hierarchy.
- **Timing:** `start_time`, `end_time`, generated `duration_ms`.
- **Gen AI:** `gen_ai_system` (provider), `gen_ai_request_model`,
  `gen_ai_response_model`, `gen_ai_usage_input_tokens`, `gen_ai_usage_output_tokens`,
  `gen_ai_response_finish_reasons[]`.
- **HTTP:** `status_code`, `error_type`, `upstream_host`.
- **Infer metadata:** `session_id`, `user_id`, `tags[]`, `metadata` (JSONB),
  `attempt_count` (> 1 means the gateway retried), `stream` (boolean),
  `stream_truncated` attribute (set true when upstream dropped mid-stream).
- **Messages:** `messages` JSONB — the full request/response content,
  OTEL-aligned. `NULL` when `x-infer-redact: true` was set on the request.

The grouping primitive for most detections and queries is `(provider, model)`.
When `x-infer-metadata={"feature":"..."}` is set, per-feature analysis becomes
possible via `get_*_stats(dimension=feature)`.

## Tool Routing — Which Tool for Which Question

| User question | Tool |
|---|---|
| "what's my LLM spend?" / "how much am I paying?" | `get_cost_stats(dimension=model, time_window=7d)` |
| "is anything slow?" / "p95 / p99 latency" | `get_latency_stats(time_window=24h)` then `get_insights` |
| "what errors have we seen?" / "show me failures" | `get_error_spans(limit=20, time_window=24h)` |
| "what did this trace do?" / "diagnose trace X" | `get_trace(trace_id=X)` then hand to `infer-debug-trace` |
| "give me a health check" / "anything I should know?" | `get_project_summary()` first, then `get_insights()` |
| "list recent spans" / "show me what's happening" | `list_spans(limit=20, time_window=1h)` |
| "how many tokens for feature X?" | `get_token_usage(dimension=feature, time_window=7d)` |
| "any anomalies this week?" | `get_insights()` (returns cron-detected findings) |
| "let me inspect this one span" | `get_span(span_id=X)` |
| "annotate what I found" | `annotate_span(span_id, content)` or `annotate_trace(trace_id, content)` |
| "about to ship X — anything I should watch?" / pre-deploy checklist | `get_latency_stats(dimension=model, time_window=7d)` + `get_token_usage(dimension=model, time_window=7d)` + `get_cost_stats(dimension=model, time_window=7d)` — capture baseline; see **Pre-Ship / Post-Ship Watch** protocol |
| "create a new project" | `create_project(name)` |

**Never pass `project_id` as a tool argument** — it's always derived from your
`pk_read_*` key. MCP tools reject any `project_id` in input.

## Interpreting Latency

Latency benchmarks **drift**. Providers roll out new silicon, routing
changes, model variants. Absolute ms thresholds go stale in weeks. The
authoritative answer is always **your project's live baseline** — run
`get_latency_stats(dimension=model, time_window=7d)` and compare against
that.

That said, here are order-of-magnitude sanity checks as of 2026-04. Treat
these as smell tests, not thresholds:

| Provider / model | Non-streaming total (typical) | Streaming first-token |
|---|---|---|
| `openai / gpt-4o` | hundreds of ms to low seconds (p95 ≈ 1–2 s under normal conditions) | sub-second |
| `openai / gpt-4o-mini` | 200–800 ms | sub-second |
| `openai / o1` / `o3-mini` (reasoning) | seconds to tens of seconds (reasoning spends upstream time before emitting) | N/A (non-streaming) |
| `anthropic / claude-sonnet-4-6` | 400–1500 ms first-token, 1–3 s typical full response | sub-second |
| `anthropic / claude-opus-4-7` | 800–2500 ms first-token, 2–5 s typical | sub-second |
| `ollama / glm-5.1:cloud` (via Ollama Cloud) | 2–10 s per call typical | 500 ms – 2 s |

Anything above ~5× these numbers for a sustained window is notable.
Anything 25%+ above your own 7-day baseline with statistical significance
is what Phase 5's `latency-regression` detection fires on (Welch's t-test
+ Bonferroni correction, critical at p95 ≥ 50% regression).

**Percentile rules of thumb:**
- Use **p95** when `n < 100`. p99 is too noisy.
- Use **p50** (median) to describe "typical" experience; p95 describes the
  unhappy user; p99 describes pathological tails.
- `get_latency_stats` excludes error spans (`status_code >= 400`) by design —
  a 500ms 500 is not a "fast" response, it's a fast failure.

**Streaming caveat:** for streaming calls (`stream=true`), `duration_ms`
measures end-to-end (first byte request to last byte response), so a long
streaming duration usually means a long response, not a slow upstream.
Check `attributes.gen_ai.response.finish_reasons` — `"length"` means
`max_tokens` hit.

## Interpreting Errors

| Signal | Meaning |
|---|---|
| `status_code >= 400` | Failure span. 4xx = client-side (rate limit, malformed); 5xx = upstream-side (outage, infrastructure) |
| `error_type` (text column) | Gateway's classified error (e.g. `upstream_error`, `infer_unreachable_upstream`, `infer_bad_gateway`). Phase 5's `new-error-type` detection fires when a value appears that wasn't in the 7-day baseline |
| `stream_truncated = true` (JSONB attribute) | Upstream dropped mid-stream. Span is partial but not pretending success |
| `attempt_count > 1` | The gateway retried. One retry is usually transient; >2 is a retry storm (Phase 5 `retry-storm` detection fires on mean `attempt_count` regressions) |

**Error-rate norms (rule of thumb):**
- **< 1%** — healthy steady state for a well-behaved agent
- **1–5%** — investigate: run `get_error_spans(limit=20)`, group by `error_type` to identify the pattern
- **> 5%** — production issue; escalate

Most error-rate spikes are either (a) a new `error_type` (deploy broke
something — check git log correlated with detection time) or (b) upstream
outage (cluster of `status_code=503` / `error_type=upstream_error` — often
self-heals within 5–15 minutes).

## Interpreting Tokens & Cost

**Token columns:**
- `gen_ai_usage_input_tokens` — input (prompt + system + tool specs)
- `gen_ai_usage_output_tokens` — output (completion)
- Both NULL if streaming without `stream_options.include_usage: true` AND
  the gateway's fallback-estimator (`js-tiktoken`) couldn't run — rare but
  possible

**`attributes.gen_ai.usage.estimated = true`** is set when:
- Client's streaming request didn't include `stream_options.include_usage: true`
- The gateway fell back to tokenizer-estimated counts via `js-tiktoken`

Estimated counts are close (within ~2–5%) but not exact. For cost analysis,
suggest to the user: "Your streaming calls aren't sending `include_usage:
true`, so Infer estimates tokens. For exact cost, pass that option."

**Cost:**
- Computed at query time from a static `PRICING_TABLE` snapshot (v1 design —
  see parent spec §7.3; no cost column write at ingest)
- `pricing_source_version` stamps the pricing date so your query result
  traces back to the exact table snapshot used
- Ollama has `{input: 0, output: 0}` (legitimate zero — no cost spike alerts
  fire on Ollama traffic)
- Unknown pricing (a model not in the table): Phase 5 wiki-compiler emits
  a `spend_caveat` section listing missing models; refresh
  `packages/shared/src/pricing.ts` to resolve

## Investigation Protocols

Each protocol uses the same 4-beat shape so you know what to expect:

1. **Symptom** — how this protocol gets triggered (what tool output or user question led here)
2. **Diagnosis sequence** — the checks to run, in order, with `get_*` tool calls
3. **Likely root causes** — the 3–5 patterns this usually turns out to be
4. **Close the loop** — what to annotate, what to suggest next

### Diagnosing a slow trace

**Symptom:** a trace's `total_duration_ms` (from `get_trace`) or a span's
`duration_ms` is high relative to model expectations or your 7-day baseline.

**Quick checklist:**
`gen_ai_usage_input_tokens` → `stream?` → `finish_reason` → `tool_calls` → `attempt_count` → upstream `status_code` → `stream_truncated`

**Diagnosis sequence:**

The #1 cause of slowness is **prompt size**. LLMs scale roughly linearly with
input tokens — a 10× prompt often means 5–10× latency. Run `get_span(span_id=X)`
and look at `gen_ai_usage_input_tokens`. If it's much larger than the 7-day
median (check `get_token_usage(dimension=model, time_window=7d)`), the prompt
regressed.

If input size is normal, check whether the call was **streaming**
(`stream: true`). Non-streaming means the full response must materialize
before the first byte; streaming usually means first-token was quick and
total is long because the response itself is long. Look at
`attributes.gen_ai.response.finish_reasons` — a value of `"length"` means
the model hit `max_tokens` and was force-stopped (truncated by policy, not
by failure). A long response that didn't hit length is just a verbose model.

If `tool_calls` or `tool_use` content blocks appear in `messages`, this was
an **agent iteration** — the `duration_ms` is turn-level work, not LLM-wait
time. Run `get_trace(trace_id)` to see iteration count; long iteration chains
legitimately take minutes.

Still slow with none of those? Check `attempt_count`. A value > 1 means the
gateway retried — the turn-level duration includes upstream wait for all
attempts. Look at `status_code`: 429 = rate-limited, 503 = upstream outage;
both explain the duration without a prompt-side problem.

Finally, `attributes.stream_truncated = true` means the upstream dropped
mid-stream. Span is partial; the `duration_ms` doesn't reflect a complete
call.

**Likely root causes (in order of prior):**
1. Prompt regression (recent code commit increased input tokens)
2. Long response (verbose model; `max_tokens` not hit; not an issue per se)
3. Agent iteration (multi-step turn; measure iteration depth, not single-span duration)
4. Retry under upstream stress (attempt_count > 1)
5. Streaming truncation (partial data)

**Close the loop:** hand off to `infer-debug-trace` if the user wants the
full walkthrough for this specific trace; otherwise, `annotate_trace` with
one of the tag prefixes below.

### Diagnosing a cost spike

**Symptom:** `get_cost_stats` shows a per-(model) cost jump, or Phase 5's
`cost-spike` detection fired an insight.

**Quick checklist:**
input-vs-output breakdown → feature attribution → recent prompt commits → estimated-tokens audit

**Diagnosis sequence:**

Start with `get_token_usage(dimension=model, time_window=24h)` — compare
input vs output token totals. Three spike shapes:

- **Input tokens spiked, output normal** → prompt inflation. Someone enlarged
  the system prompt or added context. Correlate with `git log --since='<anomaly_start - 2h>'`
  on prompt files.
- **Output tokens spiked, input normal** → generations got longer. Could be
  prompt change ("generate a comprehensive report"), `max_tokens` bumped,
  or model variant swap.
- **Both spiked** → call volume is up, not per-call token cost. Check
  `list_spans(time_window=24h)` row count (or `get_insights` for a detected
  volume anomaly) to confirm.

If `metadata.feature` is populated (requires `x-infer-metadata` header on
the client), run `get_token_usage(dimension=feature, time_window=7d)` to see
which feature is consuming the tokens. A single feature accounting for >70%
of spend is a concentration risk (and a lever).

Check `attributes.gen_ai.usage.estimated` — if most spans in the spike are
estimated, the count is approximate. Don't raise fire-level alarms on a 10%
variation when the count is ±5% approximate.

**Likely root causes:**
1. Prompt regression (system prompt or context builder enlarged)
2. Model upgrade (team switched from mini to full)
3. Call-volume spike (new feature shipped; new user segment; retry bug)
4. Loss of `stream_options.include_usage: true` on the client (tokens still
   estimated but now less accurate — spike is apparent, not real)

**Close the loop:** annotate the anomaly trace or thread with the finding;
suggest either rollback (if prompt commit) or rate-limiting (if volume).

### Diagnosing a new error_type

**Symptom:** `get_error_spans` shows an `error_type` that wasn't in the
7-day baseline, or Phase 5's `new-error-type` detection fired.

**Quick checklist:**
sample span → attributes deep-dive → correlate with recent deploys

**Diagnosis sequence:**

Run `get_error_spans(error_type="<name>", limit=5)` to pull a handful of
representative spans. Inspect one via `get_span(span_id=X)`:

- **`upstream_host`** — does this pattern affect only one provider? Often
  yes.
- **`status_code`** — 429 = rate limit (check your upstream plan tier); 500–502
  = upstream outage (usually transient, self-heals); 400 = malformed request
  (check `messages` payload); 404 = bad path (routing error).
- **`attributes`** JSONB — full details of what upstream returned in
  `upstream_response`, any `x-ratelimit-*` headers, etc.

Once you've characterized the error shape, run the provided
`correlation_hint` from the insight's `evidence` (or manually):
```bash
git log --since='<anomaly_start - 2h>' --until='<anomaly_end + 30m>' --oneline
```
Commits close in time to the first-seen timestamp are strongly associated
with new error types. Revert the suspect commit locally and see if it fixes
the error.

**Likely root causes:**
1. Deploy broke the request shape (recent code commit)
2. New upstream error class (provider changed error codes — rare but happens)
3. Upstream rate-limit threshold crossed (your traffic grew into a tier)
4. Network / DNS / infrastructure blip (transient 5xx; will self-heal)

**Close the loop:** if deploy-related, annotate with `Root cause: commit
abc123 <one-line description>`. If transient, `Observation: upstream 5xx
cluster in <time window>, self-healed, no action`.

### Diagnosing a retry storm

**Symptom:** `attempt_count > 1` on many spans, or Phase 5's `retry-storm`
detection fired.

**Quick checklist:**
`attempt_count` distribution → upstream status distribution → biscuit-side vs upstream-side attribution

**Diagnosis sequence:**

Run `get_error_spans(limit=20, time_window=1h)` and look at the
`attempt_count` distribution:
- Most spans at `attempt_count=1` (steady state) → storm is isolated to one
  (provider, model)
- Spike at `attempt_count=2–4` → gateway retried within its 3-retry budget
- Spike at `attempt_count=5+` → something's hammering the upstream (this is
  abnormal — the gateway caps retries)

Check the `status_code` distribution on the failed attempts:
- **429** → hitting the upstream's rate limit. Tune your client's concurrency
  or upgrade the provider tier.
- **503** / **502** → upstream outage. Check the provider's status page.
  Usually self-heals.
- **No pattern** → maybe the retry is *your client's* retry, not the gateway's.
  The gateway only retries on specific error codes; if your application logic
  wraps fetch with its own retry, `attempt_count` may not capture the full
  picture.

**Likely root causes:**
1. Upstream rate-limited (check tier; the gateway retries with backoff, but
   if you're persistently over the quota, it'll keep retrying)
2. Upstream outage (transient; self-heals)
3. Client-side retry loop compounding with gateway-side retry (double-retry
   pattern — 9 real attempts on the upstream for each 3-attempt gateway sequence)

**Close the loop:** if rate-limit, annotate with `Root cause: upstream rate
limit, consider tier upgrade or concurrency cap`. If outage, `Observation:
upstream outage <time window>, waited out`.

### Interpreting `gen_ai.usage.estimated=true`

**Symptom:** cost or token analysis shows `estimated: true` on many spans.

**When it's OK:**
- **Ollama** — Ollama Cloud never sent usage in streaming responses prior to
  recent versions; estimation is the norm, accurate within a few percent.
- **Non-streaming calls** — should never be estimated (upstream always
  returned usage). If you see it, something went wrong in the gateway's
  parsing. File an issue.

**When to push the user toward `include_usage`:**
- Streaming OpenAI / Anthropic calls that matter for billing accuracy
- Per-session or per-user cost analysis where ±5% error compounds into
  material mis-attribution
- Suggest the user pass:
  ```ts
  await openai.chat.completions.create({
    model: "gpt-4o",
    messages: [...],
    stream: true,
    stream_options: { include_usage: true }, // <-- this line
  });
  ```

## Pre-Ship / Post-Ship Watch (Preventive Protocol)

The four Investigation Protocols above are diagnostic — they surface root
causes *after* something changed. This one is **preventive**: capture
baselines *before* a change so regressions are detectable *after* it.

### Symptom / trigger

User said something like:
- "I'm about to ship a prompt change — anything I should watch?"
- "We're swapping `gpt-4o` for `gpt-4o-mini` next week — what should I
  measure?"
- "Adding a tool to the agent tomorrow — how do I make sure it doesn't
  regress?"
- "Pre-deploy checklist, please"

### Step 1 — capture pre-ship baseline (3 parallel calls)

```
get_latency_stats(dimension=model, time_window=7d)
get_token_usage(dimension=model,  time_window=7d)
get_cost_stats(dimension=model,    time_window=7d)
```

These are independent reads — fire them in parallel.

Surface prominently to the user:
- **p95 latency** — primary regression signal. Phase 5's `latency-regression`
  detection fires on +50% vs baseline with statistical significance; a
  cron-detected insight is the cleanest post-ship alarm.
- **Average input_tokens per span** — `input_tokens / count`. Direct
  prompt-size baseline; the most sensitive signal for prompt regressions.
- **Error rate** — fetch from `get_project_summary()`. Catches response-
  shape regressions that don't move latency.
- **`estimated_fraction`** — if `get_token_usage` reports >30% estimated,
  warn the user: post-ship token deltas will be noisy. Suggest
  `stream_options: { include_usage: true }` on the client before the ship
  if the deltas need to be precise.

### Step 2 — ship the change

User runs the deploy. No tool calls here; wait 1–2 hours so the gateway
accumulates 20–30 new spans on the new config. If the user wants the
watch automated, suggest `/loop` to re-run this skill hourly for 24h.

### Step 3 — post-ship re-check

```
get_latency_stats(dimension=model, time_window=1h)
get_token_usage(dimension=model,  time_window=1h)
get_insights()
```

**Thresholds for concern (relative to the 7d baseline from Step 1):**
1. p95 in the 1h window ≥ baseline p95 × 1.5 → a `latency-regression`
   insight is likely to fire on the next hourly cron tick; don't wait for
   it, start investigating the diff.
2. Avg input_tokens/span materially above baseline → prompt-size regression.
   Investigate the diff before more spans accumulate on the new prompt.
3. Error rate in the 1h window ≥ baseline × 2 → response-shape regression.
   Run `get_error_spans(limit=10, time_window=1h)` to identify the new
   failure mode.

### Likely regression shapes (interpretation)

1. **p95 up + input_tokens up** → prompt got bigger. Expected if you
   enlarged the system prompt or added context; investigate whether the
   size jump is intentional or accidental.
2. **p95 up + input_tokens stable** → non-prompt slowdown. Model-variant
   swap, upstream routing change, or tool-iteration bloat.
3. **Error rate up** → downstream code's response-shape assumption broke.
   Check `get_error_spans` for the new `error_type`; often a schema drift
   in tool-call arguments or finish-reason handling.
4. **Cost up disproportionately to call volume** → model swap (e.g. `mini`
   → full) OR prompt inflation concentrated in a specific feature (check
   `get_cost_stats(dimension=feature)` if `metadata.feature` is populated).

### Close the loop

- **No regression:** `annotate_trace` on a representative post-ship trace
  with `Observation: post-ship baseline stable — p95=<X>, input_tokens_avg=<Y>,
  error_rate=<Z>, within 7d pre-ship baseline.` Useful reference for the
  next ship-watch on this project.
- **Regression confirmed:** the `latency-regression` /
  `token-consumption-spike` / `new-error-type` insight's
  `evidence.trace_id_samples[0]` hands off cleanly to `infer-debug-trace`.
- **Regression suspected but no insight yet:** set up `/loop` to re-run
  `get_insights()` hourly for 24h. Most anomalies accumulate statistical
  significance within 2–6 hours of ship; waiting past 24h with no insight
  generally means there's no real regression.

## Anti-Patterns — What NOT to Do

1. **Don't cite absolute latency thresholds as pass/fail.** These drift.
   Always check against the user's own 7-day baseline via `get_latency_stats`.
   The ranges in this skill are sanity checks, not thresholds.
2. **Don't claim causation from a single trace.** One slow trace at 2:03pm
   does not mean "every trace after 2:03pm is slow." You need a pattern —
   regression across n > 30 spans, Phase 5 insight with statistical
   significance, or an observable user-facing change.
3. **Don't use p99 when n < 100.** Noise dominates; the p99 estimate swings
   by 50–100% with each new call. Use p95.
4. **Don't ignore `gen_ai.usage.estimated=true` when cost-analyzing.**
   Estimated counts are approximate (~2–5% error). Material decisions need
   exact counts — either rely on non-streaming calls or push the client
   toward `include_usage`.
5. **Don't annotate before investigating.** The annotation is the *last* step
   of the investigation arc. Premature annotations (written to look thorough)
   become confidently wrong facts that mislead future sessions.

## Annotation Convention

After every investigation, call `annotate_trace(trace_id, content)` or
`annotate_span(span_id, content)`. Lead the content with one of four tag
prefixes so future agents can filter by intent:

- **`Root cause:`** — confirmed with evidence (deploy commit, upstream
  incident, tool-call pattern). Example: `Root cause: commit abc123
  enlarged system prompt 3×; duration tripled from ~1s to 3.5s at that
  timestamp.`
- **`Observation:`** — pattern noticed but not a direct cause. Example:
  `Observation: attempt_count=3 across retries; upstream 503s clustered
  14:02-14:08 — transient Ollama outage, self-healed.`
- **`Inconclusive:`** — investigated, no systemic cause found. Still useful
  — prevents re-investigating. Example: `Inconclusive: prompt size 1200
  tokens (within 7d p50 of 1400), upstream 200 OK, attempt_count=1. One-off
  latency outlier; no systemic cause found.`
- **`Pending user:`** — investigation blocked on user-side info. Example:
  `Pending user: asked user to check Ollama rate-limit tier in cloud.ollama.com.`

Annotations are persistent across sessions and appear in
`get_project_summary` under active threads. They compound — future agents
see "this trace already investigated, here's what we learned."

## When to Hand Off to `infer-debug-trace`

Use `infer-debug-trace` when:
- The user says "diagnose trace X" or provides a specific `trace_id`
- You've identified an insight worth drilling into (e.g., `get_insights`
  returned an anomaly with `evidence.trace_id_samples`; user approved
  follow-up)
- The user says "why was this slow" / "why did this fail" without a specific
  ID — `infer-debug-trace`'s 3-tier resolution (explicit → sniff recent
  context → narrow-down) handles it

Pass the `trace_id` explicitly when you have one. Example handoff:
```
Invoke infer-debug-trace with trace_id=550e8400-e29b-41d4-a716-446655440000
```

`infer-debug-trace` cross-references this skill's protocols for the actual
diagnosis steps — it's the "execute the protocol against this specific trace"
layer.

## Rotating Tips

Pick one for the end of your response. Don't repeat within a session.

- `💡 Tip: Prompt size is the #1 cause of LLM latency — check gen_ai_usage_input_tokens first`
- `💡 Tip: attempt_count > 1 means the gateway retried; root cause is upstream-side`
- `💡 Tip: Annotate with a tag prefix so future sessions can filter — Root cause / Observation / Inconclusive / Pending user`
- `💡 Tip: Estimated tokens (gen_ai.usage.estimated=true) means your client isn't passing include_usage — cost is approximate`
- `💡 Tip: p99 is noisy when n < 100 — cite p95 instead`
