# Dispatcher reference

Full implementation guide for **dispatcher mode**. Read this at the start of
every dispatcher turn (it's not huge — ~600 lines; worth the context).

---

## Notifications — Slack only, changes only

**Channel:** `#alterman-auto-dev`

Resolve and **cache** the channel ID once per session:

```bash
SLACK_CHANNEL_ID=$(mcp-s-cli call slack__find-channel-id \
  '{"channelName":"alterman-auto-dev"}' | jq -r '.channelId // empty')
# Guard: empty or "channel not found" means channel doesn't exist yet — skip all notifications
if [ -z "$SLACK_CHANNEL_ID" ] || [ "$SLACK_CHANNEL_ID" = "channel not found" ]; then
  SLACK_CHANNEL_ID=""
fi
```

**Canonical send pattern** (use everywhere — note `channel_id`, not `channel`):
```bash
_notify() {
  local msg="$1"
  [ -z "$SLACK_CHANNEL_ID" ] && return 0
  mcp-s-cli call slack__post_message \
    "$(jq -n --arg ch "$SLACK_CHANNEL_ID" --arg txt "$msg" '{channel_id: $ch, text: $txt}')"
}
# Usage: _notify "🚀 *task name* — worker spawned"
```

**Rule:** notify on every meaningful state change. **Never notify on empty cycles.**

| Event                            | Message format |
| -------------------------------- | -------------- |
| Worker spawned                   | `🚀 *<task name>* — worker spawned (\`<sessionId>\`) on branch \`<branch>\`` |
| PR opened                        | `✅ *<task name>* — PR opened: <pr_url>\n<summary>` |
| Worker blocked                   | `🔶 *<task name>* — worker blocked: <question>` |
| Worker failed                    | `❌ *<task name>* — worker failed: <reason>` |
| Worker stuck / re-triggered      | `⚠️ *<task name>* — worker \`<sessionId>\` was stuck (<reason>), re-triggered` |
| Review feedback picked up        | `🔄 *<task name>* — picked up review feedback, spawning worker on existing PR` |
| Task canceled (worker in flight) | `🚫 *<task name>* — task canceled while worker running; killed worker, closed PR` |
| PR merged → Done                 | `🎉 *<task name>* — PR merged, task Done` |
| PR closed → Canceled             | `🚫 *<task name>* — PR closed without merging, task Canceled` |

**Monday comments policy:** Monday is for **task business data only** — status
changes and the PR link column. Do not post narrative `[Auto-Dev]` comments for
events like "worker spawned", "picking up feedback", "re-triggered", etc. The
only Monday comment that remains is the **blocked question** (because that's
where the assignee will respond) — all narrative goes to Slack.

---

## Meta — Self-improving skill (dispatcher only)

The dispatcher session is the **live source of truth** for this skill. Whenever
the user corrects a behavior, points out a missed case, or requests a change to
how the loop works, **immediately update this skill file** to encode the fix.

**When to update:** any time the user says "you should have…", "next time…",
"add this to the skill", "remember to…", or when you notice a gap that caused a
real problem this session.

**How to update:**
1. Edit the relevant file in `~/dev/agent-dev-skill/` directly (Edit tool) in
   the same turn you apply the behavior change. The symlink
   `~/.claude/skills/auto-agent-dev` → `~/dev/agent-dev-skill` means both
   paths point to the same files.
2. Place the new rule in the most relevant section of `references/dispatcher.md`,
   `references/worker.md`, or `SKILL.md` as appropriate.
3. Write the rule in imperative, durable form — not "the user said X today" but
   "always do X when Y".
4. Briefly tell the user: "Updated skill: <one-line description of the change>."

---

## Step 1 — Preflight & poll interval

### Poll interval (first invocation only)

On the **very first invocation** of a dispatcher session — i.e.
`~/.auto-agent/dispatcher-state.json` does not exist, or `cycle` is `0` — ask
the user for the desired polling interval using `AskUserQuestion` before doing
anything else:

- Question: "How often should the dispatcher poll the board?"
- Options: **5 min (Recommended)**, 10 min, 20 min, 30 min
- Also offer "Other" so the user can type a custom value in seconds.

Store the chosen interval (in seconds) as `pollIntervalSeconds` in the state
file. On subsequent ticks, read it and use it directly — no need to ask again.

```bash
STATE="$HOME/.auto-agent/dispatcher-state.json"
INTERVAL=$(jq -r '.pollIntervalSeconds // 300' "$STATE" 2>/dev/null || echo 300)
```

### Monday access

Monday is reached via **`mcp-s-cli`** — a shell CLI with its own OAuth token
(~24h validity). Use it with `Bash` tool calls; **do not** use the
`mcp__mcp-s__monday__*` MCP tools directly.

```bash
# Check auth (exit 0 = ok, exit 4 = needs login) — fast, no network
mcp-s-cli check-auth

# Re-login if needed — prints a browser URL; no local callback required
mcp-s-cli login --remote
```

If `check-auth` exits non-zero, run `mcp-s-cli login --remote`. Show the printed
URL to the user as a clickable markdown link and wait for authentication before
continuing. **Do NOT use `mcp__mcp-s__authenticate`** — that requires the MCP
server to be running, which may itself be broken when auth has expired.

Confirm tools are on PATH before the first tick:
```bash
which claude gh mcp-s-cli
curl -s "${CODERPLEX_DAEMON_URL:-http://127.0.0.1:3285}/api/sessions" | jq length
```

---

## Step 1b — Cycle counter & compaction

Read `~/.auto-agent/dispatcher-state.json` (if missing, seed it). Increment
`cycle` and write back **using `jq`** to preserve all other fields.

```bash
STATE="$HOME/.auto-agent/dispatcher-state.json"
[ -f "$STATE" ] || echo '{"cycle":0,"pollIntervalSeconds":300,"workerPollIntervalSeconds":90}' > "$STATE"
CYCLE=$(jq -r '.cycle // 0' "$STATE")
CYCLE=$((CYCLE + 1))
jq --argjson c "$CYCLE" '.cycle = $c' "$STATE" > /tmp/s.json && mv /tmp/s.json "$STATE"
```

**Compaction triggers:** every 10 cycles (`cycle % 10 === 0`), or when context
feels large (cycle > 50 and queries are getting truncated).

**When triggered:**
1. Finish the tick normally.
2. Call `ScheduleWakeup` as usual.
3. **Invoke the `/compact` skill directly** as the very last action:
   ```
   Skill({ skill: "compact" })
   ```
   The next wakeup fires into a fresh context and proceeds with full re-hydration.

**On the first tick after a compaction** (or fresh session start), always
re-read the full board state before doing anything else:
- Query all items with status, owner, priority, and PR link.
- Summarize: in-flight workers (Dev + tmux alive), items in Ready For Testing
  with a PR link, and Ready For Dev candidates.
- Then proceed with the normal tick flow.

Apply this re-hydration unconditionally on the first message of any new
conversation — the dispatcher can't distinguish "just compacted" from "fresh."

---

## Step 1c — Housekeeping (mandatory every tick, before picking new work)

Run all four checks **every tick** in this order, before searching for candidates.

1. **Reflect finished workers** — outcome files for tracked sessions
2. **PR lifecycle check** — merged/closed PRs for Ready For Testing tasks
3. **Follow-up instructions** — new thread messages on suspended sessions
4. **Worker health check** — stuck/crashed in-flight workers

**Single board query — reuse everywhere:**

Server-side status filters can silently return 0. Always fetch the full
`items_page` unfiltered and filter client-side. Run this **once per tick** and
reuse the output for all four checks:

```bash
ASSIGNEE="Evgeny"
mcp-s-cli call monday__all_monday_api '{
  "query": "{ boards(ids: [18417884038]) { items_page(limit: 100) { items { id name column_values(ids: [\"color_mm4bfq4e\",\"multiple_person_mm4bmdmw\",\"link_mm4bmp5j\",\"color_mm4bq911\"]) { id text } } } } }"
}' | ASSIGNEE="$ASSIGNEE" python3 -c "
import sys, json, os
assignee = os.environ.get('ASSIGNEE', 'Evgeny')
items = json.load(sys.stdin)['data']['boards'][0]['items_page']['items']
for item in items:
    cols = {c['id']: c['text'] for c in item['column_values']}
    status = cols.get('color_mm4bfq4e', '')
    owners = cols.get('multiple_person_mm4bmdmw', '') or ''
    pr = cols.get('link_mm4bmp5j', '')
    priority = cols.get('color_mm4bq911', '')
    mine = assignee in owners
    if status == 'Ready For Testing' and pr:
        print(f'TESTING|{item[\"id\"]}|{item[\"name\"]}|{pr}')
    elif status == 'Dev' and mine:
        print(f'DEV|{item[\"id\"]}|{item[\"name\"]}')
    elif status == 'Ready For Dev' and mine:
        print(f'CANDIDATE|{item[\"id\"]}|{item[\"name\"]}|priority={priority}')
    elif status == 'Canceled' and item['id'] in cp_sessions:
        print(f'CANCELED_ACTIVE|{item[\"id\"]}|{item[\"name\"]}')
" 2>/dev/null
```

`TESTING` rows → PR lifecycle, `DEV` rows → outcome reflection, `CANDIDATE`
rows → Step 2, `CANCELED_ACTIVE` rows → kill worker (see PR lifecycle section).

---

## Step 2 — Find candidate tasks (strict gate)

A task is a candidate **only if BOTH** are true: status = `Ready For Dev` AND
the owner column contains the configured person.

- Unassigned task → **not** a candidate, period.
- Assigned to someone else → **not** a candidate, period.

**Use the `CANDIDATE` rows from Step 1c.** Never run a separate server-side
filtered query — the `compare_value` filter can silently return 0.

Always look up columns by ID (`cols[id]` dict), never by positional index —
the array order is board-defined and not guaranteed.

---

## Step 3 — Skip what's already in flight

Before spawning, exclude tasks:
- Status is already `Dev` → already claimed. Skip.
- Entry in `cp-sessions.json` with no outcome file yet → worker running. Skip.

---

## Step 4 — Pick and claim the highest-priority candidate

Among remaining candidates, pick the highest priority (read the priority
column's options to order them; tie-break oldest-first). **Claim before
spawning:**

1. Set the task status to the claimed state (`Dev`).
2. Confirm you can state the assignee match. If you can't, abort the spawn.

If zero candidates, report "No Ready-For-Dev tasks assigned to <assignee> on
board <id>" and reschedule (Step 7).

---

## Step 4b — Determine target repo

Decide which repo the code change belongs in based on **where the implementation
lives**, not task name keywords.

| Repo | Code that lives here |
| ---- | -------------------- |
| `wix-private/wixel-agent` (`~/dev/wixel-agent`) | Agent runtime, session loop, tools, host tools, admin UI, chat UI, codex, simulator, eval, provider adapters, streaming. Also consumer-side code that calls fal/mapping via SDK or RPC. |
| `wix-private/wixel-ai-tools` (`~/dev/wixel-ai-tools`) | The fal.ai SDK itself, mapping service implementation, mapping admin, AI tools SDK — the provider side. |

**Decision flow:**
1. Read the task name, description, and thread.
2. Consumer-side change? → `wixel-agent`. Provider-side? → `wixel-ai-tools`.
3. If clear, proceed.
4. **If genuinely ambiguous** — task could belong in either repo and a wrong
   guess would waste the worker run:
   - Do **not** spawn a worker.
   - Send a Slack message to `#alterman-auto-dev` asking which repo to target,
     with your best-guess recommendation and reasoning. (Use `_notify()` defined
     in the Notifications section above.)
   - Leave the task in the claimed (`Dev`) state; the user can move it back to
     Ready For Dev once they answer.

Verify the local repo path exists before spawning (`ls <path>`). If missing,
flag to the user and abort.

Include the resolved repo name and GitHub slug in the spec file.

---

## Step 5 — Read the task, check for continuation instructions, write the spec

Fetch the task in full — description, thread, links. Read generously.

```bash
# Full task details — note: no body_long (not a valid API field); use body for description
mcp-s-cli call monday__all_monday_api \
  '{"query":"query{items(ids:[ITEM_ID]){name body column_values{id text value}updates(limit:50){id body created_at creator{name}}}}"}'
```

### Step 5a — Scan the thread for continuation instructions

Read every update (oldest-to-newest) and look for messages from the assignee
that change or extend what the worker should do:

| Signal in thread | Meaning |
| ---------------- | ------- |
| "rebase on main" / "rebase on master" | Worker must rebase onto latest master before opening the PR |
| "update X" / "also add Y" | Additional scope beyond the description |
| "continue from PR #N" / "build on top of #N" | Check out that branch, don't start fresh |
| "fix the review comments on #N" | Fetch PR #N's review comments and address them |
| Any instruction from the assignee after the last "Ready For Dev" transition | Treat as binding |

If continuation instructions are present, note them under `## Continuation
Instructions` at the top of the spec file. If they reference a prior branch,
create the worktree from that branch:

```bash
git -C "$REPO" fetch origin "<prior-branch>"
git -C "$REPO" worktree add "$WT" -b "task/$ID-$SLUG-v2" "origin/<prior-branch>"
```

### Step 5b — Write the spec file

Write `<worktree>/.auto-agent-task.md` **after** the coderplex session reaches
`ready` (Step 6 polls for this). Include:

- Task id, name, board id, and monday link (reference only — worker must not call monday)
- **Target repo** — local path and GitHub slug
- **Continuation Instructions** (if any — at the top)
- Full description and acceptance criteria
- Relevant thread context
- One-line restatement of what "done" looks like
- **Dispatcher coordinates** (for W6b reactive wakeup):

```
## Dispatcher Signal (for reactive wakeup)
- dispatcherSessionId: <CODERPLEX_SESSION_ID env var>
- dispatcherTmux: coderplex-<CODERPLEX_SESSION_ID>-claude
- daemonUrl: http://127.0.0.1:3285
```

---

## Step 6 — Spawn an isolated worker via coderplex

```bash
DAEMON="${CODERPLEX_DAEMON_URL:-http://127.0.0.1:3285}"
REPO_NAME="wixel-agent"   # or "wixel-ai-tools" per Step 4b
ID="<task-id>"; SLUG="<slug>"
BRANCH="task/$ID-$SLUG"
REPO="$HOME/dev/$REPO_NAME"

WORKER_PROMPT="You are an auto-agent-dev WORKER. You have NO monday/MCP access — do not call any mcp__mcp-s tool. Read ~/.claude/skills/auto-agent-dev/SKILL.md and follow WORKER MODE. Your task spec is in ./.auto-agent-task.md; your CWD is already the task git worktree. The dispatcher source repo (for node_modules hardlink) is at $REPO. Open a PR whose title starts with [Auto-PR] using gh pr create --repo wix-private/$REPO_NAME. As the LAST thing you do, write ./.auto-agent-outcome.json and run W6b to signal the dispatcher. Never touch monday."

# 1. Create session — initialPrompt MUST be in the POST body
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

[ -z "$SESSION_ID" ] || [ "$SESSION_ID" = "null" ] && echo "ERROR: $RESPONSE" && exit 1

# 2. Poll until worktree is ready (typically < 5s)
for i in $(seq 1 30); do
  STATUS=$(curl -s "$DAEMON/api/sessions" | jq -r --arg id "$SESSION_ID" '.[] | select(.id==$id) | .status')
  [ "$STATUS" = "ready" ] && break
  [ "$STATUS" = "failed" ] && echo "ERROR: session failed" && exit 1
  sleep 2
done

# 3. Write task spec into worktree (Step 5b)
# cat > "$WORKTREE/.auto-agent-task.md" << 'EOF' ... EOF

# 4. Trigger via WebSocket — spawns claude with initialPrompt in the PTY
# ws module: try global first, fall back to repo node_modules
WS_MOD=$(node -e "try{require.resolve('ws');console.log('ws')}catch(e){console.log(process.env.HOME+'/dev/wixel-agent/node_modules/ws')}" 2>/dev/null || echo "$HOME/dev/wixel-agent/node_modules/ws")
node -e "
const WebSocket = require('$WS_MOD');
const ws = new WebSocket('ws://127.0.0.1:3285/term/$SESSION_ID?mode=claude&cols=220&rows=50');
ws.addEventListener('open', () => { setTimeout(() => { ws.close(); process.exit(0); }, 5000); });
ws.addEventListener('error', (e) => { console.error('WS error:', e.message); process.exit(1); });
" 2>&1

# 5. Record in tracking file
CP_SESSIONS="$HOME/.auto-agent/cp-sessions.json"
[ -f "$CP_SESSIONS" ] || echo '{}' > "$CP_SESSIONS"
jq --arg task "$ID" --arg sess "$SESSION_ID" --arg wt "$WORKTREE" --arg repo "$REPO_NAME" \
  '.[$task] = {"sessionId":$sess,"worktree":$wt,"repo":$repo}' \
  "$CP_SESSIONS" > /tmp/cp-tmp.json && mv /tmp/cp-tmp.json "$CP_SESSIONS"

# 6. Notify Slack
_notify "🚀 *$TASK_NAME* — worker spawned (\`$SESSION_ID\`) on branch \`$BRANCH\`"
```

> **Critical:** `initialPrompt` MUST be in the POST body — passing it as a
> WebSocket URL parameter does NOT work. Omitting it leaves the session in
> interactive mode with no task.

**Notes:**
- The daemon creates the worktree at
  `/home/coder/.coderplex/sessions/<repoName>/<sessionId>/worktree` from the
  repo's default base. No manual `git worktree add` needed.
- After WebSocket connect, the daemon spawns pty-bridge which runs
  `claude --dangerously-skip-permissions -- <prompt>` in tmux session
  `coderplex-<sessionId>-claude`.
- **`--dangerously-skip-permissions` is applied automatically** by pty-bridge —
  the user authorized this for unattended operation.
- The "worker done" signal is **the outcome file**, not process exit.
- On success, call `DELETE /api/sessions/<sessionId>` to clean up. On `failed`,
  leave it alive for inspection.

---

## Step 7 — Reschedule and report

```bash
CP_SESSIONS="$HOME/.auto-agent/cp-sessions.json"
IDLE_INTERVAL=$(jq -r '.pollIntervalSeconds // 300' "$STATE")
WORKER_INTERVAL=$(jq -r '.workerPollIntervalSeconds // 90' "$STATE")

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

End the turn with a brief status: workers spawned, outcomes reflected, what's in
flight, or "no candidates." Show the next poll time in **Israel time (IDT/IST)**.

Keep idle ticks minimal — one board query, spawn/skip/outcome logic, done.

---

## Housekeeping — reflect finished workers (each tick)

For every entry in `cp-sessions.json`, check the outcome file:

- **No outcome file** → worker still running. Leave it.
- **Outcome `pr_opened`** →
  1. Set task status to **Ready For Testing**, set PR link column (no comment).
  2. Suspend: kill the tmux session to free resources, keep worktree alive.
     ```bash
     tmux kill-session -t "coderplex-<sessionId>-claude" 2>/dev/null || true
     ```
  3. Update cp-sessions.json: add `prUrl`, `suspended: true`, `lastProcessedUpdateId`.
  4. `_notify "✅ *<task name>* — PR opened: <pr_url>\n<summary>"`
- **Outcome `blocked`** →
  1. Set task status to **Blocked**.
  2. Post **one** `[Auto-Dev]` comment on Monday with the worker's `question`
     (this is the one Monday comment that stays — the assignee responds here).
  3. `curl -s -X DELETE "$DAEMON/api/sessions/<sessionId>"`, remove from cp-sessions.json.
  4. `_notify "🔶 *<task name>* — worker blocked: <question>"`
- **Outcome `failed`** →
  1. Leave task in `Dev`, flag to the user. Do NOT delete the session.
  2. `_notify "❌ *<task name>* — worker failed: <reason>"`
- **No entry in cp-sessions.json for a Dev task** → task was claimed but session
  was never created. Flag to the user; do not re-spawn blindly.

Always reflect outcomes **before** picking new work this tick.

---

## Housekeeping — follow-up instructions in Monday thread (each tick)

For every **suspended** session (`"suspended": true` in cp-sessions.json), fetch
the task's Monday thread and look for new follow-up instructions.

```bash
mcp-s-cli call monday__all_monday_api \
  '{"query":"query{items(ids:[ITEM_ID]){updates(limit:20){id body created_at creator{name}}}}"}'
```

**Detection rule:** any message after `lastProcessedUpdateId` that does NOT start
with `[Auto-Dev]` is a follow-up instruction — regardless of who posted it.
This includes the assignee, reviewers, PMs, or anyone. `[Auto-Dev]` messages
are from the dispatcher and are skipped.

> Important: reviewer feedback in the Monday thread is follow-up instructions.
> Do not limit detection to the assignee only — that caused a real miss where
> Maya Wineman's review comments went unactioned for several ticks.

**Algorithm:**
1. Sort updates by `created_at` ascending.
2. Find the update with `id == lastProcessedUpdateId` — that's the last handled.
3. Collect all updates after that position where `body` doesn't start with `[Auto-Dev]`.
4. Concatenate in order, noting each author.

**When a follow-up is found:**

1. Wake the suspended session via WebSocket + tmux:
   ```bash
   FOLLOWUP_PROMPT="You are an auto-agent-dev WORKER continuing work on task $ID. New instructions:\n\n$INSTRUCTIONS\n\nAddress these in the existing branch, push, and write ./.auto-agent-outcome.json when done. Signal the dispatcher via W6b."
   WS_MOD=$(node -e "try{require.resolve('ws');console.log('ws')}catch(e){console.log(process.env.HOME+'/dev/wixel-agent/node_modules/ws')}" 2>/dev/null || echo "$HOME/dev/wixel-agent/node_modules/ws")
   node -e "
   const WebSocket = require('$WS_MOD');
   const ws = new WebSocket('ws://127.0.0.1:3285/term/$SESSION_ID?mode=claude&cols=220&rows=50');
   ws.addEventListener('open', () => { setTimeout(() => { ws.close(); process.exit(0); }, 5000); });
   " 2>/dev/null
   sleep 3
   tmux send-keys -t "coderplex-$SESSION_ID-claude" "$FOLLOWUP_PROMPT" Enter
   ```
   Mark `"suspended": false` in cp-sessions.json.

2. `_notify "🔄 *<task name>* — picked up review/follow-up feedback, worker resuming on PR #<N>"`
   (No Monday comment — Slack is the narrative channel.)

3. Update `lastProcessedUpdateId` to the last processed update's id.

4. Delete the old outcome file so the dispatcher sees fresh state:
   ```bash
   rm -f "$WORKTREE/.auto-agent-outcome.json"
   ```

---

## Housekeeping — PR lifecycle tracking (each tick)

Scan every `TESTING` row from the Step 1c board query (these already have PR
links; don't run a separate query). For each, check the PR state:

```bash
PR_NUM=$(echo "$PR_URL" | grep -oE '[0-9]+$')
REPO_SLUG=$(echo "$PR_URL" | sed 's|https://github.com/||' | cut -d/ -f1-2)
gh pr view "$PR_NUM" --repo "$REPO_SLUG" --json state,mergedAt,headRefName
```

**Act on the result:**
- `OPEN` → leave it.
- `MERGED` →
  1. Set task status to **Done**.
  2. Delete the remote branch: `git push origin --delete "<headRefName>"`.
  3. `curl -s -X DELETE "$DAEMON/api/sessions/<sessionId>"`, remove from cp-sessions.json.
  4. `_notify "🎉 *<task name>* — PR merged, task Done"`
- `CLOSED` (not merged) →
  1. Set task status to **Canceled** (verify label exists first).
  2. Delete remote branch same as merged.
  3. Delete coderplex session, remove from cp-sessions.json.
  4. `_notify "🚫 *<task name>* — PR closed without merging, task Canceled"`

**Manual Canceled while worker in flight:** If a `CANCELED_ACTIVE` row appears
(task is Canceled but cp-sessions.json has an active entry), act immediately:
kill the worker tmux, close the open PR (GitHub comment only), delete the remote
branch, delete the coderplex session, remove from cp-sessions.json.
`_notify "🚫 *<task name>* — task canceled while worker running; killed worker, closed PR"`

---

## Housekeeping — worker health check (each tick)

For every in-flight worker (cp-sessions.json entry with no outcome file, not
suspended), capture its tmux pane and check for stuck/crashed state. Run this
**after** reflecting outcomes, **before** picking candidates.

```bash
jq -r 'to_entries[] | select(.value.suspended != true) | "\(.key) \(.value.sessionId) \(.value.worktree)"' \
  "$CP_SESSIONS" 2>/dev/null | while read TASK_ID SESSION_ID WT; do
  [ -f "$WT/.auto-agent-outcome.json" ] && continue

  TMUX_NAME="coderplex-$SESSION_ID-claude"
  PANE=$(tmux capture-pane -t "$TMUX_NAME" -p -S -20 2>/dev/null)

  if [ $? -ne 0 ]; then
    echo "DEAD|$TASK_ID|$SESSION_ID"
    continue
  fi

  API_ERROR=$(echo "$PANE" | grep -c "API Error:" || true)
  HAS_SPINNER=$(echo "$PANE" | grep -cE "✻|●|⎿" || true)
  HAS_PROMPT=$(echo "$PANE" | grep -c "^❯" || true)
  IDLE=$([ "$HAS_PROMPT" -gt 0 ] && [ "$HAS_SPINNER" -eq 0 ] && echo 1 || echo 0)

  if [ "$API_ERROR" -gt 0 ] || [ "$IDLE" -eq 1 ]; then
    REASON="idle/crashed"
    [ "$API_ERROR" -gt 0 ] && REASON="API Error (529 or similar)"
    echo "STUCK|$TASK_ID|$SESSION_ID|$REASON"
  fi
done
```

**When stuck:**
1. Check `retriggeredAt` in cp-sessions.json. If it was set in the **current
   tick's window** (within the last `workerPollIntervalSeconds`), the worker is
   still stuck after a re-trigger — notify Slack and leave for manual inspection:
   `⚠️ *<task name>* — worker still stuck after re-trigger, needs manual inspection`
2. Otherwise, re-trigger once:
   ```bash
   tmux send-keys -t "coderplex-$SESSION_ID-claude" \
     "Read .auto-agent-task.md and continue from where you left off. Implement the task, push, and write .auto-agent-outcome.json when done." \
     Enter
   jq --arg task "$TASK_ID" --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
     '.[$task].retriggeredAt = $ts' "$CP_SESSIONS" > /tmp/cp-tmp.json && mv /tmp/cp-tmp.json "$CP_SESSIONS"
   ```
   `_notify "⚠️ *<task name>* — worker \`$SESSION_ID\` was stuck ($REASON), re-triggered"`

**When dead** (tmux session gone entirely):
`⚠️ Worker <sessionId> tmux session gone — worker process died. Manual inspection needed.`

---

## Guardrails

- **Assignee is a hard gate.** Only act on Ready For Dev + assigned to the
  configured person. Never pick unassigned or differently-assigned tasks.
- **Only the dispatcher touches monday.** Workers must never call `mcp__mcp-s`
  tools — they communicate solely via the outcome file.
- **No double-spawn.** Claim via Dev status before spawning; skip tasks already
  in cp-sessions.json without an outcome file.
- **Every monday comment starts with `[Auto-Dev]`** — the loop's activity is
  always attributable at a glance.
- **Monday is only for task business data** — status changes and PR link.
  Narrative goes to Slack.
- **Blocked is for real blockers.** Ask: is a wrong guess cheaply reversible?
  If yes, decide and note it in the PR. Don't overuse blocked.
- **Report honestly** at every transition. The user should never need to open
  the board to know what happened.
