---
name: infer-tracking-plan
description: Use when the user wants to know what events to track, needs a tracking plan, or after installing the Infer SDK. Triggers on "what should I track", "suggest events", "tracking plan", "add tracking", or when the setup wizard reaches the custom tracking step.
---

# Infer Tracking Plan — Codebase-Driven Event Discovery

You are building a tracking plan by reading the actual codebase, not guessing.
The goal: propose specific track() calls at specific file:line locations,
named by what they mean to the business, and get user approval before touching code.

## The Process

### Phase 1: Understand the Product (read before proposing)

Before suggesting a single event, you need to understand what this app does.

**Read these files in order:**
1. `package.json` — what framework, what dependencies, what's the app about
2. `README.md` or any docs — product description, user flows
3. `CLAUDE.md` — project context, if it exists
4. Root layout/entry point — app structure, navigation, auth
5. Route/page structure — `ls src/app/` or `ls src/pages/` or equivalent

**Answer these questions before proceeding:**
- What does this app do? (1 sentence)
- Who is the user? (1 sentence)
- What is the core value action? (the ONE thing users come here to do)
- What is the activation moment? (when a new user first gets value)
- What would make a user come back tomorrow?

If you can't answer these from the code, ask the user.

### Phase 2: Map the User Journey

Read the codebase to map the complete user journey. For each stage,
identify the files where that stage happens.

```
ENTRY → ACTIVATION → CORE ACTION → ENGAGEMENT → RETENTION TRIGGER
```

**Entry stage:** How do users arrive?
- Read auth/signup flows, landing routes, onboarding components
- Look for: registration forms, OAuth handlers, invite acceptance

**Activation stage:** When does a user first get value?
- Read the onboarding flow, first-run experience
- Look for: setup wizards, profile creation, first content creation

**Core action stage:** What's the main thing users do?
- Read the primary feature components and API routes
- Look for: CRUD operations, search, generation, submission

**Engagement stage:** What keeps users active?
- Read secondary features, social features, notification handlers
- Look for: sharing, collaboration, export, integrations

**Retention triggers:** What brings users back?
- Read notification systems, scheduled jobs, email triggers
- Look for: reminders, digests, new content alerts

### Phase 3: Deep Dive — Read the Actual Code

For each stage identified above, read the specific files.

**What to look for:**

In **API routes / server actions:**
- Successful mutations (POST/PUT/DELETE handlers that return 200)
- These are the most reliable tracking points — they represent completed actions

In **form components:**
- `onSubmit` handlers — track form completion, not form views
- Distinguish between form start and form completion if the form is multi-step

In **AI/LLM tool definitions:**
- Tool call handlers — each tool invocation is a feature usage event
- Tool results — successful completions vs errors

In **State changes:**
- Zustand/Redux actions that represent user decisions
- React Query mutations that represent completed operations

In **Navigation:**
- Key page transitions that represent intent (pricing page = considering upgrade)
- Auto-tracked by the SDK, but some need custom names

**What NOT to track:**
- Every button click (auto-tracked already)
- Every page view (auto-tracked already)
- Internal state transitions users don't initiate
- Debug/dev-only actions
- Reads/views without action (passive browsing is covered by auto-track)

### Phase 4: Build the Tracking Plan

For each proposed event, document:

| Field | Description |
|-------|-------------|
| Event name | snake_case, business-meaningful (not technical) |
| File | Exact file path |
| Location | Function/handler name or line context |
| Category | entry / activation / core / engagement / retention |
| Properties | What data to include (key: type) |
| Why | Why this event matters for analytics |

**Event naming rules:**
- Use `noun_verbed` format: `resume_uploaded`, `job_searched`, `letter_generated`
- NOT `click_upload_button` or `api_post_resume` (too technical)
- NOT `event_1` or `track_signup` (too generic)
- The name should make sense in the sentence: "How many users [event_name] last week?"

