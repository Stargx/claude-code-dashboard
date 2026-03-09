# Claude Code Dashboard — Project Brief

## What This Is
A lightweight localhost web dashboard that aggregates and visualises live Claude Code
session data across multiple concurrent terminal sessions. No cloud, no auth, no backend
beyond a small Node.js watcher process. Think: master control panel for power users
running 2–4 Claude Code instances simultaneously.

## The Problem It Solves
Claude Code has no cross-session visibility. If you're running two sessions in separate
terminal windows you must alt-tab to check status, there's no combined token/cost view,
and you can't see which session is actively doing something vs waiting for input.

## Target User
Solo developers running multiple Claude Code sessions. Initially: the developer of this
tool (dogfooding). Potential open-source release if useful to others.

---

## Architecture

### Data Source
Claude Code writes JSONL session logs to:
  `~/.claude/projects/<project-hash>/<session-uuid>.jsonl`

Each line is a JSON event. Relevant event types to parse:
- `assistant` — contains `usage` (input_tokens, output_tokens), message content
- `user` — tool_result events, human turns
- `tool_use` — tool name, input (file paths, bash commands etc.)
- `summary` — session summary / task description if present

**First task: explore and document the actual JSONL schema before building anything.**
Run `ls ~/.claude/projects/` and examine a real session file to confirm field names.
Do not assume schema — verify it.

### Process Model
Two components:

1. **Watcher process** (`watcher.js`)
   - Uses `chokidar` to watch `~/.claude/projects/**/*.jsonl`
   - Parses new lines as they're appended (tail behaviour)
   - Maintains in-memory session state map
   - Serves state via a simple Express HTTP endpoint: `GET /api/sessions`
   - Also serves the static frontend

2. **Frontend** (`index.html` / vanilla JS or a single React bundle)
   - Polls `/api/sessions` every 2 seconds (simple, no WebSocket complexity yet)
   - Renders the dashboard UI
   - No build step if possible — prefer a single-file approach initially

### Session State Shape (per session)
```json
{
  "sessionId": "uuid",
  "projectHash": "abc123",
  "projectPath": "/resolved/path/if/available",
  "label": "derived from project path basename",
  "status": "thinking | waiting | idle",
  "currentTask": "last assistant message summary or tool in progress",
  "tokensIn": 48320,
  "tokensOut": 12840,
  "costUSD": 0.42,
  "turnCount": 18,
  "activeFiles": ["FacialController.cpp"],
  "recentLog": [ { "time": "...", "type": "tool|think|ok", "msg": "..." } ],
  "startedAt": "ISO timestamp of first event",
  "lastEventAt": "ISO timestamp of most recent event"
}
```

---

## Pricing Constants (verify against current Anthropic pricing)
```js
const PRICING = {
  "claude-sonnet-4-20250514": { input: 3.00, output: 15.00 }, // per 1M tokens
  "claude-opus-4-20250514":   { input: 15.00, output: 75.00 },
  "claude-haiku-4-5":         { input: 0.80, output: 4.00 },
};
```
Model name is in the JSONL — extract it and apply correct pricing.

---

## Tech Stack
- **Runtime**: Node.js (whatever version is already installed)
- **Watcher deps**: `chokidar`, `express`
- **Frontend**: Single HTML file with embedded JS. Use React via CDN if needed.
  No webpack, no Vite, no build step — keep it zero-friction to run.
- **Styling**: Dark terminal aesthetic. IBM Plex Mono. See UI reference below.

---

## UI Requirements
Reference the existing mockup design (dark/terminal aesthetic):
- Status colour coding: green = thinking/active, yellow = waiting, grey = idle
- Per-session context window % bar
- Combined stats header: total tokens in/out, total cost, active session count
- Expandable session cards with recent log feed
- Auto-refresh every 2s — no manual reload needed

The frontend mockup exists as `claude-code-dashboard.jsx` — use it as a visual
reference and adapt to vanilla JS/HTML or a CDN React bundle.

---

## Project Path Label Resolution
Claude Code project hashes are SHA-256 of the working directory path. To show human-
readable labels, reverse-lookup by reading `.claude/projects/<hash>/` for any stored
metadata, OR read the `cwd` field from the first JSONL event in the session.
Fallback: show last two path segments of the hash directory name.

---

## Status Detection Logic
Derive session status from recency of last event:
- Event within last 10s AND last event was a `tool_use` → `thinking`
- Event within last 10s AND last event was an `assistant` message with no tool → `waiting`
- Last event > 60s ago → `idle`
- Any `error` type event in last 3 events → `error`

---

## Development Phases

### Phase 1 — Schema Discovery (do this first, before any code)
- Examine real JSONL session files
- Document actual field names and event types
- Confirm token usage field locations
- Note any surprises

### Phase 2 — Watcher + API
- `npm init`, install `chokidar express`
- Build `watcher.js`: file watching, JSONL parsing, session state, `/api/sessions` endpoint
- Test with `curl localhost:3001/api/sessions`

### Phase 3 — Frontend
- Build `index.html` served by Express
- Adapt dashboard mockup design
- Poll `/api/sessions`, render cards

### Phase 4 — Polish
- Project path label resolution
- Status detection refinement
- Cost daily estimate projection
- "Open terminal at session" button (launches `wt` or `gnome-terminal` at CWD)

---

## Non-Goals (for now)
- WebSocket live push (polling is fine for v1)
- Session control / sending messages to Claude Code
- Windows tray icon or system notifications
- Multi-machine / remote sessions
- Authentication (localhost only)

---

## Running It
Target invocation:
```bash
node watcher.js
# Dashboard available at http://localhost:3001
```

Ideally add a note to personal dotfiles / shell profile so it auto-starts.

---

## Notes
- This is a personal dev tool first. Don't over-engineer it.
- If the JSONL schema differs significantly from assumptions, update this doc before
  proceeding — don't paper over it with workarounds.
- The Architect (Claude Code) should feel free to flag if any phase assumption is wrong
  rather than silently building against a bad assumption.
