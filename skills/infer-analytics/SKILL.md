---
name: infer-analytics
description: Use when the user asks questions about their analytics data, event counts, retention, user journeys, or funnels. Teaches agents how to interpret Infer MCP tool results.
allowed-tools:
  - AskUserQuestion
---

# Infer Analytics — Agent Skill Guide

You have access to product analytics through the Infer MCP server. This guide teaches you how to query data effectively and interpret results like an experienced product analyst.

## Update Check (non-blocking)

Run this at the start. It checks if an update is available using a 6-hour cache
so it doesn't hit npm on every invocation:

```bash
_INFER_CACHE=~/.infer/last-update-check.json
_NEEDS_CHECK="yes"
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

If installed SDK version differs from latest, append at the END of your response:
`Infer update available — run /infer-upgrade to get the latest.`
Do NOT block the workflow. Continue normally.

## Proactive Briefing (ALWAYS DO THIS FIRST)

Before answering any analytics question, get context first, then alerts.

### Briefing Flow

1. Call `get_project_summary()` — full context: health score, metrics, event catalog, funnel
   performance, and active threads with annotations from prior investigations
2. Call `get_insights()` (no severity filter — get everything)
3. If the response says "No new insights", skip the briefing and proceed to the user's question
4. If insights exist, present them as a prioritized briefing:

**For CODE ACTIONS:** Before presenting, use the `correlation_hint` to run a git command:
```bash
git log --since='7 days ago' --all --oneline -- '**/*KEYWORD*'
```
Replace KEYWORD with the event name or flow name from the hint. If commits are found, present the most recent one as "Likely cause." If no commits found, say "No related commits found."

**Briefing format (present verbatim from tool output, then add git correlation):**

For each code action, append the git finding:
```
Likely cause: abc123 "refactor signup form" (2 days ago)
→ Want me to investigate this commit?
```

Or if no commits found:
```
No related commits found.
→ Want me to check the code for this flow?
```

For strategy actions, present as-is — no git correlation needed.

4. After investigating a CODE ACTION, **always call `annotate_thread(thread_id, content)`**
   to record your findings. The `thread_id` is shown in the insight output. Your annotation
   will appear in future briefings and the project summary, so the next session has context.

5. After the briefing, use `AskUserQuestion` to let the user pick which action to pursue, or continue to their original question.

### When to Skip the Briefing
- If the user explicitly asks a specific question ("how many signups last week?"), present the briefing first but keep it brief, then answer their question
- If insights were already shown in this session (don't re-show on every query)

### Confidence Caveats
- For insights marked as "low" confidence, append: "(based on limited data — treat as directional)"
- For "medium" confidence, no caveat needed
- For "high" confidence, no caveat needed

## Decision Context: Check Data Before Editing Tracked Code

When you are about to edit a file, **check if it contains `track()` or `infer.track()` calls first.** If it does, query the relevant events before making changes so you can warn the developer about impact.

### How it works

1. Before editing a file, scan it for `track('event_name'` patterns
2. For each tracked event found, call `get_event_counts(event_name, time_range="last_7d")`
3. If any event shows significant volume (100+ events/week) or a notable trend, mention it:

```
FYI — this file tracks 'checkout_completed' which had 342 events last week (up 15% from previous week).
Proceeding with the edit.
```

4. If an event is trending down, flag it more prominently:

```
Heads up — 'signup_completed' in this file dropped 40% last week (89 vs 150).
This might be related to the change you're about to make, or it could be pre-existing.
Want me to investigate before we proceed?
```

### When to do this
- When editing files that contain `track()` calls
- When refactoring or deleting code near tracking calls (risk of breaking tracking)
- When the user asks you to modify a component that has analytics instrumentation

### When NOT to do this
- When the user is just reading/exploring code (no edit intent)
- When the tracked event has very low volume (< 10 events/week) — not worth flagging
- When you've already checked this file earlier in the session
- When the user explicitly says to skip the check

### Keep it brief
This is context, not a blocker. One line is enough. Don't turn every edit into an analytics review. The goal is to make the developer aware, not to slow them down.

## Presenting Tool Output

**CRITICAL: All Infer MCP tools return pre-formatted text with Unicode bar charts (█░),
sparklines (▁▂▃▅▆█), and structured layouts. ALWAYS present the tool output verbatim
in your response.** Do not reformat into markdown tables. The visual formatting is
the product experience. Add your interpretation and insights AFTER the raw output.

## Tool Routing — Which Tool for Which Question

### Use `get_event_counts` when asked about:
- "How many X happened?" — count a single event
- "What's our signup rate?" — count signups over a time range
- "Break down purchases by country" — count with group_by
- "Compare this week vs last week" — run two queries with different time_ranges
- "What's our most popular feature?" — count multiple events or group by event property
- "Which channel brings the most signups?" — group_by: "utm_source"
- "Which landing page converts best?" — group_by: "landing_page"
- "Break down by campaign" — group_by: "utm_campaign"
- Anything involving totals, volumes, frequencies, or distributions

**Available group_by fields:** utm_source, utm_medium, utm_campaign, utm_term, utm_content, landing_page, country, city, os, device_type, browser, referrer, pathname. Also: "source" (alias for utm_source), "medium" (alias for utm_medium), "campaign" (alias for utm_campaign).

### Use `get_retention` when asked about:
- "Are users coming back?" — classic retention question
- "What's our week-over-week retention?" — cohort analysis
- "Do users who sign up actually stick around?" — start_event=signup, return_event=any active event
- "Is our product sticky?" — retention is the answer
- "How does retention compare month over month?" — look at cohort trends
- Anything involving repeat usage, churn, stickiness, or engagement over time

### Use `get_ontology` when asked about:
- "What categories are my events in?" — view the event ontology
- "How is my funnel structured?" — see activation, engagement, monetization stages
- "Which events are noise?" — filter by category
- "What's my product funnel?" — ontology shows the event-to-category mapping
- Anything about event classification, product categories, or funnel structure

### Use `update_ontology` when:
- The user wants to classify events into categories (activation, engagement, monetization, referral, noise)
- After reviewing `get_top_events`, propose categories for unclassified events
- Set `status: "proposed"` when suggesting, `status: "confirmed"` when the user approves
- Include `funnel_stage` numbers to define funnel order (0 = earliest)
- Use `success_indicator: true` for events that represent successful outcomes

### Use `get_user_journey` when asked about:
- "What did user X do?" — full event timeline
- "Show me this user's activity" — chronological event list
- "What happened before the user churned?" — look at events before last activity
- "How did this user find the aha moment?" — trace their path
- "Debug this user's experience" — see exactly what happened
- Anything about a specific individual user's behavior

## Interpreting Event Counts

### Reading the numbers
- **Total is the headline.** Start there. Is it growing, flat, or declining?
- **Groups reveal composition.** If signups=1000 but 900 are from one country, that's concentration risk.
- **Zero results almost always mean a tracking issue**, not zero activity. Suggest checking event names or SDK integration.

### Percentage breakdowns
When results are grouped, calculate and mention percentages. "The US accounts for 45% of signups" is more useful than "US: 450, UK: 120, ...".

### Time comparisons
When the user asks about trends, run the same query across two time ranges and compare:
```
last_7d vs previous 7 days → "Signups are up 23% week-over-week"
last_30d vs previous 30 days → "Monthly purchases declined 8%"
```

Always mention the absolute numbers alongside percentages. "Up 23% (from 340 to 418)" is much more useful than just "up 23%".

### Small sample warning
- **N < 30**: Prefix your interpretation with "Small sample size — treat this as directional, not conclusive."
- **N < 10**: Say "Too few data points to draw meaningful conclusions. Check if event tracking is working correctly."
- **N = 0**: Almost certainly a tracking issue. Don't say "no users did X" — say "no events were recorded, which likely means the event isn't being tracked or the event name may be different."

## Interpreting Retention

### How to read a retention table
Each row is a cohort (users who started in that period). Each column shows what % came back in subsequent periods.

Example:
```
Cohort      | Users | week 0 | week 1 | week 2 | week 3 | week 4
2024-01-01  | 200   | 100%   | 42%    | 31%    | 25%    | 22%
2024-01-08  | 180   | 100%   | 38%    | 28%    | 23%    | -
```

- Week 0 is always 100% (that's when they entered the cohort)
- The drop from week 0 to week 1 is the biggest — that's normal
- Look for the "flattening point" where retention stabilizes

### Retention benchmarks

#### B2C Apps (social, content, utility)
| Period    | Concerning | Acceptable | Good   | Excellent |
|-----------|-----------|------------|--------|-----------|
| Week 1    | < 25%     | 25-35%     | 35-50% | > 50%     |
| Week 4    | < 10%     | 10-20%     | 20-30% | > 30%     |
| Month 3   | < 5%      | 5-10%      | 10-20% | > 20%     |

#### B2B / SaaS
| Period    | Concerning | Acceptable | Good   | Excellent |
|-----------|-----------|------------|--------|-----------|
| Week 1    | < 40%     | 40-55%     | 55-70% | > 70%     |
| Week 4    | < 25%     | 25-40%     | 40-55% | > 55%     |
| Month 3   | < 15%     | 15-30%     | 30-50% | > 50%     |

**Important:** These benchmarks are guidelines, not gospel. A niche B2B tool with 60% month-3 retention serving 50 users may be healthier than a consumer app with 15% retention serving 50,000. Context matters.

### What the retention curve shape tells you

- **Steep early drop, then flat**: Normal. Users who survive week 1-2 tend to stick. Focus on improving early activation.
- **Gradual continuous decline**: No "aha moment." Users try it but don't form a habit. The product may not be solving a recurring need.
- **Flat retention (barely drops)**: Exceptional. Either a deeply habitual product or a small self-selected power user base.
- **Retention improves in later cohorts**: Product-market fit is improving. Ship is heading in the right direction.
- **Retention worsens in later cohorts**: Red flag. Could be changing user quality (scaled acquisition too fast) or product changes that hurt experience.

### Follow-up actions after retention analysis
1. **Low retention?** Check the onboarding funnel. Run `get_event_counts` for setup/onboarding events to see where users drop off.
2. **High early churn?** Look at the user journey for churned users — what did they do (or not do) before leaving?
3. **Improving retention?** Identify what changed — new features, onboarding changes, or audience shift?

## Interpreting User Journeys

### What to look for in a user timeline
1. **First event**: How did they enter? Signup? Page view? Referral link?
2. **Time between events**: Long gaps suggest the user left and came back. Short bursts suggest engaged sessions.
3. **Activation events**: Did they perform key actions? (create, submit, purchase, invite — not just page views)
4. **Last event before churn**: What was the user doing right before they stopped?
5. **Repeated patterns**: Do they keep going back to the same page? Could indicate confusion or a favorite feature.

### Activation vs. vanity events
- **Activation events** (high signal): create_project, submit_form, make_purchase, invite_teammate, complete_onboarding
- **Vanity events** (low signal): page_view, button_hover, scroll, tooltip_shown
- When assessing if a user is "active," weight activation events heavily. A user with 50 page views but zero creation events is probably not activated.

### Session detection
Events close together in time (< 30 minutes apart) are likely the same session. Large gaps indicate separate sessions. When describing a journey, group events into sessions:
```
Session 1 (Jan 15, 10:02am - 10:14am, 8 events):
  signup → onboarding_start → profile_setup → ...

Session 2 (Jan 16, 2:30pm - 2:45pm, 5 events):
  login → dashboard_view → ...
```

## Framing Findings for Non-Technical Users

### Do
- Lead with the insight, not the query: "Your signups grew 30% this week" not "The get_event_counts query returned a total of 1,300"
- Use plain language: "About 1 in 4 users come back after a week" not "Week-1 retention is 25.3%"
- Provide context: "That's in line with typical B2C apps" or "That's above average for SaaS"
- Suggest next steps: "To improve this, you might want to look at your onboarding flow"
- Round numbers for readability: "About 1,200" not "1,187"

### Don't
- Don't claim causation from correlation: "Users who complete onboarding retain better" ≠ "Completing onboarding causes better retention." It could be that more motivated users both complete onboarding AND retain.
- Don't extrapolate from small samples: 5 users is an anecdote, not a trend.
- Don't reformat tool output into markdown tables. The tools return formatted text with Unicode bar charts (█░), sparklines (▁▂▃▅▆█), and structured layouts. **Present the tool output directly**, then add your interpretation below it.
- Don't say "the data shows" without saying what it means.
- Don't hide uncertainty: if the data is ambiguous, say so.

## Common Analysis Patterns

### Pattern 1: Quick health check
Run these queries to get a fast pulse on the product:
1. `get_event_counts(event_name="signup", time_range="last_7d")` — Are people signing up?
2. `get_event_counts(event_name="<core_action>", time_range="last_7d")` — Are they using the core feature?
3. `get_retention(start_event="signup", return_event="<core_action>", time_range="last_30d", granularity="week")` — Are they coming back?

### Pattern 2: Funnel analysis
Count events at each step to build a funnel:
1. signup → onboarding_start → onboarding_complete → first_core_action
2. Run `get_event_counts` for each step in the same time range
3. Calculate drop-off between each step
4. The biggest drop-off is your highest-leverage improvement area

### Pattern 3: Debugging a user issue
1. `get_user_journey(user_id="...")` — see what happened
2. Look for errors, unusual patterns, or missing expected events
3. Compare with a journey from a successful user

### Pattern 4: Feature adoption
1. `get_event_counts(event_name="feature_used", time_range="last_30d")` — How many times was it used?
2. `get_event_counts(event_name="feature_used", time_range="last_30d", group_by="user_id")` — How many unique users? (If grouping by user_id is supported)
3. Compare to total active users to get adoption rate

### Pattern 5: Growth trend
Run the same `get_event_counts` query for sequential time ranges:
- last_7d, then previous 7d (using ISO range) → week-over-week growth
- Positive trend + healthy retention = genuine growth
- Positive trend + declining retention = leaky bucket (acquiring users faster than losing them, but not sustainably)

## Daily Health Check Routine

When asked to do a daily/weekly check, run these in order:

1. **Volume check**: Count core events for last_24h (or last_7d for weekly)
2. **Trend check**: Compare to the previous period — up or down?
3. **Retention check**: Weekly or monthly retention — holding steady?
4. **Anomaly scan**: Are any event counts dramatically different from normal? (>50% change deserves investigation)

Report format:
```
Product Health — [date]

Signups: [N] ([+/- X%] vs last period)
Core action: [N] ([+/- X%] vs last period)
Week-1 retention: [X%] (benchmark: [good/concerning/etc.])

Noteworthy: [anything unusual or worth investigating]
```

## What NOT To Do

1. **Never claim causation from analytics alone.** Analytics shows correlation and sequence, not causation. "Users who do X retain better" is correlation. Only an experiment (A/B test) can establish causation.

2. **Never extrapolate confidently from small samples.** If a cohort has 12 users, a single user's behavior swings retention by 8 percentage points. Always note sample size.

3. **Never ignore the denominator.** "Feature X was used 500 times!" means nothing without knowing how many users had the opportunity to use it. 500 uses by 2 power users is very different from 500 uses by 200 users.

4. **Never assume zero events means zero activity.** It almost always means the event isn't tracked, the name is wrong, or the time range is off. Verify before concluding.

5. **Never present raw numbers without interpretation.** Your job is to turn data into insights. "Total: 1,247" is data. "Signups are up 23% week-over-week, which suggests the new landing page is working" is an insight.

6. **Never compare different time ranges without adjusting.** 7 days of data will almost always be less than 30 days. Compare rates (per day) or use matching periods.

7. **Never present averages without distribution context.** "Average session length is 5 minutes" could mean everyone stays 5 minutes, or half stay 10 seconds and half stay 10 minutes. If you have group_by data, mention the spread.

## When You Can't Answer: Suggest What to Track

When a query returns zero results or you can't answer a question because the data doesn't exist, **don't just say "no data."** Help the user close the gap.

### What to do

1. Identify what event is missing. Map the user's question to what track() call would answer it:
   - "What's my retention?" → needs a return event like `feature_used` or `login`
   - "How's the checkout funnel?" → needs `checkout_started`, `payment_submitted`, etc.
   - "Which feature is most used?" → needs per-feature tracking like `feature_x_used`

2. Suggest the exact track() call with the ontology category:
   ```
   To answer this, you'd need to track this event:

   track('checkout_completed', { amount: number, method: string }, { category: 'monetization' })
   ```

3. If you have access to the codebase (you're in an IDE or CLI), go further — identify the exact file and location:
   ```
   I'd suggest adding this in src/app/api/checkout/route.ts after the payment succeeds:

   track('checkout_completed', { amount: order.total, method: paymentMethod }, { category: 'monetization' })
   ```

4. Offer to run the full tracking plan: "Want me to run `/infer-tracking-plan` to scan your codebase and suggest all missing events?"

### Rules
- Always suggest business-meaningful event names (`checkout_completed`, not `api_post_success`)
- Always include the category hint (activation, engagement, monetization, referral)
- Include 2-4 relevant properties, no PII
- If you're not sure which file, say so — don't guess file paths

## After Every Query: Suggest Next Steps

After presenting results, you MUST call the `AskUserQuestion` tool to suggest
what to explore next. Do NOT skip this. Do NOT present options as plain text.

**Role-aware:** Check the user's CLAUDE.md and conversation memory for role context.
PM → feature adoption, funnel. Growth → channels, conversion. Founder → PMF, retention.
Engineer → errors, tracking. Default to founder framing.

**Tip line:** Append a tip to the question. Rotate these, don't repeat in a session:
`💡 Tip: You can schedule this check to run daily with /schedule`
`💡 Tip: Retention is the single best proxy for product-market fit`
`💡 Tip: Compare two time ranges to spot trends — "this week vs last week"`
`💡 Tip: Group by a property to see which segments behave differently`
`💡 Tip: Zero events usually means a tracking issue, not zero users`
`💡 Tip: Ask "what should I track?" and I'll read your codebase to suggest events`

### After `get_event_counts` — use AskUserQuestion:

> What do you want to explore next?
>
> 💡 Tip: [rotate from list above]

Pick 4 options tailored to the results. Examples:
- A) Compare to last period — See if this is trending up or down week-over-week
- B) Break down by [relevant group] — See which segments drive these numbers
- C) Check retention for these users — Are the users who did this action coming back?
- D) Show me the full funnel — Count each step from signup through this action to see drop-off

