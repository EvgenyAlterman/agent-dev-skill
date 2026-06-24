# auto-agent-dev

An autonomous "pick a ticket and ship it" loop powered by Claude Code. The skill turns a monday.com board into a self-driving dev queue: it polls for tasks, spawns isolated Claude workers that implement them, and reflects outcomes (PR opened, merged, GA deployed) back to monday and Slack — unattended.

---

## Core concept: Dispatcher / Worker split

The loop has two roles that never mix:

```
┌─────────────────────────────────────────────────────────┐
│  DISPATCHER (this Claude session)                       │
│  - only party authenticated to monday + Slack           │
│  - polls the board, claims tasks, spawns workers        │
│  - reads outcome files, updates monday                  │
│  - runs on ScheduleWakeup every 5 min (idle) / 90s      │
└───────────────────┬─────────────────────────────────────┘
                    │  coderplex HTTP API
                    │  (creates isolated session + worktree)
                    ▼
┌─────────────────────────────────────────────────────────┐
│  WORKER (fresh Claude, one per task)                    │
│  - zero monday / MCP access                             │
│  - reads .auto-agent-task.md, implements, opens PR      │
│  - writes .auto-agent-outcome.json and stops            │
│  - lives in its own git worktree and tmux session       │
└─────────────────────────────────────────────────────────┘
```

**Why the split?** Workers run headless and can't complete the interactive browser OAuth that `mcp-s-cli` requires. All board I/O stays in the one authenticated dispatcher, workers communicate back through a plain JSON file.

---

## How it's wired on this machine

### Infrastructure

| Component | What it is | Where |
|-----------|-----------|-------|
| **Coderplex daemon** | Local HTTP server that manages worktrees + tmux sessions for Claude workers | `http://127.0.0.1:3285` (`$CODERPLEX_DAEMON_URL`) |
| **mcp-s-cli** | CLI that talks to the Wix MCP gateway (monday, Slack, devex tools). The dispatcher's only external I/O channel | `/usr/bin/mcp-s-cli` |
| **gh** | GitHub CLI — used for PR creation, PR state checks, branch deletion | `/usr/bin/gh` |
| **tmux** | Used by coderplex to host worker Claude sessions; also used by dispatcher to send follow-up prompts to suspended workers | system |
| **ScheduleWakeup** | Claude Code harness tool — reschedules the dispatcher's next tick | built-in |

### Source repositories

| Repo | Local path | Primary devex project | What lives here |
|------|-----------|----------------------|-----------------|
| `wix-private/wixel-agent` | `~/dev/wixel-agent` | `com.wixpress.wixel.wixel-agent-server` | Agent runtime, session loop, tools, admin UI, chat UI, codex, simulator |
| `wix-private/wixel-ai-tools` | `~/dev/wixel-ai-tools` | `com.wixpress.wixel.wixel-ai-models-mapping-service` | fal.ai SDK, model mapping service and admin |
| `wix-private/business-formation-tools` | `~/dev/business-formation-tools` | Multiple (see GA section) | AI assistant chat UI, business creation/launcher/branding services |

### State files

```
~/.auto-agent/
  dispatcher-state.json   # cycle counter + poll intervals
  cp-sessions.json        # taskId → {sessionId, worktree, repo, prUrl, suspended, ...}
  pending-ga.json         # taskId → {mergeCommit, projectName, rcVersion, ...}
```

### monday board

