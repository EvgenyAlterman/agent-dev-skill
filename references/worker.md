# Worker reference

Full implementation guide for **worker mode**. Read this at the start of your
worker turn. You were spawned for **one specific task**, your CWD is already the
task's git worktree, and you have **no monday/MCP access**. Read the spec,
implement it, open the PR, write your outcome file, and stop.

**Never call any `mcp__mcp-s` tool. Never wait for terminal input.**

---

## W1 — Read the spec

Your task is in `./.auto-agent-task.md`. Read it in full — description,
acceptance criteria, continuation instructions, and dispatcher coordinates.
Summarize your understanding in a sentence before writing code. Any monday link
in it is reference only; do not call monday.

### W1a — Clarity gate (block back if the task is vague)

Before writing any code, judge whether the spec tells you **what to build and
how you'll know it's done.** Block when:

- Desired outcome or acceptance criteria are missing or contradictory.
- It names a thing you can't locate or disambiguate.
- Scope is open-ended enough that two engineers would build materially different things.

**Resolution order:**
1. First, try to resolve it cheaply — read the relevant code in the worktree.
   Often the spec is thin but the answer is right there. Only escalate what you
   genuinely can't settle.
2. For what remains, write a `blocked` outcome file (see W4/W6) with a specific,
   answerable `question` (include options and your recommended default) and exit.

Same high bar as W4: only escalate genuine ambiguity where a wrong guess can't
be cheaply corrected. Routine choices you can reason about are not vagueness.

---

## W2 — Warm dependencies

These are **yarn workspace monorepos**. A fresh worktree has **no `node_modules`**.
A cold `yarn install` can take 10+ minutes — longer than a single foreground
Bash call is allowed to run.

1. **Hardlink from the dispatcher source repo** (near-instant):
   ```bash
   SRC="<dispatcher repo root from spec>"  # e.g. ~/dev/wixel-agent
   cp -al "$SRC/node_modules" "$(pwd)/node_modules" 2>/dev/null || true
   ```
   Only works when `$SRC` and the worktree are on the same filesystem (they
   usually are). If `cp -al` fails, the `|| true` lets the background install
   handle it — not fatal.

2. **Install in the background, then poll:**
   ```bash
   nohup yarn install --prefer-offline --frozen-lockfile > .auto-agent-install.log 2>&1 &
   ```
   Poll the log every ~60–120s. Only proceed once install has succeeded. If it
   fails, read the log tail before reacting.

3. If the repo uses a different bootstrap (`yarn bootstrap`, `nx`, a Makefile
   target), use that instead — check root `package.json` / README.

---

## W3 — Implement

Follow the repo's conventions and lean on existing project skills (Wix
serverless, PR skill, etc.) rather than reinventing them.

- You're already on the task's feature branch in an isolated worktree — work here.
- Keep the change scoped to what the spec asks. Note adjacent work for a
  follow-up rather than expanding the PR.
- Run the project's build/tests before calling it done. A PR that doesn't build
  is not "ready for testing." Builds/tests in these monorepos are slow — run in
  background and poll, same as install.

---

## W4 — Critical-blocker protocol

Reserve this for **critical** blockers — where a wrong guess means throwing work
away. Covers both a vague spec you can't safely start (W1a) and a wall hit
partway through: ambiguous requirements, missing credential/endpoint, or a
product decision only the assignee can make.

On a real blocker: **write a `blocked` outcome file** with a specific, answerable
`question` — include what you tried, the options, and your recommendation —
then **exit.** You have no monday access; the dispatcher posts your question as
an `[Auto-Dev]` comment and sets the task Blocked.

---

## W5 — Open the PR

Push the branch and open a PR with `gh`.

- **Title MUST start with `[Auto-PR]`** — e.g. `[Auto-PR] feat(admin): add export button`.
- Use `gh pr create --repo <github-slug>` where the slug is from the spec file
  (e.g. `wix-private/wixel-agent` or `wix-private/wixel-ai-tools`).
- Description references the monday task (id + link from spec) and summarizes
  what changed and how it was tested.

---

## W6 — Write the outcome file

**As the last thing you do**, write `./.auto-agent-outcome.json`:

```json
// Success:
{"status": "pr_opened", "pr_url": "<url>", "summary": "<one line>"}

// Blocked:
{"status": "blocked", "question": "<specific question + options + recommended default>"}

// Failed:
{"status": "failed", "reason": "<what went wrong>"}
```

After writing the file, stop working. Do not wait for further input. The
dispatcher reads this file on its next tick (default every 90s) and updates
monday. **Do not call monday yourself.**

Your claude session remains alive after writing the outcome — that's fine. The
dispatcher detects completion via the file, not process exit.

---

## W6b — Signal the dispatcher immediately (reactive mode)

If the spec has a `## Dispatcher Signal` section, you can wake the dispatcher
instantly instead of waiting for the next 90-second poll. **This is optional**
— the scheduled poll is the safety net regardless. If you want faster
turnaround, do both signals after writing the outcome file:

```bash
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

Both calls are fire-and-forget (`|| true`). The 90-second scheduled poll remains
the safety net in all cases.

---

## Worker guardrails

- **No monday/MCP access.** Never call any `mcp__mcp-s` tool. The outcome file
  is your only channel back to the dispatcher.
- **Never wait for terminal input.** Write a `blocked` outcome and exit instead.
- **`[Auto-PR]` title is mandatory.** No exceptions.
- **Build must pass before PR.** Don't write `pr_opened` if the build is red.
- **Blocked is for real blockers.** Routine choices you can reason about are not
  blockers — make a sensible call and note it in the PR.
