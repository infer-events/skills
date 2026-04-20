---
name: infer-setup
description: Use when setting up Infer observability in a project, routing an LLM client through the gateway, integrating the MCP server, or migrating from the legacy @inferevents/sdk web-analytics SDK. Triggers on "set up Infer", "add Infer", "install Infer", /infer-setup.
allowed-tools:
  - AskUserQuestion
---

# Infer Setup — The Wizard (gateway-era)

Infer is an AI-agent observability gateway. You change one `baseURL` in your
LLM client — that's it. Every OpenAI / Anthropic / Ollama Cloud call routes
through `gateway.infer.events`, captures an OpenTelemetry GenAI span, and
lands in Postgres. The MCP server reads those spans back as insights,
investigations, and proactive briefings.

This skill walks you through five things:

1. Account + project setup (signup or use existing)
2. Provider table — which providers Infer supports today (and which don't work)
3. Detect your LLM client in this codebase
4. 3-line baseURL change (with opt-in auto-edit)
5. Live-verify the first span lands + MCP config + restart

If you installed the old `@inferevents/sdk` (web-analytics era), the wizard
also walks you through removing it.

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
  _MCP_LATEST=$(npm view @inferevents/mcp version 2>/dev/null || echo "unknown")
  mkdir -p ~/.infer
  echo "{\"checked\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"mcp_latest\":\"$_MCP_LATEST\"}" > "$_INFER_CACHE"
  echo "INFER_MCP_LATEST=$_MCP_LATEST"
else
  cat "$_INFER_CACHE"
  echo "INFER_CHECK=cached"
fi
```

If a newer MCP version is available, append at the END of your response:
`Infer update available — run /infer-upgrade to get the latest.`
Do NOT block the wizard. Continue normally.

## How Infer Works

- **Gateway** (`gateway.infer.events`): Cloudflare Worker that proxies your LLM
  calls to the real provider and captures a span per call.
- **Project ID** (`proj_...`): Public identifier in the baseURL path. Same
  privacy class as your Cloudflare account_id.
- **Read key** (`pk_read_...`): Secret. Stored in `~/.infer/config.json`. Used
  by the MCP server to query your spans.
- **MCP server** (`@inferevents/mcp`): Connects to Claude Code (or any MCP
  client). Runs locally via `npx` (stdio) or remotely at `mcp.infer.events/mcp`.
- **Skills** (`infer-events/skills`): This package. Teaches your agent how
  to use Infer.

## Supported Providers (v1)

| Provider | URL pattern | v1 | Notes |
|---|---|---|---|
| OpenAI | `gateway.infer.events/v1/{project_id}/openai/v1` | ✓ | GPT-4o, 4o-mini, o1, o3-mini, etc. |
| Anthropic | `gateway.infer.events/v1/{project_id}/anthropic` | ✓ | claude-sonnet-4-6, claude-opus-4-7, claude-haiku-4-5 |
| Ollama Cloud | `gateway.infer.events/v1/{project_id}/ollama/v1` | ✓ | Cloud only (api.ollama.com); self-hosted Ollama is blocked — see troubleshooting |
| Gemini | — | — | v1.1 roadmap |
| Cohere / Mistral | — | — | v1.1+ roadmap |
| Fireworks / Together / Groq / other OpenAI-compatible | — | — | **Not supported.** Each needs its own gateway segment. Open an issue for the provider you need. |

**⚠️ Footgun:** If your client uses an OpenAI-compatible `baseURL` pointed at
a non-OpenAI provider (e.g. `https://api.fireworks.ai/inference/v1`), Infer
v1 cannot route it — the `/openai/` path segment is hardcoded to
`api.openai.com`. Each additional OpenAI-wire-compatible provider needs its
own gateway segment. Open an issue requesting the one you need, or use
OpenAI / Anthropic / Ollama Cloud for now.

## Setup Flow

```
1. Account setup (signup or existing)
2. Project setup (pick or create; fetch keys)
3. Migration check (if @inferevents/sdk present, remove it)
4. Detect LLM client + propose 3-line baseURL diff
5. Apply (auto-edit) or show (manual-apply) + live-verify
6. Configure MCP + restart
```

MCP is last because it needs a Claude Code restart. When the user returns,
everything works — spans flow through the gateway, MCP reads them back.

## Entry Points

There are two ways users arrive here:

### Entry A: User says "set up Infer" or invokes /infer-setup
Start from Step 1 below.

### Entry B: User pasted a setup prompt from infer.events/signup
The prompt contains credentials inline. Extract them and skip to Step 4.
Look for: `project_id`, `read_key`, `endpoint` in the pasted text.

## The Wizard

### Step 1: Account setup

Read `~/.infer/config.json`. Four possible states:

**State A: Config exists with projects**
Existing Infer user.
→ Say: "Found your Infer account ([N] projects). Let's set up this codebase."
→ Ask via `AskUserQuestion`: "Use an existing project or create a new one?"
  - A) Use existing → list projects, let user pick, then Step 3
  - B) Create new → Step 2

