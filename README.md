# Infer Skills

Agent skills for [Infer](https://infer.events) — LLM observability for AI
agents. One `baseURL` change, every call instrumented, insights pushed
hourly. No dashboards, no SDK integration.

## Install

```
npx skills add infer-events/skills
```

Then run `/infer-setup` in Claude Code. The wizard handles everything:
- Creates your Infer account (or signs in)
- Detects your LLM client (`openai`, `@anthropic-ai/sdk`, `ai` + `@ai-sdk/*`, `ollama`)
- Proposes the 3-line `baseURL` change
- Applies it (with your approval) + live-verifies the first span lands
- Configures the MCP server
- Walks you through removing the legacy `@inferevents/sdk` if present

## Skills included

| Skill | Command | What it does |
|---|---|---|
| **infer-setup** | `/infer-setup` | Full setup wizard — signup, detect LLM client, 3-line baseURL change, live-verify, MCP config |
| **infer-observability** | Auto-loaded when you ask about LLM data | Interprets MCP results — latency benchmarks, error norms, investigation protocols, anti-patterns |
| **infer-insights** | "Give me insights" / scheduled daily | Proactive briefing — pre-computed anomalies via Phase 5 hourly cron + thread context |
| **infer-debug-trace** | "Diagnose trace X" / "why was this slow" | Live investigation on a specific trace — 3-tier entry + 4-beat arc + annotated root cause |
| **infer-upgrade** | `/infer-upgrade` | Updates MCP server + skills to latest versions |

## How it works

Infer sits in the network path between your agent and the LLM provider:

```
 Your agent                gateway.infer.events             OpenAI / Anthropic / Ollama
      │                           │                                  │
      │  POST /v1/{proj}/openai/  │                                  │
      │  Authorization: sk-...    │                                  │
      │  ───────────────────────▶ │                                  │
      │                           │  POST /v1/chat/completions        │
      │                           │  Authorization: sk-...            │
      │                           │  ───────────────────────────────▶ │
      │                           │  ◀──────────────────────────────  │
      │                           │  span → api.infer.events          │
      │  ◀─────────────────────── │                                  │
      │  (your agent gets the     │                                  │
      │   upstream response)      │                                  │
```

One `baseURL` change. That's the whole integration. Every call captures:

- OpenTelemetry GenAI span (30 columns of structured data)
- Messages (request + response) as JSONB, or NULL if `x-infer-redact: true`
- Latency, tokens, cost, status code, error type, retry count, stream
  truncation flag

The Phase 5 cron detects 10 anomaly classes hourly. The MCP server lets
your agent query spans directly. Skills teach the agent how to interpret
findings and run investigations.

## Example: briefing

```
You: "how's the agent doing today?"

Agent calls get_project_summary() + get_insights(), presents:

  Infer Pulse — 2026-04-19
  Spans last 24h: 1,247 | Error rate: 1.5% | Active threads: 1

  [!!] CRITICAL: openai/gpt-4o p95 up 58% since Wed 14:00 UTC
       n=1247, p_value_adjusted=0.0003, baseline=890ms, current=1410ms
       Likely cause: commit abc123 "refactor prompt builder" (3h before)
       → Invoke infer-debug-trace on trace_id=550e8400-... to diagnose?
```

Without skills: you get raw tool output; you run the investigation. With
skills: the agent runs it.

## Example: debug

```
You: "diagnose trace bda7a45e-d2d4-4d82-9c7a-e963e86bd9c2"

Agent (via infer-debug-trace):
  Phase 1: get_trace → 3 spans, total 181s, all status 200
  Phase 2: Classify → slow trace protocol
  Phase 3: get_span on the longest → input_tokens=48000 (baseline p50 is 1200)
  Phase 3 (cont): git log --since='2026-04-19T12:00:00Z' → commit abc123
  Phase 4: annotate_trace(trace_id, "Root cause: commit abc123 enlarged
           system prompt 40x; input_tokens 1200 → 48000; duration tripled.
           Fix: revert or cap prompt size.")
```

## Links

- **Documentation:** https://infer.events
- **MCP server:** https://www.npmjs.com/package/@inferevents/mcp
- **Public mirror (gateway + MCP + shared):** https://github.com/infer-events/infer

## Migrating from v1.x (web-analytics era)

If you were using Infer's web-analytics SDK (`@inferevents/sdk`), the
2.0.0 release is a clean break:

- The SDK is being deprecated (Phase 8 of the gateway pivot)
- Skill tools (`get_event_counts`, `get_retention`, `get_top_events`, etc.)
  are retired; the 7 deprecation shims return `isError: true` envelopes
  until 2026-06-17
- Run `/infer-setup` — the wizard detects `@inferevents/sdk` and walks you
  through removing it + switching to the gateway
