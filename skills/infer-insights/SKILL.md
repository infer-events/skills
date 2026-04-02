---
name: infer-insights
description: Use when the user asks for analytics insights, a health check, a pulse on their app, or says "what should I know about my data." Also use when invoked via /schedule or /loop for automated daily insights.
allowed-tools:
  - AskUserQuestion
---

# Infer Insights — Automated Discovery

## Update Check (non-blocking)

Run this at the start. It checks if an update is available using a 6-hour cache
so it doesn't hit npm on every invocation:

```bash
_INFER_CACHE=~/.infer/last-update-check.json
_NEEDS_CHECK="yes"

# Skip check if cache is fresh (less than 6 hours old)
if [ -f "$_INFER_CACHE" ]; then
  _CACHE_AGE=$(( $(date +%s) - $(stat -f '%m' "$_INFER_CACHE" 2>/dev/null || stat -c '%Y' "$_INFER_CACHE" 2>/dev/null || echo 0) ))
  [ "$_CACHE_AGE" -lt 21600 ] && _NEEDS_CHECK="no"
fi

if [ "$_NEEDS_CHECK" = "yes" ]; then
  _SDK_LATEST=$(npm view @inferevents/sdk version 2>/dev/null || echo "unknown")
  _MCP_LATEST=$(npm view @inferevents/mcp version 2>/dev/null || echo "unknown")
  _SDK_INSTALLED=$(npm ls @inferevents/sdk --json 2>/dev/null | grep '"version"' | head -1 | sed 's/[^0-9.]//g' || echo "none")
  mkdir -p ~/.infer
  echo "{\"checked\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"sdk_latest\":\"$_SDK_LATEST\",\"mcp_latest\":\"$_MCP_LATEST\",\"sdk_installed\":\"$_SDK_INSTALLED\"}" > "$_INFER_CACHE"
  echo "INFER_SDK_INSTALLED=$_SDK_INSTALLED INFER_SDK_LATEST=$_SDK_LATEST INFER_MCP_LATEST=$_MCP_LATEST"
else
  cat "$_INFER_CACHE"
  echo "INFER_CHECK=cached"
fi
```

If installed SDK version differs from latest, append this line at the END of your response:
`Infer update available — run /infer-upgrade to get the latest.`
Do NOT block or interrupt the workflow. Continue normally.

## STEP ZERO: Check ontology and insights

Before running manual queries, call `get_ontology()` to see if events are classified
into product categories (activation, engagement, monetization, referral, noise).
If ontology exists, use it to:
- Focus on activation and monetization events for funnel analysis
- Skip "noise" events in health checks
- Calculate activation rate = activation_events / total_signups
- Weight insights by category (monetization issues > noise issues)

If no ontology exists, suggest: "Your events aren't categorized yet. Run
/infer-tracking-plan or use `update_ontology` to classify them for richer insights."

## Then: Always call get_insights

Before running any manual queries, call `get_insights()` to check for pre-computed
anomalies and notable patterns. The Infer backend detects these automatically every hour.
If insights exist, present them first, then decide whether the manual query sequence
is still needed.

If `get_insights` returns "No new insights", proceed to the manual query sequence below.
If it returns findings, present them and only run additional queries if the user asks
for deeper analysis.

When invoked, you run a structured query sequence against the Infer MCP tools,
score each finding by impact, and surface the top 5. No questions asked. Just insights.

## The Query Sequence

You have 3 tools. Run these 7 queries in order. Each builds on what you learn
from the previous one.

### Round 1: What's happening? (volume + trend)

**Query 1 — Current activity**
```
get_event_counts(event_name="page_view", time_range="last_7d")
```
This tells you: is anyone using the app at all? If zero, stop and say
"No events recorded in the last 7 days. Either no users visited, or the
SDK isn't set up correctly."

