---
name: infer-setup
description: Use when setting up Infer analytics in a new project, integrating the SDK, or configuring the MCP server. Triggers on "set up analytics", "add tracking", "integrate infer", "install infer", or when a project has no analytics and the user wants to add it.
---

# Infer Setup — The Wizard

## Update Check (non-blocking)

Run this at the start. Checks for newer Infer versions using a 6-hour cache:

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
Do NOT block the wizard. Continue normally.

This is the full setup wizard. It detects your current state and walks you through
everything step by step, asking before each change.

## How Infer Works

- **Write key** (`pk_write_...`): Public. Embedded in app JS. Can ONLY send events.
- **Read key** (`pk_read_...`): Secret. In `~/.infer/config.json`. Used by this MCP server to query data.
- **SDK** (`@inferevents/sdk`): Installed in the user's app. Tracks events.
- **MCP server** (`@inferevents/mcp`): Separate package. Connects to Claude Code as an MCP server. Queries the API.
- **Skills** (`@inferevents/skills`): This package. Teaches agents how to use Infer.

## Entry Points

There are two ways users arrive here:

### Entry A: User says "set up Infer" or invokes /infer-setup
Start from Step 1 below.

### Entry B: User pasted a setup prompt from infer.events/signup
The prompt contains credentials inline. Extract them and skip to Step 3.
Look for: `projectId`, `writeKey`, `readKey`, `endpoint` in the pasted text.

## The Wizard

### Step 1: Check MCP server

First, check if the Infer MCP server is connected. Try calling any Infer tool
(e.g., `get_top_events`). If it works, the MCP server is already set up.

If the MCP server is NOT connected:
Tell the user: "The Infer MCP server isn't connected yet. Let me set it up."

Add to the user's Claude Code MCP config (settings.json or claude_desktop_config.json):
```json
{
  "mcpServers": {
    "infer": {
      "command": "npx",
      "args": ["@inferevents/mcp"]
    }
  }
}
```

Ask: "I need to add the Infer MCP server to your Claude Code config. OK?"
After adding, the user may need to restart Claude Code for MCP to connect.

### Step 2: Check existing account

Read `~/.infer/config.json`. Three possible states:

**State A: Config exists with projects**
The user already has an Infer account. Check if the current directory already has
Infer configured (check `package.json` for `@inferevents/sdk` in dependencies).

If SDK is already installed:
→ Say: "This project already has Infer configured (project: [active project name])."
→ Ask: "Want to keep this config, switch to a different project, or create a new one?"
  A) Keep current config → Jump to Step 4 (verify)
  B) Switch project → Call `switch_project` MCP tool to list and select
  C) Create new project → Go to Step 3

If SDK is NOT installed:
→ Say: "Found your Infer account ([N] projects). Let's set up this codebase."
→ Ask: "Use an existing project or create a new one?"
  A) Use existing → Call `switch_project` to select, then jump to Step 5 (detect project)
  B) Create new → Go to Step 3

**State B: Config exists but empty/invalid**
→ Say: "Found Infer config but it's invalid. Let's reconnect."
→ Go to Step 3.

**State C: Config exists with session but no projects (new signup)**
The user just signed up. The config has `{"endpoint":"...","session":"abc123"}`
but no projects yet. This is the normal state after pasting the signup command.
→ Say: "Found your Infer session. Let me fetch your project."
→ Go to Step 2b (fetch keys).

**State D: No config file**
→ Say: "No Infer account found. Let's get you set up."
→ Ask: "Do you have an Infer account?"
  A) Yes → Open https://infer.events/signup, sign in, paste the setup command here.
  B) No → Open https://infer.events/signup, create an account, paste the setup command.

When they paste it, the config is saved to `~/.infer/config.json` with a session
token. Proceed to Step 2b.

### Step 2b: Fetch project keys via session

If the config has a `session` field but no project keys, fetch them from the API.

