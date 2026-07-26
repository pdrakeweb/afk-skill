# The supervisor protocol

**The supervisor does not do the work.**

That is the whole design, and it is easy to erode. An agent that hits a
permission prompt is stuck and *cannot report that it is stuck* — being stuck is
precisely the state of having no turn left in which to speak. So something
outside it must notice. An agent that is busy doing the work has no attention
left to notice, and its context fills with implementation detail until it can no
longer hold the absence's state.

So: sub-agents do the work; the supervisor watches, cancels, re-briefs, and
logs. The supervisor's context stays small enough to last seven hours.

---

## Tools this uses

| Need | Tool |
|---|---|
| Spawn a worker | `Agent`, `run_in_background: true` (the default) |
| Isolate parallel file edits | `Agent` with `isolation: "worktree"` |
| See what's running | `TaskList` |
| Kill a stalled worker | `TaskStop` with the agent's name or ID |
| Send an agent more context | `SendMessage` (resumes it with context intact) |
| Watch an external condition | `Monitor` (`persistent: true` for the session) |
| Reach the user's phone | `PushNotification` (only if Remote Control connected) |

A background agent's completion arrives as a notification; you do not poll for
it. **Do not `TaskOutput` a local agent's `.output` file** — it is a symlink to
the full sub-agent transcript and will flood the supervisor's context, which is
the one thing that must survive. Use the Agent result the notification carries.

---

## Spawn

One background sub-agent per queue item. Every brief carries all seven of these
— a brief missing any one of them produces an agent that stalls or oversteps:

1. **The task**, concretely, with its definition of done.
2. **The scope boundary** — the paths and the queue item it may touch. Anything
   outside is a return-to-supervisor, not an exploration.
3. **The permission grants it may rely on**, verbatim from `afk_session.md`,
   with their scope.
4. **The denials it must not test.** Explicitly. An agent that "just tries" a
   denied command burns itself and teaches nothing not already recorded.
5. **The human-approval guard** (SKILL.md §4) — the interactive-command table
   and the standing rules.
6. **The tier rules**, with: *you may act on low-stakes decisions; anything
   medium or above, return it, do not decide it.*
7. **Return, do not ask.** Verbatim, because it is the single most important
   line in the brief:

   > If you need a decision you cannot make at your tier, **return immediately
   > with the question and everything you learned**. Do not stall waiting for an
   > answer — nobody is there. Do not decide above your tier. Do not "try
   > something and see". Returning with a question is a **successful** outcome
   > and costs the session nothing; stalling costs it hours.

Sub-agents return **decisions and rationale, not narration**. The supervisor
writes every log entry, so the log keeps one voice and lives in one file.

### Parallel or serial?

Parallel by default — that is the point of delegating. But:

- Two agents editing the same files is a self-inflicted merge conflict. Use
  `isolation: "worktree"` if they genuinely must run together, or serialize them
  if they don't.
- Two agents running the test suite against one database will produce a flake
  the supervisor then has to diagnose. Serialize anything sharing mutable state.
- Do not exceed three or four concurrent workers. The supervisor has to be able
  to actually watch them, and diagnosing four simultaneous stalls is worse than
  running them in two waves.

---

## Watch

Between notifications, the supervisor may **only**: read logs and output files,
update `afk_session.md` and the AFK log, classify tiers, and build council
briefs.

**It must not start doing the work.** The moment it picks up a task, it stops
being able to notice a stall — and the stall it misses will be the expensive
one. If the queue is genuinely empty and a worker is still running, the correct
action is to wait, not to find something to do.

### Stall signals

| Signal | What it usually means |
|---|---|
| Past its expected window, no completion notification | Prompt, infinite loop, or a much harder task than briefed |
| Output shows an interactive prompt (`[Y/n]`, `Password:`, `Press ENTER`) | Interactive command — §4 guard was missed |
| Output shows a permission denial | Ungranted permission — should have been caught in §1.3 |
| The agent asks a question instead of returning one | Brief item 7 didn't land |
| Repeated identical actions | Retry loop against something that won't succeed |
| Touching paths outside its scope | Scope boundary didn't land, or the task was under-specified |

The expected window comes from the queue item's shape recorded at departure
(SKILL.md §1.5). Absent one, treat **20 minutes of silence** on a task briefed
as short as the threshold. Err toward checking early: a false-positive stall
costs one `TaskStop`; a missed one costs hours.

---

## Cancel and retry

### Diagnose before respawning

`TaskStop` the agent, then **read its output and work out what actually
happened** before spawning anything. A retry issued without a diagnosis is the
same failure at the same cost, and you have now spent two agents learning
nothing.