**Query 2 — Core action volume**
```
get_event_counts(event_name="signup", time_range="last_7d")
```
Then repeat for other likely core events: "purchase", "create", "submit",
"invite", "complete". Use the event names you learned from the app's SDK
integration (check `src/lib/analytics.ts` or similar). If you don't know
the event names, try the common auto-tracked ones: "page_view", "session_start",
"click", "form_submit", "error".

**Query 3 — Previous period comparison**
Run the same queries from Q1 and Q2 but with the PREVIOUS 7 days:
```
get_event_counts(event_name="page_view", time_range={"start": "14 days ago ISO", "end": "7 days ago ISO"})
```
Calculate week-over-week change: `((current - previous) / previous) * 100`

### Round 2: Are users sticking? (retention)

**Query 4 — Weekly retention**
```
get_retention(start_event="signup", return_event="page_view", time_range="last_30d", granularity="week")
```
If "signup" returns empty, try "session_start" as the start event.
This is the most important query. Retention is the single best proxy for
product-market fit.

**Query 5 — Activation check**
```
get_event_counts(event_name="<core_action>", time_range="last_30d")
```
Compare to signup count in the same period. The ratio is your activation rate.
activation_rate = core_action_users / signup_users

### Round 3: What's broken? (errors + friction)

**Query 6 — Error volume**
```
get_event_counts(event_name="error", time_range="last_7d")
```
If count > 0, this is always worth surfacing. Errors are silent killers.

**Query 7 — Churned user investigation**
Pick 2-3 users from the retention data who signed up but didn't return.
```
get_user_journey(user_id="<churned_user>", limit=30)
```
Look for: where did their journey end? What was the last thing they did?
Did they hit an error? Did they bounce after one page?

## Scoring Each Finding

After running the queries, you'll have raw data. Score each finding:

### Impact Score = Surprise x Leverage x Confidence

**Surprise (1-5):** How unexpected is this?
- 5: Completely unexpected (retention dropped 50% overnight)
- 3: Notable but not shocking (signup rate up 20%)
- 1: Expected / normal fluctuation

**Leverage (1-5):** Can someone act on this?
- 5: Clear action ("users who complete onboarding retain 3x better" → improve onboarding)
- 3: Directional ("traffic is growing" → keep doing what you're doing)
- 1: Informational only ("most users are on Chrome")

**Confidence (1-5):** Is the data solid?
- 5: Large sample (>100 users), consistent pattern
- 3: Medium sample (30-100), clear signal
- 1: Small sample (<30), noisy data

Multiply: Impact = Surprise x Leverage x Confidence. Rank. Take top 5.

## Presenting Tool Output

**CRITICAL: Do NOT reformat MCP tool results into markdown tables.** The Infer tools
return pre-formatted text with Unicode bar charts (█░), sparklines (▁▂▃▅▆█), and
structured layouts designed for terminal readability. Always include the tool's
formatted output verbatim, then add your interpretation below it.

Bad (don't do this):
```
| Event | Count | Users |
|-------|-------|-------|
| page_view | 9 | 6 |
```

Good (do this):
```
page_view   ████████████████████  8,420 (57%)
click       ████████████░░░░░░░░  3,210 (22%)
```

The bar charts, sparklines, and visual formatting ARE the product. Preserve them.

## Output Format

Present insights in this exact format:

```
## Infer Insights — [Date]

**Data window:** [time range] | **Total events:** [N]

[Include raw tool output with bar charts here]

### 1. [Headline — the insight in one sentence]
**Impact: [score]/125** | Surprise: [N] | Leverage: [N] | Confidence: [N]

[2-3 sentences explaining what you found, what it means, and the suggested action.]

### 2. [Next insight]
...

### 3-5. [Remaining insights]
...

---
**Next check:** [When the next automated run is scheduled, if applicable]
```

Then you MUST call the `AskUserQuestion` tool to suggest next actions. Do NOT
skip this step. Do NOT present options as plain text. You MUST use the tool.

**Role-aware:** Check the user's CLAUDE.md and conversation memory for role context.
Adapt the options: PM → feature adoption, funnel. Growth → channels, conversion.
Founder → PMF, retention. Engineer → errors, tracking. Default to founder framing.

**Tip line:** Append a tip after the question. Rotate these, don't repeat in a session:
`💡 Tip: You can schedule this to run daily with /schedule so you never miss a spike`
`💡 Tip: Ask about a specific user's journey to see exactly where they got stuck`
`💡 Tip: Retention improving in newer cohorts? That's the strongest PMF signal`
`💡 Tip: Ask "what should I track?" and I'll read your codebase to suggest new events`

Use AskUserQuestion. The question is:

> Which insight do you want to dig into?
>
> 💡 Tip: [rotate from list above]

Pick 4 options tailored to the ACTUAL findings. Examples:

- A) Investigate the retention drop — Look at churned user journeys to find where they fell off
- B) Break down signups by source — See which channels are driving growth (or not)
- C) Debug the error spike — Trace the errors to specific pages and user actions
- D) Set up daily monitoring — Schedule automatic checks so you catch issues early