1. Call `GET /v1/auth/me?session=SESSION` to get the user's project list
2. If user has one project:
   - Call `GET /v1/auth/project-keys?session=SESSION&project_id=PROJECT_ID`
   - Save the returned write_key and read_key to `~/.infer/config.json`
   - Jump to Step 4 (verify MCP)
3. If user has multiple projects:
   - Ask which project to use for this codebase
   - Fetch keys for the chosen project
   - Save to config as active project
4. If user has no projects:
   - Go to Step 3 (create new project)

This step is the magic: the user never sees or handles API keys. The session
token authenticates, the API returns the keys, and they're saved automatically.

### Step 3: Create new project (CLI-first)

This runs entirely in the CLI. No website redirect needed.

Ask: "What should we call this project?" (suggest: the current directory name)

Then call the `create_project` MCP tool with the project name. This:
1. Reads the saved session token from `~/.infer/config.json`
2. Calls the Infer API to create a new project
3. Generates write + read API keys
4. Saves the new project to `~/.infer/config.json` as the active project

If the tool returns "No session found": the user needs to sign in first.
Say: "You need to sign in once. Opening infer.events/signup..."
After they complete signup and paste the setup command, extract the session token
and retry the project creation.

After project creation succeeds:
→ Say: "Project '[name]' created! Write key and config saved."
→ Jump to Step 5 (detect project).

### Step 4: Verify MCP connection

The MCP server is already running if the user is reading this skill. But check
if the config is complete:

1. Verify `~/.infer/config.json` has `apiKey`, `endpoint`
2. Test connectivity: call `get_top_events(time_range="last_24h")` as a ping
3. If it works: "MCP server connected and working."
4. If it fails: troubleshoot (wrong endpoint, invalid key, network issue)

### Step 5: Detect project

Read `package.json` in the current working directory.

**Framework detection:**

| Signal | Framework | Entry point |
|--------|-----------|-------------|
| `"next"` in deps + `src/app/` exists | Next.js App Router | `src/app/layout.tsx` |
| `"next"` in deps + `src/pages/` exists | Next.js Pages Router | `src/pages/_app.tsx` |
| `"react-scripts"` in deps | Create React App | `src/index.tsx` |
| `"vite"` in devDeps + `"react"` in deps | Vite + React | `src/main.tsx` |
| `"expo"` in deps | Expo / React Native | `App.tsx` or `app/_layout.tsx` |
| `"nuxt"` in deps | Nuxt.js | `app.vue` or `plugins/` |
| `"@sveltejs/kit"` in devDeps | SvelteKit | `src/routes/+layout.svelte` |
| None detected | Unknown | Ask user |

Tell the user what you detected:
"Detected **[framework]** project. Entry point: `[path]`."

If `@inferevents/sdk` is already in dependencies:
"SDK already installed. Checking integration..."
→ Skip to Step 6.

### Step 6: Install SDK

Ask: "Install @inferevents/sdk? (This adds the tracking library to your project)"

If yes:
```bash
npm install @inferevents/sdk
```

If SDK is not published to npm yet, install from local path or suggest:
```bash
npm install @inferevents/sdk@latest
```

### Step 7: Integrate SDK

Ask: "Add tracking to [detected entry point]? I'll create the analytics module and
wire it into your app. I'll ask before changing each file."

**Then create files based on detected framework:**

#### Next.js App Router

**Gate 1:** "Create `src/lib/analytics.ts`?"
```typescript
import { init, track, identify, page, reset } from "@inferevents/sdk";

export function initAnalytics() {
  if (typeof window === "undefined") return;
  init({
    projectId: "[WRITE_KEY]",
    endpoint: "[ENDPOINT]",
    autoTrack: true,
    debug: process.env.NODE_ENV === "development",
  });
}

export { track, identify, page, reset };
```

**Gate 2:** "Create `src/components/analytics-provider.tsx`?"
```typescript
"use client";
import { useEffect } from "react";
import { initAnalytics } from "@/lib/analytics";

export function AnalyticsProvider() {
  useEffect(() => { initAnalytics(); }, []);
  return null;
}
```