**Property rules:**
- Include identifying properties: which item, which type, which category
- Do NOT include PII: no emails, names, phone numbers
- Do NOT include high-cardinality strings: no free text, no full URLs
- Do include enums: plan type, role, category, status

### Phase 5: Present for Approval

Present the tracking plan as a numbered table:

```
## Tracking Plan: [App Name]

**Product:** [1-sentence description]
**Core value action:** [what users come here to do]
**Activation moment:** [when new users first get value]

### Proposed Events

| # | Event | Category | File | Properties | Why |
|---|-------|----------|------|------------|-----|
| 1 | signup_completed | entry | src/app/api/auth/route.ts | method: string | Funnel start, measures acquisition |
| 2 | profile_created | activation | src/lib/actions/profile.ts | has_photo: bool | Activation milestone |
| 3 | project_created | core | src/lib/actions/project.ts | type: string | Core value delivery |
| ... | ... | ... | ... | ... | ... |

### Already auto-tracked (no code needed)
- page_view (all navigation)
- session_start (new sessions)
- click (interactive elements)
- form_submit (form submissions)
- error (JS exceptions)

### Funnel this enables
signup → [activation event] → [core action] → [engagement] → return visit
With these events, you can answer:
- "What's my signup-to-activation conversion?"
- "Which users are most engaged?"
- "Where do users drop off?"

? Approve all? Or select by number (e.g., "1,2,3,5" or "all except 4")
```

### Phase 6: Implement Approved Events

For each approved event, add the track() call.

**Ask before each file change:** "Adding [event_name] to [file]. OK?"

**Implementation patterns:**

For API route handlers (server-side, need client-side tracking):
- Don't track on the server. Track on the client after a successful API response.
- Find the component that calls this API and add track() in the success handler.

For form submissions:
```typescript
import { track } from "@/lib/analytics";

// In the onSubmit handler, AFTER successful submission:
track("form_completed", { form: "signup", method: "email" });
```

For AI tool calls:
```typescript
import { track } from "@/lib/analytics";

// After the tool returns a successful result:
track("tool_used", { tool: "search_jobs", results_count: results.length });
```

For state changes:
```typescript
import { track } from "@/lib/analytics";

// After the state mutation succeeds:
track("status_changed", { from: "draft", to: "published" });
```

**Add identify() at the auth boundary:**
Find where the app resolves the current user (after login, after OAuth callback,
after session restore). Add:
```typescript
identify(user.id, { plan: user.plan, role: user.role });
```

**Add reset() at logout:**
```typescript
import { reset } from "@/lib/analytics";
// In the logout handler:
reset();
```

### Phase 7: Summary

After implementing, show what was added:

```
## Tracking Plan Implemented

Added [N] events across [M] files:
✓ signup_completed (src/components/auth/signup-form.tsx)
✓ resume_uploaded (src/components/chat/chat-input.tsx)
✓ job_searched (src/lib/ai/tools.ts)
✗ job_applied (skipped by user)

Funnel: signup → resume_uploaded → job_searched → job_matched → application_tracked

To see your data:
- "What events are being tracked?" → get_top_events
- "What's my signup-to-search conversion?" → get_event_counts for each step
- "Show me retention" → get_retention
```

## Important Rules

1. **Read the code first.** Never propose events from guessing. Every suggestion
   must reference a specific file and function you actually read.

2. **Business names, not technical names.** `resume_uploaded` not `post_api_upload`.
   The name should be readable by a non-engineer.

3. **Approval before implementation.** Present the full plan. Get explicit approval.
   Never add track() calls without permission.

4. **Don't over-track.** 5-10 custom events is plenty for an MVP. More is noise.
   Focus on the funnel: entry → activation → core → engagement.

5. **Properties are minimal.** 2-4 properties per event max. No PII. No free text.
   Enums and IDs only.

6. **Auto-track handles the basics.** Don't manually track page views or clicks.
   That's what autoTrack: true does. Custom events are for business-meaningful actions.