**State B: Config exists but empty/invalid**
→ Say: "Found Infer config but it's invalid. Let's reconnect."
→ Go to signup (Step 1c).

**State C: Config exists with session but no projects (new signup)**
→ Say: "Found your Infer session. Let me fetch your project."
→ Go to Step 1b (fetch keys).

**State D: No config file**
→ Say: "No Infer account found. Let's get you set up."
→ Go to signup (Step 1c).

### Step 1b: Fetch project keys via session

If the config has a `session` field but no project keys, call the API:
1. `GET /v1/auth/me?session=SESSION` → user's project list
2. If exactly one project:
   - `GET /v1/auth/project-keys?session=SESSION&project_id=PROJECT_ID`
   - Save `{projectId, readKey, endpoint}` to `~/.infer/config.json`
   - Write `.infer.json` to project root: `{"project": "<slug>"}`
   - Add `.infer.json` to `.gitignore`
   - Jump to Step 3
3. If multiple projects: ask which one, then fetch keys for that one.
4. If no projects: go to Step 2.

### Step 1c: Signup

→ Ask via `AskUserQuestion`: "Do you have an Infer account?"
  - A) Yes → open `https://infer.events/signup`, sign in, paste the setup command here
  - B) No → open `https://infer.events/signup`, create account, paste setup command

When pasted, the config is saved to `~/.infer/config.json` with a session
token. Proceed to Step 1b.

### Step 2: Create new project

Runs entirely in the CLI. Ask: "What should we call this project?" (suggest
the current directory name).

Call the `create_project` MCP tool with the project name:
- Reads the session token from `~/.infer/config.json`
- Creates the project via the API
- Receives `project_id` + `pk_read_*` key in response
- Saves to `~/.infer/config.json` as the active project

After creation:
- Write `.infer.json` to project root: `{"project": "<slug>"}`
- Add `.infer.json` to `.gitignore`
- Say: "Project '[name]' created. This directory is linked to it."
- Jump to Step 3.

### Step 3: Check for legacy @inferevents/sdk

Read `package.json` in the current working directory. If `@inferevents/sdk`
appears in `dependencies` or `devDependencies`:

→ Say: "You have the legacy web-analytics Infer SDK. Infer pivoted to an
  LLM observability gateway in 2026-04 — the SDK is being deprecated
  (parent spec §8 Phase 8). Let's remove it before setting up the gateway
  mode."