**Gate 3:** "Add `<AnalyticsProvider />` to `src/app/layout.tsx`?"
- Import the component
- Add inside `<body>` tag

#### Next.js Pages Router

**Gate 1:** Create `src/lib/analytics.ts` (same as above)
**Gate 2:** Add to `src/pages/_app.tsx`:
```typescript
import { useEffect } from "react";
import { initAnalytics } from "@/lib/analytics";

// Inside the component:
useEffect(() => { initAnalytics(); }, []);
```

#### Vite / CRA / Plain React

**Gate 1:** Create `src/lib/analytics.ts` (same as above)
**Gate 2:** Add to `src/main.tsx`:
```typescript
import { initAnalytics } from "./lib/analytics";
initAnalytics();
```

### Step 7b: Check for Content Security Policy

After integrating the SDK, check if the project has a CSP that would block
requests to `api.infer.events`. Search for Content-Security-Policy in:
- `next.config.ts` / `next.config.js` (Next.js headers)
- `middleware.ts` (Next.js middleware)
- `<meta http-equiv="Content-Security-Policy">` in HTML
- `.htaccess`, `nginx.conf`, `vercel.json` headers

If a CSP with `connect-src` is found and does NOT include `api.infer.events`:
→ Say: "Your site has a Content Security Policy. I need to add api.infer.events
   to connect-src so the SDK can send events."
→ Add `https://api.infer.events` to the existing `connect-src` directive.
→ Ask before making the change.

If no CSP is found, skip this step silently.

### Step 8: Build tracking plan

Ask: "Want me to analyze your codebase and suggest what events to track?"

If yes, read the `infer://tracking-plan` resource and follow it completely.
It will:
1. Deep dive into the codebase to understand the product
2. Map the user journey (entry → activation → core action → engagement)
3. Propose specific events at specific file:line locations
4. Present a table for approval
5. Implement only the approved events

This is the most valuable step — it's doing the PM's job of deciding what to measure,
based on the actual code, not guessing.

### Step 9: Verify

Say: "Let me verify everything is working."

1. Ask the user to open their app in the browser
2. Wait 15 seconds for the batch to flush
3. Run `get_top_events(time_range="last_24h")`
4. If events found: "✓ Events flowing. [N] events received."
5. If no events: troubleshoot (check write key, check console errors, check app is running)

### Step 10: Suggest automation

Say:

> Everything is set up! Here are some things you can ask me:
>
> - "What's happening in my app?" — quick overview
> - "What's my retention?" — are users coming back?
> - "Show me top events" — what users do most
>
> For automated monitoring:
> - `/schedule daily "Run Infer analytics health check"`
> - `/loop 24h "Check analytics and flag anything unusual"`

## From the Website: Install Flow

When a user signs up on the website, the signup page shows two steps:

**Step 1: Install skills**
```
npx skills add infer-events/skills
```

**Step 2: Paste into Claude Code**
```
Save this config to ~/.infer/config.json:
{"apiKey":"[READ_KEY]","endpoint":"[ENDPOINT]","projectId":"[PROJECT_ID]"}.
Then run /infer-setup to configure everything.
```

The setup wizard handles the rest: MCP server install, SDK install, framework
detection, tracking integration, verification.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| "No events found" after setup | Write key wrong or app not running | Check write key in analytics.ts, check browser console |
| 401 from API | Invalid or wrong key type | Write key for SDK, read key for MCP |
| SDK not initializing | SSR calling init() | Check `typeof window !== "undefined"` guard |
| MCP server can't connect | Wrong endpoint or read key | Check ~/.infer/config.json |
| Auto-track not firing | autoTrack not set | Set `autoTrack: true` in init() |
| Console: "Refused to connect" / CSP error | Content Security Policy blocks api.infer.events | Add `https://api.infer.events` to `connect-src` in your CSP header |
| SDK retries then stops after 5 attempts | CSP or network blocking the endpoint | Check console for the CSP help message, add domain to connect-src |