Board ID: `18417884038` ([wix.monday.com/boards/18417884038](https://wix.monday.com/boards/18417884038))

Column IDs (resolved once, hardcoded):
- `color_mm4bfq4e` — Status
- `multiple_person_mm4bmdmw` — Owner
- `link_mm4bmp5j` — PR link
- `color_mm4bq911` — Priority

Status flow: `Ready For Dev` → `Dev` (claimed) → `Ready For Testing` (PR opened) → `Done` (PR merged) → GA notification

### Slack

Channel: `#alterman-auto-dev` (ID: `C0BCE486ENS`)

---

## Dispatcher tick — step by step

Each wakeup runs through these steps in order:

### 1. Preflight
- `mcp-s-cli check-auth` — if expired, run `mcp-s-cli login --remote`, show URL to user
- Verify tools on PATH: `claude`, `gh`, `mcp-s-cli`

### 2. Board query (single query, reused everywhere)
Fetch all items unfiltered — **never use server-side status filters** (they silently return 0 results). Filter client-side by status + owner. Produces four row types:
- `TESTING` — PR opened, awaiting merge
- `DEV` + assigned to Evgeny — in-flight (claimed)
- `CANDIDATE` — Ready For Dev + assigned to Evgeny
- `CANCELED_ACTIVE` — task was canceled while a worker was running

### 3. Housekeeping (all four checks, in order)

**3a. Reflect finished workers** — for every entry in `cp-sessions.json`, check if `.auto-agent-outcome.json` exists in the worktree:
- `pr_opened` → set status to Ready For Testing, set PR link column, post monday comment, suspend session (don't kill — keeps it alive for follow-up feedback via tmux), notify Slack
- `blocked` → set status to Blocked, post monday `[Auto-Dev]` comment with the question, delete session, notify Slack
- `failed` → leave session alive for inspection, notify Slack

**3b. PR lifecycle** — for every `TESTING` row, `gh pr view` to check state:
- `MERGED` → set task Done, delete remote branch, add to `pending-ga.json`, delete coderplex session, notify Slack
- `CLOSED` (not merged) → set task Canceled, delete branch + session, notify Slack

**3c. Follow-up instructions** — for every suspended session, fetch monday thread updates after `lastProcessedUpdateId`. Any message that doesn't start with `[Auto-Dev]` or `✅ [Auto-PR]` is a follow-up (from anyone — assignee, reviewer, PM). When found: delete old outcome file, send instructions to the worker via `tmux send-keys`, mark session not-suspended, update `lastProcessedUpdateId`.

**3d. GA check** — for every entry in `pending-ga.json`:
1. If `rcVersion` is null, call `devex__where_is_my_commit` to find which RC contains the merge commit
2. Call `devex__get_rollout_history` and find the latest SUCCEEDED GA `toVersion`
3. Semver compare: if `gaVersion >= rcVersion` → GA deployed → post monday comment + Slack, remove from file

**3e. Worker health** — for every in-flight (not suspended, no outcome file) worker, capture 5 lines of its tmux pane. If idle prompt visible or API error detected → re-trigger once via `tmux send-keys`. If still stuck after a previous re-trigger → notify Slack for manual inspection.

### 4. Pick candidate
Among `CANDIDATE` rows not already in `cp-sessions.json`, pick highest priority. Set status to `Dev` (claim), then spawn a worker.

### 5. Spawn worker

```
1. git pull origin master on target repo
2. POST /api/sessions to coderplex daemon
   → creates worktree at /home/coder/.coderplex/sessions/<repo>/<sessionId>/worktree
   → initialPrompt MUST be in the POST body (not the WebSocket URL)
3. Poll GET /api/sessions until status == "ready"
4. Write .auto-agent-task.md into the worktree
5. Connect WebSocket to ws://127.0.0.1:3285/term/<sessionId>?mode=claude
   → daemon spawns: claude --dangerously-skip-permissions -- <prompt>
   → close WebSocket after 5s (trigger-only, not persistent)
6. Record in cp-sessions.json
7. Notify Slack
```

`--dangerously-skip-permissions` is applied automatically by the coderplex pty-bridge. The user explicitly authorized this for unattended operation.

### 6. Reschedule
- Workers alive (no outcome file, not suspended) → 90s interval
- No active workers → 300s (5 min) interval

---

## Worker flow

The worker reads `.auto-agent-task.md`, which contains:
- Task name, monday link (reference only — never call monday)
- Target repo path and GitHub slug
- Full description and acceptance criteria
- Continuation instructions (if any)
- Dispatcher tmux coordinates (for reactive W6b signal)

Then:
1. **W1** — Read spec, clarity gate (block if genuinely ambiguous)
2. **W2** — Warm dependencies: `cp -al <dispatcher-repo>/node_modules ./node_modules`, then `yarn install --prefer-offline` in background
3. **W3** — Implement. Stay on the feature branch (already checked out by coderplex). Build + test in background, poll for results.
4. **W4** — If blocked, write `blocked` outcome and exit
5. **W5** — `gh pr create --repo <slug>` with `[Auto-PR]` title
6. **W6** — Write `.auto-agent-outcome.json` as the very last action
7. **W6b** — Optional: signal dispatcher immediately via `tmux send-keys` and daemon event POST

---

## GA tracking

When a PR merges, the dispatcher adds it to `pending-ga.json` and tracks it until the artifact is GA'd.

**Resolving the RC version:**
```bash
mcp-s-cli call devex__where_is_my_commit '{
  "repo_url": "git@github.com:wix-private/<repo>.git",
  "commit_hash": "<merge-commit>",
  "project_name": "<GroupId.ArtifactId>"
}'
# Returns: {"release": {"version": "1.102.0", ...}}
# Returns NOT_FOUND if the build hasn't been cut yet — retry next tick
```

**Checking if GA'd:**
```bash
mcp-s-cli call devex__get_rollout_history '{"groupId": "...", "artifactId": "...", "limit": 5}'
# Response is wrapped: content[0].text is a JSON string — parse with python3
# Look for: type == "GA" && status == "SUCCEEDED" → toVersion
# GA'd if: semver(toVersion) >= semver(rcVersion)
```

**Multi-project repos** (business-formation-tools has 5+ deployable projects): iterate through all known projects and call `where_is_my_commit` for each; collect all hits and report all artifact versions in the GA notification.

**Repo → project mapping:**

| Repo | Projects to check |
|------|------------------|
| `wixel-agent` | `com.wixpress.wixel.wixel-agent-server` |
| `wixel-ai-tools` | `com.wixpress.wixel.wixel-ai-models-mapping-service` |
| `business-formation-tools` | `com.wixpress.wixel-ai-assistant`, `com.wixpress.exposure.business-creation-service`, `com.wixpress.brand-palette-service`, `com.wixpress.branding.preset-creation-service`, `com.wixpress.formation-tools.business-term-service` |

---

## Key nuances and gotchas

### ScheduleWakeup always triggers a "no visible output" follow-up
The harness always fires `[Your previous response had no visible output. Please continue and produce a user-visible response.]` after a turn that ends with `ScheduleWakeup`, even if text was produced before it. **Rule: never write text before `ScheduleWakeup`.** Write the cycle summary only in response to the `[no visible output]` prompt — that's the one place where it appears exactly once.

### Never use `mcp__mcp-s__*` tools — always use `mcp-s-cli`
The `mcp__mcp-s__*` MCP tools use a short-lived in-process token that expires frequently and cannot be refreshed without an interactive browser flow. `mcp-s-cli` manages a separate ~24h OAuth token. Always use `mcp-s-cli call <tool> '<json>'` for all monday, Slack, and devex calls.

### Server-side monday status filters silently return 0 results
The monday `items_page` API's server-side `compare_value` filter for status columns is unreliable — it silently returns an empty list for valid filters. Always fetch **all items unfiltered** and filter client-side in Python.

### Column values must be accessed by ID, not by index
The `column_values` array order is board-defined and not guaranteed. Always build a `{id: text}` dict: `cols = {c['id']: c['text'] for c in item['column_values']}`.

### devex__get_rollout_history response format
The response wraps the history inside `content[0].text` as a JSON string (not a native object). Always unwrap:
```python
outer = json.load(sys.stdin)
history = json.loads(outer['content'][0]['text'])
```

### Worker sessions stay alive after PR (suspended, not deleted)
After a worker opens a PR, the session is **suspended** (not deleted). The tmux session stays alive so the dispatcher can send follow-up review feedback via `tmux send-keys` without needing a new WebSocket spawn. Sessions are only deleted when: PR merges, PR closes, or worker is blocked/failed.

### Follow-up detection uses lastProcessedUpdateId, not author
Any monday update after the anchor (the PR-opened `[Auto-PR]` comment) that doesn't start with `[Auto-Dev]` is treated as a follow-up — from anyone (assignee, reviewer, PM). Filtering by author caused real misses.

### initialPrompt must be in the POST body
When creating a coderplex session, `initialPrompt` must be a field in the JSON POST body. Passing it as a WebSocket URL parameter does NOT work — the session starts in interactive mode with no task.

### Claim before spawn (no double-spawn)
Set task status to `Dev` before spawning the worker. This is the idempotency gate — if the dispatcher crashes between claim and spawn, the task stays in `Dev` and won't be re-picked on the next tick.

### monday is for task data only, Slack is for narrative
The only monday comment the loop posts is the **blocked question** (because the assignee replies there). All other events — worker spawned, PR opened, re-triggered, merged, GA'd — go to Slack only.

---

## Self-improving skill

The dispatcher session is the live source of truth for this skill. When the user corrects a behavior, the dispatcher immediately edits the relevant file in `~/dev/agent-dev-skill/` (symlinked from `~/.claude/skills/auto-agent-dev`) and pushes to `github.com/EvgenyAlterman/agent-dev-skill`. Changes take effect on the next wakeup.