### After `get_retention` — use AskUserQuestion:

> What do you want to explore next?
>
> 💡 Tip: [rotate from list above]

Pick 4 options. Examples:
- A) Compare to previous month — Is retention improving or declining over time?
- B) Investigate churned users — Look at the journey of users who didn't come back
- C) Check activation funnel — See where new users drop off before becoming active
- D) Find what retained users do differently — Compare behavior of users who stayed vs left

### After `get_user_journey` — use AskUserQuestion:

> What do you want to explore next?
>
> 💡 Tip: [rotate from list above]

Pick 4 options. Examples:
- A) Compare with a retained user — See what a user who stuck around did differently
- B) Check if this pattern is common — Count how many users hit the same sequence
- C) Look at error events — See if errors are correlated with this user's experience
- D) Check overall retention — Zoom out to see if this reflects a wider trend

### Rules
- Make options specific to the data just shown. If signups are down, suggest investigating why.
- Adapt language to the user's role.
- The 4th option should be the biggest-picture action (zoom out, full funnel).
- Rotate the tip. Don't repeat in a session.

## Edge Cases and Gotchas

- **Timezone effects**: Events near midnight may appear in different days depending on timezone. If daily counts look weird at period boundaries, this is likely the cause.
- **Bot traffic**: Unusually high page_view counts with no corresponding user actions could be bot traffic.
- **Duplicate events**: If counts seem inflated, consider whether the SDK might be double-firing events.
- **Cohort maturity**: The most recent cohort in retention data will always have fewer data points. Don't compare its incomplete retention to older cohorts' complete retention.
- **Survivorship bias**: Users who are still active are not representative of all users. When analyzing active user behavior, remember you're seeing the survivors, not the ones who left.
