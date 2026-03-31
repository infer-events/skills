# Infer Skills

Agent skills for [Infer](https://infer.events) — headless analytics for AI-first developers.

## Install

```
npx skills add @inferevents/skills
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
| **infer-insights** | "Give me insights" | Automated discovery of top 5 impactful findings |
| **infer-tracking-plan** | "What should I track?" | Reads your codebase, proposes events with file:line references |

## How it works

```
You: "What's my retention?"

Agent reads infer-analytics skill, calls get_retention MCP tool, interprets:

  Retention: signup → page_view
  weekly cohorts | last_30d

  Mar 3 (142 users)
    week 0  █████████████████████████  100%
    week 1  ██████████░░░░░░░░░░░░░░░  42% 🟢
    week 4  ███░░░░░░░░░░░░░░░░░░░░░░  12% 🔴

  Week-4 is 12%, below the 20% B2C benchmark.
  Check your onboarding flow.

  💡 Dig deeper: ask "Show me the journey of a user who churned"
```

## Architecture

```
Skills (this package)          MCP Server (@inferevents/mcp)       SDK (@inferevents/sdk)
~/.claude/skills/infer/        Connected as MCP server              Installed in your app

infer-setup     ──────────►    get_event_counts                     init()
infer-analytics ──────────►    get_retention        ◄──────────     track()
infer-insights  ──────────►    get_user_journey                     identify()
infer-tracking-plan            get_top_events                       page()
```

Skills teach the agent WHAT to do. MCP tools do the actual queries. SDK collects the data.