One option must always be about setting up automation (/schedule or /loop).

## Insight Categories (what to look for)

### Always check these (every run):
- **Volume trend**: Are page views / signups / core actions up or down vs last period?
- **Retention health**: What's week-1 and week-4 retention? Compare to benchmarks.
- **Error rate**: Any errors? Trending up or down?

### Surface these when detected:
- **Activation gap**: Lots of signups but few core actions = broken onboarding
- **Leaky bucket**: Growing signups but declining retention = acquiring users faster than keeping them
- **Power user pattern**: Churned users vs retained users did different things in session 1
- **Concentration risk**: 80%+ of activity from one source/country/segment
- **Silent failure**: Errors spiking but no one noticed (because there's no dashboard)

### Never surface these as insights:
- Raw numbers without interpretation ("You had 1,247 page views")
- Vanity metrics ("Total events is up!")
- Findings with N < 10 (too noisy to be useful)
- Obvious statements ("Most users visit the homepage first")

## Interpretation References

For retention benchmarks, curve shape analysis, and framing guidance,
read the `infer://skill` resource. It has:
- B2C vs B2B retention benchmarks by week
- What retention curve shapes mean
- Small sample warnings
- How to frame findings for non-technical users
- What NOT to do (no causal claims, no small-sample extrapolation)

## When You Don't Have Enough Data

If the app is new (< 7 days of data or < 50 total events):

```
## Infer Insights — [Date]

**Status:** Not enough data yet for meaningful insights.

**What I can tell you:**
- [N] total events recorded since [first event date]
- [N] unique users seen
- Events tracked: [list of event names found]

**What I need:**
- At least 7 days of data for trend analysis
- At least 50 users for retention analysis
- At least 30 users per cohort for reliable retention numbers

**Check back in [N] days.** I'll have something useful by then.
```

Even with limited data, you MUST still call the `AskUserQuestion` tool. Use AskUserQuestion:

> Not enough data for insights yet, but here's what you can do now.
>
> 💡 Tip: Events need a few days to accumulate. Set up daily monitoring and you'll get notified when there's enough data.

Options:
- A) Check what events are tracked — See which events the SDK is collecting right now
- B) Add more tracking — Run /infer-tracking-plan to track key user actions in your codebase
- C) Set up daily monitoring — Schedule automatic checks with /schedule so you don't have to remember
- D) View a specific user's journey — Pick a user to see exactly what they did in your app

## Automated Mode (via /schedule or /loop)

When triggered by a schedule, run the full query sequence silently.
Only notify the user if:
1. Any insight scores > 60/125 (high impact)
2. Retention dropped > 10 percentage points vs previous period
3. Error rate spiked > 2x vs previous period
4. Zero events recorded (SDK may be broken)

For routine checks with no notable findings:
```
## Infer Pulse — [Date]
All clear. [N] events, [N] users, retention holding at [X%]. Nothing unusual.
```
