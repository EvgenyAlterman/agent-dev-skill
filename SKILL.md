---
name: auto-agent-dev
description: >-
  Autonomous "pick a ticket and ship it" dev loop driven by a monday.com board.
  A lightweight dispatcher (the only party authenticated to monday) polls the
  board for the highest-priority task that is Ready for Dev AND assigned to a
  given person, then spawns an isolated, monday-free worker (a fresh Claude in
  its own tmux session + git worktree) that implements the task and opens an
  [Auto-PR] PR. The worker reports its result back through an outcome file; the
  dispatcher reflects it to monday — moving the task to Ready for Testing with
  the PR link, or to Blocked with a specific [Auto-Dev] question when the task is
  too vague to start or a worker hits a critical blocker. Also tracks merged/closed
  PRs: moves tasks to Done on merge or Canceled on close, and deletes the branch.
  Use this whenever the
  user wants to auto-pick / auto-implement monday tasks, work a board ticket
  end-to-end, "grab the next ticket", run an unattended dev loop, dispatch board
  work to background agents, or have an agent self-assign and ship board work —
  even if they don't say the skill name. Monday access goes through `mcp-s-cli`
  (the mcp-s CLI — no direct MCP tool calls), and only the dispatcher uses it.
---

# auto-agent-dev

Turn a monday.com board into an autonomous dev queue using a **dispatcher /
worker** split:

- The **dispatcher** is the only party that talks to monday. It polls the board,
  claims a task, reads its full spec, and spawns an isolated **worker**. On later
  ticks it reads each finished worker's outcome and reflects it back to the board.
- Each **worker** is a fresh Claude in its own **coderplex session** (managed
  worktree + tmux + PTY) with **no monday/MCP access at all**. The dispatcher
  creates the session via the local coderplex daemon HTTP API, writes a task-spec
  file into the worktree, then triggers the session via a brief WebSocket
  connection. The worker implements the task, opens an `[Auto-PR]` PR, writes a
  small outcome file, and continues idling in its session (which the dispatcher
  then deletes via the daemon API).

Why workers are monday-free: workers can't complete the mcp-s interactive
browser sign-in in a headless context, so keeping **all** board I/O in the
authenticated dispatcher sidesteps that entirely and keeps the loop hands-off.
The worker→dispatcher channel is a plain file in the worktree.

This split also keeps polling cheap, lets multiple tasks run in parallel without
clobbering each other, and isolates failures — a stuck worker never takes down
the dispatcher.

> **Authorization note:** the user explicitly opted into this unattended
> architecture and authorized spawning workers with
> `--dangerously-skip-permissions` (see the spawn step). That is a deliberate
> choice for background, hands-off operation — keep it visible, never silent.

## Configuration

Defaults below. Treat them as overridable — if the user names a different board,
assignee, or status label, use theirs.