> **If you cannot name what went wrong, it is a block, not a retry.**

### The retry brief

Every retry brief names the specific thing to avoid **and what to do instead**:

> Previous attempt (`queue-worker-1`) stalled at 11:05 on `npm install` waiting
> for a peer-dependency prompt.
> - Do **not** run `npm install`.
> - Use `npm ci --frozen-lockfile`.
> - If the lockfile is stale, that is a **BLOCK**, not something to resolve —
>   return with the question.
> Everything else in the original brief stands.

### Two retries maximum

Then `BLOCKED_`. Three failures is a signal about the **approach**, not the
agent — a fourth attempt is a slower way to reach the same conclusion. Write the
block with all three diagnoses; the pattern across them is usually the most
useful thing in the entire briefing.

### `SendMessage` vs. a fresh agent

- **`SendMessage`** when the agent is alive and merely needs a fact or a
  correction — it resumes with its context intact, which is cheaper and keeps
  the work it has already done.
- **Kill and respawn** when it is stuck, looping, or has gone down a wrong path.
  A confused agent's context is a liability; you want the clean start.

Never `SendMessage` an agent stalled on a permission prompt hoping to unstick
it. It has no turn in which to read you. Kill it.

---

## Worked failures

### 1. Permission prompt (the common case)

`migration-agent` goes quiet at 11:05. At 11:25 the supervisor checks: output
ends at `alembic upgrade head` with a permission request.

`alembic upgrade` **was** granted at departure — but scoped "local dev DB only",
and the agent's config pointed at staging.

- `TaskStop migration-agent`
- Diagnosis: not a missing permission — an **out-of-scope** one. That is a
  block, not a retry. The grant's scope is the authorization.
- `BLOCKED_2026-07-26T11-25.md`: what it tried, which grant it fell outside,
  the recommendation (run against dev, or grant staging explicitly).
- Log the stall. Briefing: "worth deciding whether staging migrations are ever
  in scope for AFK."

**No retry.** Retrying an out-of-scope action is attempting the same
unauthorized thing twice.

### 2. Interactive command (retry succeeds)

`queue-worker` silent 25 minutes past window. Output ends at `npm install`,
peer-dependency conflict prompt.

- `TaskStop`; diagnosis: §4 guard missed — `npm install` can prompt.
- Retry with the brief above (`npm ci --frozen-lockfile`). Succeeds in 4
  minutes.
- Log both. Briefing: **"allowlist `npm ci`; never allowlist bare `npm
  install`"** — this is the kind of finding that makes the next absence better.

### 3. Question, correctly returned (not a stall)

`auth-fix` returns after 12 minutes: "The leak is in the refresh path. Fixing it
properly changes `Session.refresh()`'s signature — that's a public interface, so
it's medium tier. Returning rather than deciding. Here's the analysis: …"

This is the system **working**. The agent recognized its tier and returned in 12
minutes instead of stalling for six hours or quietly changing a public API.

- Supervisor confirms the tier (agrees: medium).
- Routes to the council (SKILL.md §6), which returns `MEDIUM` for the additive
  overload over the signature change.
- Respawns `auth-fix` with the verdict as a decision, not a question.
- Logs the decision with the council reference.

Note the shape: **the agent never had to decide, and never had to wait.**

### 4. Wrong-path loop

`test-refactor` has been running 40 minutes; output shows the same three files
edited and reverted repeatedly.

- `TaskStop`. Diagnosis: chasing a circular import it keeps re-introducing.
- Retry with: "Previous attempt looped re-introducing a circular import between
  `conftest.py` and `fixtures/db.py`. Do not move fixtures between those two.
  If the cycle can't be broken without moving them, return — that's a
  restructure, which is medium tier."
- Second attempt returns with exactly that question in 8 minutes. Councilled.

The retry did not solve the problem; it converted an infinite loop into a
question in 8 minutes. **That is a successful retry.**

---

## What to record

Every stall goes in the log, even the recovered ones:

```markdown
### 11:05 — STALL: migration-agent
- **Symptom:** silent 20 min past window; output ends at a permission request
- **Diagnosis:** `alembic upgrade` granted for dev only; agent targeted staging
- **Action:** stopped, blocked (out-of-scope grant — not a retry candidate)
- **Retry:** none — see BLOCKED_2026-07-26T11-25.md
- **Next time:** decide at departure whether staging migrations are ever in scope
```

The `Next time:` line is the point. The stalls section of the return briefing is
where the **next** absence gets better: a stall on a missing permission is a
permission to pre-authorize next time; a stall on an interactive command is a
line for the §4 table. Say it explicitly rather than leaving the user to infer
it from a symptom.