→ Ask via `AskUserQuestion`:
  - A) Remove automatically — I'll `npm uninstall @inferevents/sdk` and remove `init()`/`track()`/`identify()`/`reset()` calls
  - B) Show me the removal steps — I'll do it manually
  - C) Keep it (not recommended; SDK won't be maintained past 2026-07)

On (A):
```bash
npm uninstall @inferevents/sdk
```
Then grep for legacy calls and remove file-by-file, asking approval per file:
```bash
grep -rnE "from ['\"]@inferevents/sdk['\"]|init\(\s*\{\s*projectId|\bidentify\(|\btrack\(|\breset\(\s*\)" src/
```
For each file:
→ Say: "Removing init()/track()/identify()/reset() calls in `src/lib/analytics.ts`. OK?"
Remove the imports, the init() call, and any helper exports (`track`, `identify`, etc.). If the only thing the file does is wire analytics, offer to delete the whole file.

Also remove:
- `src/components/analytics-provider.tsx` (or equivalent) — the wrapper
- `<AnalyticsProvider />` mount in `src/app/layout.tsx` (or equivalent entry)

On (B): emit a concise manual removal guide and pause for "done, continue".

### Step 4: Detect LLM client

Read `package.json`. Match against these client detectors (v1 scope per
spec Decision #6):

| Package | Provider | Detect pattern |
|---|---|---|
| `openai` | OpenAI | `new OpenAI({...})` or `import ... from 'openai'` |
| `@anthropic-ai/sdk` | Anthropic | `new Anthropic({...})` |
| `ai` + `@ai-sdk/openai` | OpenAI (Vercel AI SDK) | `createOpenAI({...})` |
| `ai` + `@ai-sdk/anthropic` | Anthropic (Vercel AI SDK) | `createAnthropic({...})` |
| `ai` + `@ai-sdk/openai-compatible` | OpenAI-compatible (Ollama, etc.) | `createOpenAICompatible({...})` |
| `ollama` | Ollama | `new Ollama({...})` |

For each matching file, locate the client constructor. Build the proposed
3-line diff using the detected pattern and the user's actual `project_id`
+ the correct gateway URL for the provider.

**If no client is detected** (langchain, llamaindex, raw fetch, Python, etc.),
say: "Couldn't auto-detect a supported LLM client. Here's the generic
pattern — apply it to whichever HTTP client your code uses:

```
OpenAI-compat:  baseURL = "https://gateway.infer.events/v1/<PROJECT_ID>/openai/v1"
Anthropic:      baseURL = "https://gateway.infer.events/v1/<PROJECT_ID>/anthropic"
Ollama Cloud:   baseURL = "https://gateway.infer.events/v1/<PROJECT_ID>/ollama/v1"
```

Open an issue if your client library should be auto-detected."

### Step 5: Propose the 3-line diff (and apply, or don't)

For each detected client, show the diff inline. Examples:

**OpenAI (`openai` package):**
```ts
-const openai = new OpenAI({
-  apiKey: process.env.OPENAI_API_KEY,
-});
+const openai = new OpenAI({
+  apiKey: process.env.OPENAI_API_KEY,
+  baseURL: "https://gateway.infer.events/v1/<PROJECT_ID>/openai/v1",
+});
```

**Anthropic (`@anthropic-ai/sdk`):**
```ts
-const anthropic = new Anthropic({
-  apiKey: process.env.ANTHROPIC_API_KEY,
-});
+const anthropic = new Anthropic({
+  apiKey: process.env.ANTHROPIC_API_KEY,
+  baseURL: "https://gateway.infer.events/v1/<PROJECT_ID>/anthropic",
+});
```

**Vercel AI SDK — OpenAI:**
```ts
-import { createOpenAI } from "@ai-sdk/openai";
-export const openai = createOpenAI({
-  apiKey: process.env.OPENAI_API_KEY,
-});
+import { createOpenAI } from "@ai-sdk/openai";
+export const openai = createOpenAI({
+  apiKey: process.env.OPENAI_API_KEY,
+  baseURL: "https://gateway.infer.events/v1/<PROJECT_ID>/openai/v1",
+});
```

**Vercel AI SDK — Anthropic:**
```ts
-export const anthropic = createAnthropic({
-  apiKey: process.env.ANTHROPIC_API_KEY,
-});
+export const anthropic = createAnthropic({
+  apiKey: process.env.ANTHROPIC_API_KEY,
+  baseURL: "https://gateway.infer.events/v1/<PROJECT_ID>/anthropic",
+});
```

**Vercel AI SDK — openai-compatible (Ollama):**
```ts
-export const ollama = createOpenAICompatible({
-  name: "ollama-cloud",
-  apiKey: process.env.OLLAMA_API_KEY,
-  baseURL: "https://ollama.com/v1",
-});
+export const ollama = createOpenAICompatible({
+  name: "ollama-cloud",
+  apiKey: process.env.OLLAMA_API_KEY,
+  baseURL: "https://gateway.infer.events/v1/<PROJECT_ID>/ollama/v1",
+});
```

Substitute the real `PROJECT_ID` (from `~/.infer/config.json`) in each URL.

Ask via `AskUserQuestion`:
- A) Apply automatically — I'll edit the file
- B) Just show me the diff — I'll apply it

On (A): edit each file with the diff. Confirm per file:
→ Say: "Editing `src/lib/openai.ts`. OK?"
Apply and move on.

On (B): leave the diffs on screen; wait for "done, continue".