| Setting              | Default value                                  |
| -------------------- | ---------------------------------------------- |
| Board                | `18417884038` (wix.monday.com/boards/18417884038) |
| Assignee             | `Evgeny Alterman`                              |
| "pick" status        | `Ready For Dev`                                |
| "claimed" status     | `Dev` (set when a worker is spawned, so it isn't re-picked) |
| "done" status        | `Ready For Testing`                            |
| "blocked" status     | `Blocked`                                      |
| "merged" status      | `Done` (set when the task's PR is merged)      |
| "pr-closed" status   | `Canceled` (set when the PR is closed without merging; match by intent at runtime — if the board has no such label, surface to the user rather than guessing) |
| `wixel-agent` repo   | `~/dev/wixel-agent` — all agent runtime, service, admin, chat-ui, simulator work |
| `wixel-ai-tools` repo | `~/dev/wixel-ai-tools` — fal.ai integration, SDK mapping, mapping admin, mapping service |
| Coderplex daemon     | `http://127.0.0.1:3285` (or `$CODERPLEX_DAEMON_URL`) |
| Session tracking     | `~/.auto-agent/cp-sessions.json` — maps `taskId → {sessionId, worktree, repo}` |
| Task-spec file       | `<worktree>/.auto-agent-task.md` (dispatcher → worker) |
| Outcome file         | `<worktree>/.auto-agent-outcome.json` (worker → dispatcher) |
| Poll interval (idle) | 10 minutes (`600s`) — default; used when no workers are in flight |
| Poll interval (active) | 90 seconds — default; workers may also signal instantly via tmux (see W6b) |
| Dispatcher state     | `~/.auto-agent/dispatcher-state.json` — persists cycle counter, `pollIntervalSeconds`, `workerPollIntervalSeconds` |

Status **labels vary per board** — read the board's status column options at
runtime and match by intent rather than assuming these exact strings. If you
can't confidently map a label, that's a question for the user, not a guess.

## The worker outcome file

The single channel from a worker back to the dispatcher. The worker writes
`<worktree>/.auto-agent-outcome.json` as the last thing it does, then exits:

```json
{
  "status": "pr_opened" | "blocked" | "failed",
  "pr_url":  "https://github.com/.../pull/123",   // when pr_opened
  "summary": "one-line description of what shipped",  // when pr_opened
  "question": "specific, answerable question + options + recommended default",  // when blocked
  "reason":  "what went wrong"                     // when failed
}
```

The dispatcher reads this on its next tick and performs the monday update —
**workers never touch monday.**

## Modes

This skill runs in one of two modes. Decide which from how you were invoked:

- **Dispatcher mode (default)** — invoked with no specific task id ("run
  auto-agent-dev", "poll the board", "grab the next ticket"). You own all monday
  I/O, poll, spawn workers, and reflect their outcomes. You do **not** implement
  tasks yourself.
- **Worker mode** — invoked with a specific task id and the word "worker" (the
  dispatcher spawns you this way; see the spawn command). You implement that one
  task with no monday access and exit. You do **not** poll, pick, or touch the
  board.

---

# Dispatcher mode

## Step 1 — Preflight & poll interval

### Poll interval (first invocation only)

On the **very first invocation** of a dispatcher session — i.e. `~/.auto-agent/dispatcher-state.json` does not exist yet, or `cycle` is `0` — ask the user for the desired polling interval using `AskUserQuestion` before doing anything else:

- Question: "How often should the dispatcher poll the board?"
- Options: 5 min, **10 min (Recommended)**, 20 min, 30 min
- Also offer "Other" so the user can type a custom value in seconds.

Store the chosen interval (in seconds) in the state file as `pollIntervalSeconds`. Use it for every `ScheduleWakeup` call in this session. On subsequent ticks the state file already has `pollIntervalSeconds` — read it and use it directly, no need to ask again.

```bash
STATE="$HOME/.auto-agent/dispatcher-state.json"
INTERVAL=$(jq -r '.pollIntervalSeconds // 600' "$STATE" 2>/dev/null || echo 600)
# use $INTERVAL for ScheduleWakeup delaySeconds
```

### Monday access

Monday is reached via **`mcp-s-cli`** — a shell CLI that holds its own OAuth
token (~24h validity) and requires no MCP server or browser re-auth. Use it
with `Bash` tool calls; **do not** use the `mcp__mcp-s__monday__*` MCP tools.

```bash
# Check auth (exit 0 = ok, exit 4 = needs login) — fast, no network
mcp-s-cli check-auth

# Re-login if needed — prints a browser URL to stdout, no local callback required
mcp-s-cli login --remote
```

If `check-auth` exits non-zero, run `mcp-s-cli login --remote`. It prints a
URL to stdout — show that URL to the user as a clickable markdown link and wait
for them to authenticate in the browser before continuing.

**Do NOT use `mcp__mcp-s__authenticate`** — that MCP tool requires the mcp-s
server to be connected, which may itself be broken when auth has expired. The
`mcp-s-cli` path is always available regardless of MCP server state.

**Common operations:**

```bash
# Read board items
mcp-s-cli call monday__all_monday_api '{"query":"query { boards(ids:[18417884038]){items_page(limit:50){items{id name column_values(ids:[\"color_mm4bfq4e\",\"multiple_person_mm4bmdmw\",\"link_mm4bmp5j\"]){id text value}}}}}"}'

# Set status
mcp-s-cli call monday__all_monday_api '{"query":"mutation{change_multiple_column_values(board_id:18417884038,item_id:ITEM_ID,column_values:\"{\\\"color_mm4bfq4e\\\":{\\\"label\\\":\\\"Ready For Testing\\\"},\\\"link_mm4bmp5j\\\":{\\\"url\\\":\\\"PR_URL\\\",\\\"text\\\":\\\"PR #N\\\"}}\"){id}}"}'

# Post a comment
mcp-s-cli call monday__create_update '{"itemId":"ITEM_ID","body":"[Auto-Dev] message here"}'
```

Confirm `claude`, `gh`, and `mcp-s-cli` are on PATH (`which claude gh mcp-s-cli`)
and the coderplex daemon is reachable (`curl -s $CODERPLEX_DAEMON_URL/api/sessions | jq length`).
Workers need an authenticated `gh` to open PRs non-interactively.

## Step 1b — Cycle counter & compaction

Read `~/.auto-agent/dispatcher-state.json` (create it with `{"cycle":0}` if missing).
Increment `cycle` by 1 and write it back. Then check two compaction triggers:

- **Every 10 cycles** (`cycle % 10 === 0`)
- **Token pressure** — treat this as triggered if the cycle counter is high (>50) AND
  you notice the board query responses are getting truncated or context feels large.
  There is no precise token count available inside the skill; use the cycle count as
  the primary signal and token pressure as a secondary one.

**When a compaction trigger fires:**
1. Finish the tick normally (reflect outcomes, pick new work, spawn workers).
2. At the end of the turn, after `ScheduleWakeup`, output a clearly visible notice:

   > **[Auto-Dev] Compaction cycle N reached — please run `/compact` then `start auto-dev polling` to keep context lean.**

3. Do **not** skip the wakeup — the loop continues regardless. The user can compact
   at their convenience; the notice is advisory, not blocking.

**On the first tick after a compaction** (i.e. the tick that runs after the user
has compacted and typed `start auto-dev polling`), re-read the full board state
**before** doing anything else so the fresh context is primed:
- Query all items with their status, owner, priority, and PR link columns.
- Summarize the current state in a short bullet list at the top of the turn output:
  in-flight workers (Dev status + tmux alive), items awaiting PR lifecycle checks
  (Ready For Testing + PR link), and any Ready For Dev candidates.
- Then proceed with the normal tick flow.

The dispatcher cannot tell whether the user just compacted, so apply this re-hydration
whenever it notices its working context feels empty (no remembered board state from
prior ticks). In practice: if this is the first message of a new conversation, always
do the full re-read.

```bash
# Read/increment cycle counter — use jq to preserve all other state fields
STATE="$HOME/.auto-agent/dispatcher-state.json"
[ -f "$STATE" ] || echo '{"cycle":0,"pollIntervalSeconds":600,"workerPollIntervalSeconds":90}' > "$STATE"
CYCLE=$(jq -r '.cycle // 0' "$STATE")
CYCLE=$((CYCLE + 1))
jq --argjson c "$CYCLE" '.cycle = $c' "$STATE" > /tmp/s.json && mv /tmp/s.json "$STATE"
```

## Step 1c — Housekeeping (mandatory every tick, before picking new work)

Run all three housekeeping checks **every tick**, in this order, before searching
for new candidates. Never skip because a prior query returned zero items — a
zero result from a filtered query can be a silent filter failure (see note below).

1. **Reflect finished workers** — check every task in `Dev` state via cp-sessions.json
   (see "Housekeeping — reflect finished workers" section below).
2. **PR lifecycle check** — check every task in `Ready For Testing` that has a PR link;
   handle merged/closed (see "Housekeeping — PR lifecycle tracking" section below).
3. **Follow-up instructions** — check suspended sessions for new thread messages
   (see "Housekeeping — follow-up instructions" section below).

**Query reliability note — always use the full board query for housekeeping:**

Server-side status filters (`compare_value`) can silently return 0 even when
matching items exist (observed in practice). For housekeeping, always fetch the
full `items_page` without a filter and do the filtering client-side:

```bash
# Full board query — reliable, use for all housekeeping checks
mcp-s-cli call monday__all_monday_api '{
  "query": "{ boards(ids: [18417884038]) { items_page(limit: 100) { items { id name column_values(ids: [\"color_mm4bfq4e\",\"multiple_person_mm4bmdmw\",\"link_mm4bmp5j\"]) { id text } } } } }"
}' | python3 -c "
import sys, json
items = json.load(sys.stdin)[\"data\"][\"boards\"][0][\"items_page\"][\"items\"]
for item in items:
    cols = {c[\"id\"]: c[\"text\"] for c in item[\"column_values\"]}
    status = cols.get(\"color_mm4bfq4e\", \"\")
    owners = cols.get(\"multiple_person_mm4bmdmw\", \"\")
    pr = cols.get(\"link_mm4bmp5j\", \"\")
    # Filter client-side:
    # Ready For Testing items with a PR link → PR lifecycle check
    if status == \"Ready For Testing\" and pr:
        print(f'TESTING|{item[\"id\"]}|{item[\"name\"]}|{pr}')
    # Dev items → reflect worker outcomes
    elif status == \"Dev\" and \"Evgeny\" in owners:
        print(f'DEV|{item[\"id\"]}|{item[\"name\"]}')
"
```

Run this once per tick and pipe the output into both the reflect-outcomes check
and the PR lifecycle check — one round-trip to monday, two checks covered.

## Step 2 — Find candidate tasks (strict gate)

A task is a candidate **only if BOTH are true**: status = Ready For Dev **AND**
the owner column contains the configured person. Both are hard gates:

- An **unassigned** Ready-For-Dev task is **not** a candidate — no matter how
  it's named or how obviously it "looks meant for you." Empty owner = skip it.
- A task assigned to **someone else** is **not** a candidate, even if it's the
  only Ready-For-Dev task on the board.

Do not rationalize past the assignee gate. If you catch yourself thinking "this
one isn't assigned to them but is clearly intended," **stop** — that is exactly
the case this guard prevents. Report zero candidates instead.

Query mechanics (keep it token-cheap):

- Filter server-side where possible; request **only** `id`, `name`, the priority
  column, the status column, and the owner column. Pipe through `jq` to trim the
  payload before it reaches context.
- If a filter returns nothing unexpectedly, **probe one item** in full to learn
  the real column IDs/value shapes, then re-run. Never dump the unfiltered board.

**IMPORTANT — always use column ID lookups, never positional index:**

The `column_values` array order is board-defined and not guaranteed. Using
`column_values[0]` or `column_values[1]` will silently read the wrong column and
cause tasks to be missed. Always select by column ID:

```bash
# CORRECT — look up by column ID
mcp-s-cli call monday__all_monday_api '...' | jq -r '
  .data.boards[0].items_page.items[] |
  (.column_values | (map(select(.id=="color_mm4bfq4e"))[0].text) as $status |
   map(select(.id=="multiple_person_mm4bmdmw"))[0].text as $owner |
   select($status=="Ready For Dev" and ($owner | test("Evgeny")))) |
  "\(.id) | \(.name)"'

# WRONG — positional index silently reads wrong column
# .column_values[0].text  ← DO NOT USE
```

## Step 3 — Skip what's already in flight

Before spawning, exclude tasks already being worked:

- Status is already the **claimed** state (`Dev`) → a worker has it (or it's
  awaiting outcome reflection — see Housekeeping). Skip.
- An entry in `~/.auto-agent/cp-sessions.json` exists for the task ID **and** no
  outcome file has been written yet → worker still running. Skip.

This is the double-spawn guard: claiming via status (next step) plus the
cp-sessions check means a task is never handed to two workers.

## Step 4 — Pick and claim the highest-priority candidate

Among remaining candidates, pick the highest priority (read the priority
column's options to order them; tie-break oldest-first). Then **claim it before
spawning** so the next tick won't re-pick it:

1. Set the task status to the **claimed** state (`Dev`).
2. State which task you picked, confirm it's assigned to the configured person,
   and give its priority. If you can't state the assignee match, you don't have a
   valid pick — abort the spawn.

If there are **zero** candidates, that's a correct outcome: report "No
Ready-For-Dev tasks assigned to <assignee> on board <id>", then reschedule
(Step 7) — do not invent or relax anything.

## Step 4b — Determine target repo

Before creating a worktree, decide which repo the code change belongs in. Base this on
**where the implementation will live**, not just task name keywords.

| Repo | Code that lives here |
|------|----------------------|
| `wix-private/wixel-agent` (`~/dev/wixel-agent`) | Agent runtime, session loop, tools, host tools, admin UI, chat UI, codex, simulator, eval, provider adapters, streaming. **Also includes code that consumes fal/mapping via SDK calls or service RPCs** — if the change is on the consumer side (e.g. logging what the agent sends to fal, adjusting how the agent calls a mapping service), it belongs here. |
| `wix-private/wixel-ai-tools` (`~/dev/wixel-ai-tools`) | The fal.ai SDK itself, mapping service implementation, mapping admin, AI tools SDK — i.e. the *provider-side* code, not the consumer. |

**Decision flow:**

1. Read the task name, description, and thread carefully.
2. Ask: *where does the code change actually need to happen?* Consumer-side (agent calling
   an SDK/service) → `wixel-agent`. Provider-side (the SDK or service itself) →
   `wixel-ai-tools`.
3. If the answer is clear, proceed.
4. **If genuinely ambiguous** — the task could reasonably be implemented in either repo
   and a wrong guess would waste the worker run — treat it as a blocker:
   - Do **not** spawn a worker.
   - Post an **`[Auto-Dev]`** comment on the Monday task asking which repo to target,
     with your best-guess recommendation and the reasoning.
   - Leave the task status at the claimed (`Dev`) state for now; do not re-pick it this
     tick. (The user can move it back to Ready For Dev once they answer, and the next
     poll will pick it up.)

**After deciding:**
- Verify the local repo path exists (`ls <path>`). If missing, flag to the user and
  abort the spawn.
- Set `REPO` to the resolved local path. Subsequent steps (spec file, worker spawn,
  `node_modules` hardlink) use this `$REPO`.
- Include the resolved repo name and GitHub slug in the spec file so the worker knows
  which repo it's in and can open the PR against the right one.

## Step 5 — Read the task, check for continuation instructions, write the spec file

The worker has no monday access, so **you** read the task and hand it the spec.
Fetch the task in full via `mcp-s-cli`: description, the updates/conversation thread,
and any links/attachments. Read generously — implementing the wrong thing is far
more expensive than a few tokens of context.

```bash
# Read full task details (description + column values)
mcp-s-cli call monday__all_monday_api \
  '{"query":"query{items(ids:[ITEM_ID]){name body_long column_values{id text value}updates(limit:50,order_by:created_at){id body created_at creator{name}}}}"}'
```

### Step 5a — Scan the thread for continuation instructions

Before writing the spec, **read every update in the task thread** (sorted
oldest-to-newest) and look for recent messages from the assignee that change or
extend what the worker should do. Common patterns:

| Signal in thread | What it means for the worker |
|------------------|------------------------------|
| "rebase on main" / "rebase on master" | Worker must rebase the prior branch onto the latest master before opening the PR — don't create a fresh branch from master |
| "update X in the implementation" / "also add Y" | Additional scope beyond the original description — include it in the spec |
| "continue from PR #N" / "build on top of #N" | Worker should check out the branch from PR #N and continue from there rather than starting fresh |
| "fix the review comments on #N" | Worker should fetch PR #N's review comments and address them |
| Any other instruction from the assignee added **after** the task was last set to Ready For Dev | Treat as binding additional requirements |

**If continuation instructions are present:**
- Note them prominently at the top of the spec file under a `## Continuation Instructions` heading.
- If they reference a prior PR/branch, include the PR URL and branch name so the worker
  can check it out instead of branching from master.
- If a prior branch exists, the worktree should be created from **that branch** rather
  than `origin/master`:
  ```bash
  git -C "$REPO" fetch origin "<prior-branch>"
  git -C "$REPO" worktree add "$WT" -b "task/$ID-$SLUG-v2" "origin/<prior-branch>"
  ```

**If the thread is empty or has no new instructions** — proceed normally, nothing
special to pass.

### Step 5b — Write the spec file

The spec is written **after** the coderplex session reaches `ready` status (Step 6
creates the session and polls for readiness before writing the spec). Write
everything the worker needs into `<worktree>/.auto-agent-task.md`:

- The task id, name, board id, and monday link (for reference only — the worker
  must not call monday).
- The **target repo** — local path and GitHub slug (e.g. `wix-private/wixel-agent`
  or `wix-private/wixel-ai-tools`). The worker uses this for `gh pr create --repo`.
- **Continuation Instructions** (if any — see Step 5a) — placed at the top so the
  worker sees them immediately.
- The full description and acceptance criteria.
- Relevant context from the thread/attachments you judged important.
- A one-line restatement of what "done" looks like.
- **Dispatcher coordinates** (always include — needed for optional reactive wakeup in W6b):

```bash
DISPATCHER_SESSION_ID="${CODERPLEX_SESSION_ID}"
DAEMON="${CODERPLEX_DAEMON_URL:-http://127.0.0.1:3285}"
```

Add near the top of the spec heredoc:
```
## Dispatcher Signal (for reactive wakeup)
- dispatcherSessionId: ${DISPATCHER_SESSION_ID}
- dispatcherTmux: coderplex-${DISPATCHER_SESSION_ID}-claude
- daemonUrl: ${DAEMON}
```

## Step 6 — Spawn an isolated, monday-free worker via coderplex

Workers are created through the **local coderplex daemon** (no raw tmux/git
worktree commands needed — the daemon handles both). Derive a short slug from
the task name (lowercase, hyphenated).

```bash
DAEMON="${CODERPLEX_DAEMON_URL:-http://127.0.0.1:3285}"
REPO_NAME="wixel-agent"   # or "wixel-ai-tools" per Step 4b
ID="<task-id>"; SLUG="<slug>"
BRANCH="task/$ID-$SLUG"
REPO="$HOME/dev/$REPO_NAME"

WORKER_PROMPT="You are an auto-agent-dev WORKER. You have NO monday/MCP access — do not call any mcp__mcp-s tool. Read ~/.claude/skills/auto-agent-dev/SKILL.md and follow WORKER MODE. Your task spec is in ./.auto-agent-task.md; your CWD is already the task git worktree. The dispatcher source repo (for node_modules hardlink) is at $REPO. Open a PR whose title starts with [Auto-PR] using gh pr create --repo wix-private/$REPO_NAME. As the LAST thing you do, write ./.auto-agent-outcome.json and run W6b to signal the dispatcher. Never touch monday."

# 1. Create session — MUST include initialPrompt in the POST body
RESPONSE=$(curl -s -X POST "$DAEMON/api/sessions" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
    --arg repo "$REPO_NAME" \
    --arg branch "$BRANCH" \
    --arg name "auto-task-$ID" \
    --arg prompt "$WORKER_PROMPT" \
    '{repoName:$repo,branch:$branch,initialMode:"claude",customName:$name,initialPrompt:$prompt}')")
SESSION_ID=$(echo "$RESPONSE" | jq -r '.id')
WORKTREE=$(echo "$RESPONSE" | jq -r '.worktree')

if [ -z "$SESSION_ID" ] || [ "$SESSION_ID" = "null" ]; then
  echo "ERROR: session create failed: $RESPONSE"; exit 1
fi

# 2. Poll until worktree is ready (typically < 5s)
for i in $(seq 1 30); do
  STATUS=$(curl -s "$DAEMON/api/sessions" | jq -r --arg id "$SESSION_ID" '.[] | select(.id==$id) | .status')
  [ "$STATUS" = "ready" ] && break
  [ "$STATUS" = "failed" ] && echo "ERROR: session failed" && exit 1
  sleep 2
done

# 3. Write task spec (Step 5b) — worktree is now available
# cat > "$WORKTREE/.auto-agent-task.md" << 'EOF' ... EOF

# 4. Trigger via WebSocket — this causes pty-bridge to spawn claude with the initialPrompt
node -e "
const ws = new WebSocket('ws://127.0.0.1:3285/term/$SESSION_ID?mode=claude&cols=220&rows=50');
ws.addEventListener('open', () => { setTimeout(() => { ws.close(); process.exit(0); }, 5000); });
ws.addEventListener('error', (e) => { console.error('WS error:', e.message); process.exit(1); });
" 2>&1

# 5. Record session in tracking file
CP_SESSIONS="$HOME/.auto-agent/cp-sessions.json"
[ -f "$CP_SESSIONS" ] || echo '{}' > "$CP_SESSIONS"
jq --arg task "$ID" --arg sess "$SESSION_ID" --arg wt "$WORKTREE" --arg repo "$REPO_NAME" \
  '.[$task] = {"sessionId": $sess, "worktree": $wt, "repo": $repo}' \
  "$CP_SESSIONS" > /tmp/cp-sessions-tmp.json && mv /tmp/cp-sessions-tmp.json "$CP_SESSIONS"
```

> **Critical:** `initialPrompt` MUST be in the POST body — passing it as a WebSocket URL
> parameter does NOT work. The daemon reads it from the stored session record when
> pty-bridge spawns claude. Omitting it leaves the session in interactive mode with no task.

**Notes:**
- The coderplex daemon creates the worktree at
  `/home/coder/.coderplex/sessions/<repoName>/<sessionId>/worktree` and the branch
  from the repo's default base (master/main). No manual `git worktree add` needed.
- After the WebSocket connect, the daemon spawns pty-bridge which runs
  `claude --dangerously-skip-permissions -- <prompt>` in a tmux session named
  `coderplex-<sessionId>-claude`. The user can watch it via the coderplex UI.
- **`--dangerously-skip-permissions` is required** (applied automatically by the
  daemon's pty-bridge) — the worker can't answer permission prompts. The user
  explicitly authorized this for unattended operation.
- The "worker done" signal is **the outcome file**, not session/process exit — the
  claude session stays alive after completing. The dispatcher checks the outcome
  file on each tick.
- On success, the dispatcher calls `DELETE /api/sessions/<sessionId>` to clean up
  the session and worktree. On `failed`, leave the session alive for inspection.
- Spawn one worker per valid candidate this tick (respecting the in-flight skip).
- **Self-healing:** If the daemon API changes (wrong endpoint, unexpected response),
  check the GitHub repo `wix-private/coderplex` for the latest API shape before
  retrying. The daemon version is at `$DAEMON/api/health` or via `GET /api/sessions`
  response headers. Key files: `dist/server.js` for routes, `dist/pty-bridge.py`
  for how the initial prompt is launched.

## Step 7 — Reschedule and report

Choose the `ScheduleWakeup` delay dynamically based on whether any workers are
still running:

```bash
# After all spawning/outcome logic is done:
CP_SESSIONS="$HOME/.auto-agent/cp-sessions.json"
IDLE_INTERVAL=$(jq -r '.pollIntervalSeconds // 600' "$STATE")
WORKER_INTERVAL=$(jq -r '.workerPollIntervalSeconds // 90' "$STATE")

# Count workers in flight: tracked sessions whose outcome file doesn't exist yet
WORKERS_ALIVE=0
if [ -f "$CP_SESSIONS" ]; then
  while IFS= read -r ENTRY; do
    WT=$(echo "$ENTRY" | jq -r '.worktree')
    [ ! -f "$WT/.auto-agent-outcome.json" ] && WORKERS_ALIVE=$((WORKERS_ALIVE + 1))
  done < <(jq -c '.[]' "$CP_SESSIONS" 2>/dev/null)
fi

DELAY=$([ "$WORKERS_ALIVE" -gt 0 ] && echo "$WORKER_INTERVAL" || echo "$IDLE_INTERVAL")
# ScheduleWakeup delaySeconds=$DELAY
```

- **Workers in flight** → use `workerPollIntervalSeconds` (default 90s) — catch
  outcomes fast.
- **No workers** → use `pollIntervalSeconds` (default 600s) — cheap idle polling.

Then end the turn with a short status: workers spawned, outcomes reflected, what's
in flight, or "no candidates." Always show the next poll time in **Israel time (IDT/IST)**.

Keep each idle tick to one minimal board query + spawn/skip/outcome logic.
Don't re-tour the board or re-summarize state every cycle.

## Housekeeping — reflect finished workers to monday (each tick)

This is now a core dispatcher job, not just cleanup. For every task currently in
the **claimed** (`Dev`) state, look it up in `~/.auto-agent/cp-sessions.json` and
check its outcome file:

```bash
CP_SESSIONS="$HOME/.auto-agent/cp-sessions.json"
DAEMON="${CODERPLEX_DAEMON_URL:-http://127.0.0.1:3285}"
```

- **Entry in cp-sessions.json, no outcome file** → worker still running. Leave it.
- **Outcome file present** → read `<worktree>/.auto-agent-outcome.json`:
  - `pr_opened` → set status **Ready For Testing**, set the PR-link column, and
    post an **`[Auto-Dev]`** comment with the PR link + the worker's `summary`.
    Record the ID of that comment as `lastProcessedUpdateId`.
    **Suspend the session** — kill the tmux process to free resources, but keep the
    worktree and coderplex session record for potential review-comment work:
    ```bash
    tmux kill-session -t "coderplex-<sessionId>-claude" 2>/dev/null || true
    ```
    Update cp-sessions.json to include `prUrl` and `lastProcessedUpdateId`:
    ```bash
    jq --arg task "$ID" --arg prUrl "$PR_URL" --arg updateId "$UPDATE_ID" \
      '.[$task].prUrl = $prUrl | .[$task].lastProcessedUpdateId = $updateId | .[$task].suspended = true' \
      "$CP_SESSIONS" > /tmp/cp-tmp.json && mv /tmp/cp-tmp.json "$CP_SESSIONS"
    ```
    Full cleanup happens when the PR merges or closes (PR lifecycle step below).
  - `blocked` → set status **Blocked** and post an **`[Auto-Dev]`** comment with
    the worker's `question` (specific + options + recommended default). Then clean
    up: `curl -s -X DELETE "$DAEMON/api/sessions/<sessionId>"` and remove the entry
    from cp-sessions.json. (The dispatcher re-picks the task if/when it returns to
    Ready For Dev.)
  - `failed` → leave the task in `Dev`, post an **`[Auto-Dev]`** comment noting
    the failure `reason`, and flag it to the user. **Do not delete the session** —
    leave it alive so the user can attach and inspect. Keep the entry in
    cp-sessions.json.
- **No entry in cp-sessions.json for a Dev task** → the task was claimed but no
  session was created (e.g. pre-coderplex session, or the dispatcher crashed
  between claim and spawn). Flag to the user; do not re-spawn blindly.

Always reflect outcomes **before** picking new work this tick, so the board never
lags reality.

## Housekeeping — follow-up instructions in Monday thread (each tick)

For every **suspended** session in `cp-sessions.json` (those with `"suspended": true`),
fetch the task's Monday update thread and look for new follow-up instructions from
the assignee.

**The rule:** messages in the thread from the assignee (Evgeny Alterman) that do
**not** start with `[Auto-Dev]` are treated as instructions to the worker. Since
the dispatcher posts as Evgeny's account, `[Auto-Dev]`-prefixed messages are from
the dispatcher; unprefixed messages are from the human.

```bash
# Fetch updates for a task
mcp-s-cli call monday__all_monday_api \
  '{"query":"query{items(ids:[ITEM_ID]){updates(limit:20){id body created_at creator{name}}}}"}'
```

**How to detect a new follow-up:**
1. Get all updates sorted by `created_at` ascending.
2. Find the update whose `id == lastProcessedUpdateId` — that's the last one the
   dispatcher handled.
3. Look at all updates **after** that position.
4. Filter for updates where `creator.name` contains "Evgeny" (or the configured
   assignee) AND `body` does **not** start with `[Auto-Dev]`.
5. If any such updates exist, they are follow-up instructions. Concatenate them in
   order as the instruction text.

**When a follow-up is found:**

1. **Wake the suspended session** — reconnect via WebSocket with a prompt that
   includes the follow-up instructions and the PR context:
   ```bash
   FOLLOWUP_PROMPT="You are an auto-agent-dev WORKER continuing work on task $ID. Your worktree is already set up with the existing PR branch. New instructions from the task owner:\n\n$INSTRUCTIONS\n\nAddress these in the existing branch, push the changes (the PR will update automatically), and write ./.auto-agent-outcome.json when done. Signal the dispatcher via W6b."

   node -e "
   const ws = new WebSocket('ws://127.0.0.1:3285/term/$SESSION_ID?mode=claude&cols=220&rows=50');
   ws.addEventListener('open', () => { setTimeout(() => { ws.close(); process.exit(0); }, 5000); });
   " 2>/dev/null

   sleep 3
   tmux send-keys -t "coderplex-$SESSION_ID-claude" "$FOLLOWUP_PROMPT" Enter
   ```
   Mark `"suspended": false` in cp-sessions.json.

2. **Acknowledge in Monday** — post an `[Auto-Dev]` comment:
   `[Auto-Dev] Picked up follow-up instructions — worker resuming on PR #<N>.`

3. **Update `lastProcessedUpdateId`** to the ID of the last follow-up update you
   processed.

4. **Outcome file** — delete (or rename) the old `.auto-agent-outcome.json` so the
   dispatcher sees fresh state:
   ```bash
   rm -f "$WORKTREE/.auto-agent-outcome.json"
   ```

The worker then runs normally, pushes to the same branch, writes a new outcome,
and signals via W6b. On the next tick the dispatcher reads the outcome and either
posts a new update or handles a block/failure.

## Housekeeping — PR lifecycle tracking (each tick)

After reflecting worker outcomes, scan every **Ready For Testing** task that has a
PR link set (`link_mm4bmp5j` non-empty). For each, check whether the PR has been
merged or closed and update the board accordingly. Run this before picking new
work so the board never lags reality.

**Use the full board query result from Step 1c** — do not run a separate
filtered query here. The server-side status filter for Ready For Testing has been
observed to silently return 0; the unfiltered query is authoritative.

**How to check PR state:**

```bash
# Extract the PR number from the stored URL (e.g. https://github.com/org/repo/pull/123)
PR_NUM=$(echo "$PR_URL" | grep -oE '[0-9]+$')
REPO_SLUG=$(echo "$PR_URL" | sed 's|https://github.com/||' | cut -d/ -f1-2)
gh pr view "$PR_NUM" --repo "$REPO_SLUG" --json state,mergedAt,headRefName
```

**Act on the result:**

- `state == "OPEN"` → leave it. Nothing to do this tick.
- `state == "MERGED"` →
  1. Set the task status to **Done** (the "merged" status).
  2. Post an **`[Auto-Dev]`** comment: `[Auto-Dev] PR #<N> merged — task complete.`
  3. Delete the remote branch: `git push origin --delete "<headRefName>"` (get
     `headRefName` from the `gh pr view` output). Swallow errors if already deleted.
  4. **Delete the coderplex session**: look up the task's entry in `cp-sessions.json`,
     call `curl -s -X DELETE "$DAEMON/api/sessions/<sessionId>"`, and remove the entry.
- `state == "CLOSED"` (not merged) →
  1. Look up the board's status labels at runtime. If a **Canceled** label exists,
     set the task to that status. If not, leave the status unchanged and flag to
     the user — do **not** map it to an arbitrary label.
  2. Post an **`[Auto-Dev]`** comment: `[Auto-Dev] PR #<N> closed without merging — marked Canceled (or see flag above).`
  3. Delete the remote branch the same way as for merged.
  4. **Delete the coderplex session**: same as merged — look up cp-sessions.json,
     DELETE via daemon API, remove the entry.

**Scope:** only act on tasks whose PR link was set by this loop (i.e. the task was
in `Ready For Testing` with a non-null `link_mm4bmp5j`). Do not touch tasks that
reached `Ready For Testing` by other means — if the PR link column is empty, skip
the item entirely. One GraphQL query fetching `id`, `name`, `color_mm4bfq4e`, and
`link_mm4bmp5j` for all `Ready For Testing` items is sufficient; skip any with a
null/empty link value.

---

# Worker mode

You were spawned for **one specific task**, your CWD is already that task's git
worktree, and you have **no monday/MCP access**. Read the spec, implement it,
open the PR, write your outcome file, and exit. Do not poll, do not pick other
tasks, **do not call any monday/mcp-s tool, and never wait for terminal input.**

## W1 — Read the spec

Your task is in `./.auto-agent-task.md` (the dispatcher wrote it). Read it in
full — description, acceptance criteria, and any context it carried over.
Summarize your understanding in a sentence before writing code. Any monday link
in it is reference only; do not call monday.

### W1a — Clarity gate (block back to monday if the task is vague)

Before writing any code, judge whether the spec actually tells you **what to
build and how you'll know it's done.** If it's too thin to implement confidently,
**don't guess** — a wrong guess wastes the whole worker run. Treat it as vague
when, for example:

- The desired outcome or acceptance criteria are missing or contradictory ("make
  it better", "fix the admin page" with no symptom).
- It names a thing you can't locate or disambiguate (which component? which
  endpoint/env? which of several plausible files?).
- Scope is open-ended enough that two reasonable engineers would build
  materially different things.

When it's vague:

1. First try to **resolve it yourself cheaply** — read the relevant code in the
   worktree and the context in the spec. Often the spec is thin but the answer is
   right there in the codebase. Only escalate what you genuinely can't settle.
2. For what remains, **write a `blocked` outcome file** with a specific,
   answerable `question` (offer options + your recommended default) and **exit.**
   The dispatcher posts it to monday as an `[Auto-Dev]` comment and sets Blocked.

Same high bar as W4: only escalate genuine ambiguity a wrong guess can't cheaply
recover from. Routine choices you can reason about are not vagueness — make the
call and note it in the PR.

## W2 — Warm dependencies (yarn monorepos are slow)

These are **yarn workspace monorepos**, and a fresh git worktree starts with **no
`node_modules`**. A cold `yarn install` here routinely takes **10+ minutes** —
longer than a single foreground Bash call is allowed to run. Plan for it:

1. **Reuse what's already installed.** The dispatcher source repo (passed to you
   in the initial prompt, e.g. `~/dev/wixel-agent`) has a warm `node_modules`.
   Hardlink-copy it into your worktree first — near-instant and turns the install
   into a quick reconcile instead of a full download:
   ```bash
   SRC="<dispatcher repo root from spec>"; WT="$(pwd)"
   cp -al "$SRC/node_modules" "$WT/node_modules" 2>/dev/null || true   # hardlinks, seconds not minutes
   ```
2. **Install in the background, then poll** — never block a foreground call on it:
   ```bash
   nohup yarn install --prefer-offline --frozen-lockfile > .auto-agent-install.log 2>&1 &
   ```
   Then poll the log/exit until it finishes (check every ~60–120s; it can run
   well past 10 minutes). Only proceed once install has succeeded. If it fails,
   read the log tail before reacting.
3. If the repo uses a different package manager or a bootstrap script (`yarn
   bootstrap`, `lerna`, `nx`, a `Makefile` target), use that instead — check the
   root `package.json` / README rather than assuming plain `yarn install`.

Don't skip the install and hope — a build against missing/stale deps fails in
confusing ways. Getting deps warm first is the cheapest path to a green build.

## W3 — Implement in the worktree (CWD)

Follow the repo's conventions and lean on existing project skills (Wix
serverless, Scala/Nile tests, PR skill, etc.) rather than reinventing them.

- You're already on the task's feature branch in an isolated worktree — work here.
- Keep the change scoped to what the spec asks; note adjacent work for a
  follow-up rather than expanding the PR.
- Run the project's build/tests before calling it done. A PR that doesn't build
  isn't "ready for testing." Builds/tests in these monorepos are also slow —
  run them in the background and poll, same as install.

## W4 — Critical-blocker protocol (block back to monday)

Reserve this for **critical** blockers — where a wrong guess means throwing the
work away. Covers both a **vague spec you can't safely start** (W1a) and a wall
you hit **partway through**: ambiguous requirements, a missing
credential/endpoint, or a product decision only the assignee can make. Routine
choices you can reason about are not blockers; make a sensible call and note it
in the PR.

On a real blocker: **write a `blocked` outcome file** with a specific, answerable
`question` — include what you tried, the options, and your recommendation — then
**exit.** You have no monday access; the dispatcher posts your question as an
`[Auto-Dev]` comment and sets the task Blocked. Never stop and wait for
interactive terminal input — there is no human watching your console.

## W5 — Open the PR

Once the implementation is verified, push the branch and open a PR with `gh`.

- **The title MUST start with `[Auto-PR]`** so reviewers see at a glance it came
  from this autonomous loop — e.g. `[Auto-PR] docs(admin): mark internal only`.
  No exceptions.
- Use `gh pr create --repo <github-slug>` where `<github-slug>` is the value from
  the spec file (e.g. `wix-private/wixel-agent` or `wix-private/wixel-ai-tools`).
- The description references the monday task (id + link from the spec) and
  summarizes what changed and how it was tested.

## W6 — Write the outcome file

As the **last** thing you do, write `./.auto-agent-outcome.json`:

- Success → `{"status":"pr_opened","pr_url":"<url>","summary":"<one line>"}`
- Blocked → `{"status":"blocked","question":"<specific question + options + recommended default>"}`
- Failed → `{"status":"failed","reason":"<what went wrong>"}`

After writing the file, stop working — do not wait for further input. The
dispatcher reads this file on its next tick (default every 90s), updates monday
(status + `[Auto-Dev]` comment), and keeps the session alive until the PR is
merged or closed. Do **not** call monday yourself.

Note: unlike the old `-p` (headless) mode, your claude session will remain alive
after you write the outcome file. That is fine — the dispatcher detects completion
via the file, not via process exit.

## W6b — Optional: signal the dispatcher immediately (reactive mode)

If the task spec contains a `## Dispatcher Signal` section, you can wake up the
dispatcher instantly instead of waiting for its next 90-second poll. **This is
optional** — the dispatcher will pick up your outcome on the next scheduled wakeup
regardless. But if you want faster turnaround, do both signals after writing the
outcome file:

```bash
# Read dispatcher coordinates from spec
DISPATCHER_TMUX=$(grep 'dispatcherTmux:' .auto-agent-task.md | awk '{print $2}')
DISPATCHER_SID=$(grep 'dispatcherSessionId:' .auto-agent-task.md | awk '{print $2}')
DAEMON_URL=$(grep 'daemonUrl:' .auto-agent-task.md | awk '{print $2}')
OUTCOME_STATUS=$(jq -r '.status' .auto-agent-outcome.json)

# Primary: tmux send-keys — wakes dispatcher in < 1 second
[ -n "$DISPATCHER_TMUX" ] && \
  tmux send-keys -t "$DISPATCHER_TMUX" "start auto-dev polling" Enter 2>/dev/null || true

# Secondary: daemon event — structured persistent record
[ -n "$DAEMON_URL" ] && [ -n "$DISPATCHER_SID" ] && \
  curl -s -X POST "$DAEMON_URL/api/sessions/$DISPATCHER_SID/events" \
    -H "Content-Type: application/json" \
    -d "{\"type\":\"event\",\"message\":\"worker-done:${CODERPLEX_SESSION_ID}:${OUTCOME_STATUS}\",\"agent\":\"worker\"}" \
    2>/dev/null || true
```

Both calls are fire-and-forget (`|| true`). If the dispatcher is mid-turn when
the tmux keys arrive, they buffer in the PTY and fire at the next prompt —
correct behavior, slight delay. The 90-second scheduled poll remains the safety
net in all cases.

---

# Guardrails

- **Assignee is a hard gate, no exceptions.** Only act on a task that is Ready
  For Dev *and* assigned to the configured person. Never pick an unassigned task
  or one assigned to someone else — not even if it's the only Ready-For-Dev task,
  not even if its name suggests it was made for you. No match → report "no
  candidates" and stop. (This rule exists because the skill once picked an
  unassigned task that merely looked intended — don't repeat that.)
- **Only the dispatcher touches monday.** Workers have no monday/MCP access and
  must never call an `mcp__mcp-s` tool; they communicate solely via the outcome
  file. This is what makes unattended workers possible at all.
- **No double-spawn.** Claim via the `Dev` status before spawning, and skip tasks
  already tracked in `cp-sessions.json` without an outcome file. One worker per task.
- **Every PR title starts with `[Auto-PR]`**, and **every monday comment the
  dispatcher posts starts with `[Auto-Dev]`** — so the loop's own activity is
  always attributable at a glance and never mistaken for a human's.
- **Monday is the only place the loop asks for human input.** It never pauses to
  ask in the terminal — a worker writes a `blocked` outcome and the dispatcher
  turns it into a Blocked status + `[Auto-Dev]` question on the task. The loop
  stays hands-off. (Genuine *infrastructure* failures the board can't fix — e.g.
  the dispatcher itself losing monday auth — are the one thing worth surfacing to
  the user directly.)
- **Don't fabricate completion.** A task only moves to Ready For Testing after a
  real PR exists and the build/tests pass. If tests fail, the worker writes
  `failed` (not `pr_opened`) — don't ship.
- **Blocked is for real blockers.** Overusing it turns the loop into a chat;
  underusing it ships wrong work. Ask: is a wrong guess cheaply reversible? If
  yes, decide and note it.
- **Report honestly** at every transition: workers spawned, outcomes reflected,
  PR opened, status set, question posted, or "nothing to do." The user should
  never need to open the board to know what happened.
