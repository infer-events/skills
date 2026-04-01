# Infer Skills

Agent skills for [Infer](https://infer.events) — analytics for your AI, not AI for your analytics. Insights pushed hourly. No dashboards.

## Install

```
npx skills add infer-events/skills
```

Then run `/infer-setup` in Claude Code. The wizard handles everything:
- Creates your Infer account
- Installs the MCP server
- Installs the SDK in your project
- Detects your framework and adds tracking
- Verifies events are flowing

## Skills included

| Skill | Command | What it does |
|-------|---------|-------------|
| **infer-setup** | `/infer-setup` | Full setup wizard — signup, install, integrate, verify |
| **infer-analytics** | Auto-loaded when querying | Teaches agents to interpret retention, funnels, benchmarks |
| **infer-insights** | "Give me insights" | Checks auto-detected anomalies first, then runs discovery |
| **infer-tracking-plan** | "What should I track?" | Reads your codebase, proposes events with file:line references |
| **infer-upgrade** | `/infer-upgrade` | Updates SDK, MCP server, and skills to latest versions |

## How it works

Infer detects insights automatically every hour. Your agent checks them first:

```
Agent calls get_insights, receives pre-computed finding:

  Insights — 2 new
  ──────────────────────────────────────────────────

  [!!] CRITICAL: page_view dropped 52% vs 7-day average (120 today vs avg 250)
      event_name=page_view, today_count=120, avg_7d=250
      detected: Apr 1, 02:00 PM

  [!] NOTABLE: error spiked 4x in the last hour (28 vs hourly avg of 7)
      last_hour=28, avg_hourly=7, multiplier=4
      detected: Apr 1, 02:00 PM
```

Or ask a direct question:

```
You: "What's my retention?"

Agent calls get_retention, reads infer-analytics skill, interprets:

  Retention: signup → page_view
  weekly cohorts | last_30d

  Mar 3 (142 users)
    week 0  █████████████████████████  100%
    week 1  ██████████░░░░░░░░░░░░░░░  42% 🟢
    week 4  ███░░░░░░░░░░░░░░░░░░░░░░  12% 🔴

  Week-4 is 12%, below the 20% B2C benchmark.
  Check your onboarding flow.
```

## Architecture

```
Skills (this package)          MCP Server (@inferevents/mcp)       SDK (@inferevents/sdk)
~/.claude/skills/infer/        Connected as MCP server              Installed in your app

                               get_insights (hourly push)
infer-setup     ──────────►    get_event_counts                     init()
infer-analytics ──────────►    get_retention        ◄──────────     track()
infer-insights  ──────────►    get_user_journey                     identify()
infer-tracking-plan            get_top_events                       page()
infer-upgrade                  (updates all components)

                               Cloudflare Cron (hourly)
                               ────────────────────────
                               8 insight queries per project
                               → writes to insights table
                               → get_insights reads them
```

Skills teach the agent WHAT to do. MCP tools do the queries. The cron detects anomalies. SDK collects the data.
