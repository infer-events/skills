---
name: infer-insights
description: Use when the user asks for analytics insights, a health check, a pulse on their app, or says "what should I know about my data." Also use when invoked via /schedule or /loop for automated daily insights.
---

# Infer Insights — Automated Discovery

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

## Output Format

Present insights in this exact format:

```
## Infer Insights — [Date]

**Data window:** [time range] | **Total events:** [N]

### 1. [Headline — the insight in one sentence]
**Impact: [score]/125** | Surprise: [N] | Leverage: [N] | Confidence: [N]

[2-3 sentences explaining what you found, what it means, and the suggested action.]

### 2. [Next insight]
...

### 3-5. [Remaining insights]
...

---
**Next check:** [When the next automated run is scheduled, if applicable]
**Deep dive:** Ask me to investigate any of these further.
```

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
