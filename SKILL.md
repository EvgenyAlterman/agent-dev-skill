---
name: auto-agent-dev
model: sonnet
description: >-
  Autonomous "pick a ticket and ship it" dev loop driven by a monday.com board.
  A dispatcher (the only party authenticated to monday) polls the board for the
  highest-priority Ready-For-Dev task assigned to a given person, spawns an
  isolated monday-free worker (fresh Claude in its own tmux + git worktree) that
  implements the task and opens an [Auto-PR] PR, then reflects outcomes back to
  monday. Tracks merged/closed PRs, handles blocked/failed workers, detects
  stuck workers, and picks up follow-up review comments from the thread.
  Use this skill whenever the user wants to: auto-pick or auto-implement monday
  tasks, work a board ticket end-to-end, "grab the next ticket", run an
  unattended dev loop, dispatch board work to background agents, or have an
  agent self-assign and ship board work — even if they don't say the skill name.
  Also triggers on "start polling", "run the dispatcher", "agent-dev", or any
  phrasing about auto-implementing or background-shipping monday tasks.
  Monday access goes through `mcp-s-cli` only; the dispatcher is the sole
  monday-authenticated party.
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

| Setting               | Default value                                                                                                                                                     |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Board                 | `18417884038` (wix.monday.com/boards/18417884038)                                                                                                                 |
| Assignee              | `Evgeny Alterman`                                                                                                                                                 |
| "pick" status         | `Ready For Dev`                                                                                                                                                   |
| "claimed" status      | `Dev` (set when a worker is spawned, so it isn't re-picked)                                                                                                       |
| "done" status         | `Ready For Testing`                                                                                                                                               |
| "blocked" status      | `Blocked`                                                                                                                                                         |
| "merged" status       | `Done` (set when the task's PR is merged)                                                                                                                         |
| "pr-closed" status    | `Canceled` (set when the PR is closed without merging; match by intent at runtime — if the board has no such label, surface to the user rather than guessing)     |
| `wixel-agent` repo    | `~/dev/wixel-agent` — agent runtime, service, admin, chat-ui, simulator work                                                                                     |
| `wixel-ai-tools` repo | `~/dev/wixel-ai-tools` — fal.ai SDK, mapping service implementation, mapping admin                                                                               |
| Coderplex daemon      | `http://127.0.0.1:3285` (or `$CODERPLEX_DAEMON_URL`)                                                                                                             |
| Session tracking      | `~/.auto-agent/cp-sessions.json` — maps `taskId → {sessionId, worktree, repo}`; entries grow `prUrl`, `lastProcessedUpdateId`, `suspended`, `retriggeredAt`      |
| Task-spec file        | `<worktree>/.auto-agent-task.md` (dispatcher → worker)                                                                                                           |
| Outcome file          | `<worktree>/.auto-agent-outcome.json` (worker → dispatcher)                                                                                                      |
| Poll interval (idle)  | 5 minutes (`300s`) — used when no workers are in flight                                                                                                           |
| Poll interval (active)| 90 seconds — workers may also signal instantly via tmux (see W6b)                                                                                                |
| Dispatcher state      | `~/.auto-agent/dispatcher-state.json` — persists cycle counter, `pollIntervalSeconds`, `workerPollIntervalSeconds`                                               |
| Slack channel         | `#alterman-auto-dev` — notifications only; resolve channel ID at runtime via `mcp-s-cli`                                                                         |

Status **labels vary per board** — read the board's status column options at
runtime and match by intent rather than assuming these exact strings. If you
can't confidently map a label, that's a question for the user, not a guess.

## The worker outcome file

The single channel from a worker back to the dispatcher. The worker writes
`<worktree>/.auto-agent-outcome.json` as the last thing it does, then exits:

```json
{
  "status": "pr_opened" | "blocked" | "failed",
  "pr_url":  "https://github.com/.../pull/123",    // when pr_opened
  "summary": "one-line description of what shipped", // when pr_opened
  "question": "specific question + options + recommended default", // when blocked
  "reason":  "what went wrong"                       // when failed
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
  → **Immediately `Read ~/.claude/skills/auto-agent-dev/references/dispatcher.md`**
    before doing anything else.

- **Worker mode** — invoked with a specific task id and the word "worker" (the
  dispatcher spawns you this way; see the spawn command). You implement that one
  task with no monday access and exit. You do **not** poll, pick, or touch the
  board.
  → **Immediately `Read ~/.claude/skills/auto-agent-dev/references/worker.md`**
    before doing anything else.