### Step 6: Live-verify (the first span lands)

After baseURL change is applied:
→ Say: "Make one test LLM call from your app (or run `npm run dev` and trigger
  a feature that uses the client). I'll check that the span landed within
  ~2 seconds."

Then call the MCP tool:
```
list_spans(limit=1, time_window=5m)
```

If at least one span is returned:
→ Say: "Your first span is in Infer — model=X, duration=Yms, status=Z. You're
  live."

If `list_spans` returns zero:
→ Say: "No span landed yet. Possible reasons:
  - You haven't made an LLM call since the baseURL change
  - Your dev server cached the old config — restart it
  - The client's baseURL change didn't save — re-check the file
  
  Make an LLM call and ask me to re-check."

### Step 7: Configure MCP server

Check `.mcp.json` in the project root or the user's Claude Code settings.

If NOT configured, ask:
→ "Add the Infer MCP server to your project config? This lets your agent
   query your spans."

Add to `.mcp.json`:
```json
{
  "mcpServers": {
    "infer": {
      "command": "npx",
      "args": ["--yes", "@inferevents/mcp@latest"]
    }
  }
}
```

If `.mcp.json` already exists, merge the `infer` entry in.

### Step 8: Restart + closing summary

Present a summary:
```
Setup complete:
- ~/.infer/config.json — account credentials (read key, endpoint, project_id)
- .infer.json — links this directory to your project
- .mcp.json — Infer MCP server
- <file>.ts — baseURL pointed at gateway.infer.events
- First span confirmed landed

Next: restart Claude Code to connect the MCP server. After restart, ask me
anything about your LLM observability data.
```

Call `AskUserQuestion`:

> Restart Claude Code to load the MCP server. Once connected, try these next.
>
> 💡 Tip: After a baseURL change, the first span lands within ~2s of your first LLM call

Options:
- A) Verify spans are flowing — I'll run `list_spans` after you restart
- B) See my first insights — run `/infer-insights` to get a briefing
- C) Set up daily monitoring — `/loop` or `/schedule` for recurring checks
- D) I'll make a test call first — explore on my own

Rotating tips (pick one that hasn't appeared in this session):
- `💡 Tip: After a baseURL change, the first span lands within ~2s of your first LLM call`
- `💡 Tip: Use x-infer-redact: true to drop message bodies for sensitive data`
- `💡 Tip: x-infer-session-id + x-infer-user-id headers unlock per-session / per-user analysis in get_latency_stats + get_cost_stats`

## From the Website: Install Flow

When a user signs up on infer.events, the signup page shows two steps:

**Step 1: Install skills**
```
npx skills add infer-events/skills
```

**Step 2: Paste into Claude Code**
```
Save this config to ~/.infer/config.json:
{"project_id":"proj_...","read_key":"pk_read_...","endpoint":"https://api.infer.events"}
Then run /infer-setup to configure everything.
```

The wizard resumes from Step 3 (migration check) → Step 4 (detect) → ...

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| "No spans landed" after setup | App not making LLM calls OR dev server cached old baseURL | Trigger a feature; restart dev server |
| 401 on MCP tool call | Wrong read key in `~/.infer/config.json` | Check key format (`pk_read_...`); re-run setup |
| 401 through the gateway | Upstream key (OpenAI/Anthropic) missing or wrong | Check your `apiKey` field in the client; the gateway passes this through unchanged |
| Spans have null session_id / user_id / metadata | Client doesn't set `x-infer-*` headers | These are optional; set them via the SDK's `defaultHeaders` / `headers` option if you want per-session / per-user analysis |
| "Failed to reconnect" after MCP upgrade | `.mcp.json` has pinned version (`@inferevents/mcp@1.0.0`) | Change args to `["@inferevents/mcp"]` (no version); restart via `/mcp` |
| 502 `infer_unreachable_upstream` on Ollama | baseURL pointed at localhost:11434 (self-hosted) | v1 gateway blocks private addresses. Use Ollama Cloud (`api.ollama.com`); self-hosted is on the v1.1 roadmap |
| 501 `infer_not_implemented` on some endpoint | Endpoint not in the v1 passthrough list (e.g. `/fine_tuning`, `/files`) | Non-chat endpoints are partial in v1 per parent spec §4.4 — open an issue listing the endpoint |
