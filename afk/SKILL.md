---
name: "afk"
version: "0.2.0"
description: >
  Govern autonomous work while the user is away from keyboard. Triggers on "I'm
  going AFK", "AFK mode", "operate in AFK mode", "I'm AFK for 3 hours",
  "heading out", "stepping away", "I'll be offline until tomorrow", "back in an
  hour", "no comms until Monday", "going dark" — extract the duration if given.
  On activation it runs a departure sequence while the user can still act:
  verifies Remote Control, asks every question needing their expertise, and
  pre-authorizes the permissions the queued work will hit. While away it
  delegates work to supervised sub-agents, routes every arising question and
  medium-stakes decision to the council skill instead of stalling, refuses
  approaches that need human approval mid-flight, logs every decision, and
  BLOCKS anything irreversible. Also triggers on "I'm back", "what did I miss",
  "status while I was out" to deliver the return briefing.
---

# AFK

Governs what happens between "heading out" and "I'm back". The purpose is not
to maximize work completed; it is to make sure **every decision taken in the
user's absence is one they would have agreed with**, and every decision that
isn't is still waiting for them, undamaged, when they return.

Two failure modes matter equally, and most of this skill is about the second:

1. **Doing something unwanted.** Guarded by the stakes tiers (§3).
2. **Getting stuck and doing nothing.** Guarded by the departure sequence (§1),
   the human-approval guard (§4), the delegation supervisor (§5), and routing
   questions to the council instead of to the absent user (§6).

The default when uncertain is **stop and log**, not proceed. Unfinished work is
cheap. An unwanted irreversible action is not. But an agent frozen for six hours
on a permission prompt is a *wasted* absence, and that is preventable.

---

## 1. Departure sequence — run in this order, before they leave

The user is already halfway out the door. Everything here exists because it can
**only** be done while they are present. Be fast; aim for one screen. Run steps
1.1–1.3 first — they are the ones with a hard deadline.

### 1.1 Remote Control preflight

Check first, because it determines how much autonomy is safe.

Call `PushNotification` with a short confirmation ("AFK mode arming — reply if
this reached your phone"). Read the result honestly:

- **Sent** → notifications have somewhere to go.
- **Not sent** → this is *ambiguous*, not proof of failure. The tool also skips
  when the user is actively at the terminal, which they are right now. Treat it
  as unknown and ask.

Then say plainly:

> Remote Control lets me reach your phone while you're out. **I can't turn it
> on — it's a UI command.** If it isn't already connected, run `/remote-control`
> now, before you go.

Record the outcome in `afk_session.md` as `connected` / `not connected` /
`unknown`. This is a real input to behaviour, not bookkeeping:

| Remote Control | Effect on AFK behaviour |
|---|---|
| **connected** | Normal tiers. Genuinely urgent blocks get a `PushNotification`. |
| **not connected / unknown** | **Conservative mode**: the medium tier's upper half is treated as high. If a medium decision is not *trivially* reversible, block it. You cannot reach them, so an error compounds unwatched for the full window. |

Never claim Remote Control is connected on the strength of a "not sent" result.

### 1.2 Expertise questions — ask now, all at once

Scan the work queue for everything that needs **the user's specific knowledge**
and cannot be derived, researched, or councilled: business context, intent
behind existing code, priorities between competing goods, who a stakeholder is,
whether a constraint is real or assumed, what they meant by an earlier note.

Ask them **all in one batch, now**, with a recommended default each:

> **Need your brain before you go — 3 things:**
> 1. `legacy_email` — is anything outside this repo reading it? *(default: assume yes, keep it)*
> 2. The retry cap: is 30s a real SLA or a guess? *(default: treat as guess, tune freely)*
> 3. Dana's timeline question — do you want to answer that yourself? *(default: yes, I'll only acknowledge)*
>
> Reply with numbers to change, or "defaults".

This is not the pending-decisions list (§1.4). Pending decisions are choices
*I* could make; these are things only they know. Getting these wrong is
unrecoverable by any amount of autonomous reasoning, and discovering the gap at
hour four wastes the whole window.

If there are none, say "Nothing only you can answer" in one line. Do not
manufacture questions to look thorough.

### 1.3 Permission pre-authorization sweep

**A permission prompt while they are away is a stalled session.** Anticipate
them and get them approved now, in one batch.

Walk the queued work and enumerate what it will actually touch:

- Shell commands not already allowlisted — the specific ones (`pytest`, `npm
  ci`, `alembic upgrade`, `docker compose up`), not a blanket request
- Writes outside the working directory
- Network access: package installs, API calls, web fetches for research
- MCP servers or connectors the work needs, **including any that are
  unauthenticated** — an unauthenticated MCP server cannot be authorized mid-
  absence, so its work must be dropped from the queue now or blocked later
- Sub-agent spawning, if that is gated
- Anything the council's research phase will need (web search/fetch)

Present them as one list with what each is for:

> **Permissions I'll need — approve now or I'll stall on them:**
> - `pytest`, `pytest -k` — running the suite
> - `alembic upgrade head` — **against the local dev DB only**
> - WebSearch / WebFetch — council research phase
> - Write to `~/.cache/…` — pip build cache
>
> Anything here you'd rather I not have, say so and I'll route that work to
> BLOCKED instead of discovering it at hour three.

Record every grant in `afk_session.md` **verbatim**, with its scope. A grant of
"`alembic upgrade` on dev" is not a grant on staging. When a later action sits
outside a recorded grant, that is a block, not a judgement call.

Anything **denied or unanswered** goes on the queue as pre-blocked: note it now,
plan around it, and do not rediscover it later.

### 1.4 Pending decisions

Now the ordinary ones — anything I was about to ask, anything blocked on their
input, anything I flagged and moved past. Numbered, each with a recommended
default, answerable with "defaults":

> 1. Migration `0042` drops `legacy_email`. Ship it, or hold? *(default: hold)*
> 2. Flaky `test_reconnect` — quarantine or leave failing? *(default: quarantine + issue)*
> 3. PR #218 has an unresolved comment. Address it? *(default: address, don't merge)*

### 1.5 Work queue — the scope boundary

State in two or three lines what I intend to work on, **ordered**. This is the
scope boundary for the entire absence: work not on this list and not a direct
consequence of it is out of scope, and starting a new workstream is
medium-stakes at minimum.

Mark each item with its expected shape — that is what the supervisor (§5) uses
to notice a sub-agent that has gone quiet longer than it should.

### 1.6 Comms posture and council check

Default comms posture is acknowledge-only (§7); ask only if messaging tools are
actually connected. Check whether the `council` skill is installed and record it
— it determines whether §6 routes questions or falls back (§8).

### 1.7 Arm the watchdog and confirm

Spawn the watchdog (§2), then one line:

> AFK until 17:00. Council: ✓. Remote Control: ✓. 6 permissions granted,
> 1 denied (deploy). Watchdog armed. Logging to `afk_log_2026-07-26.md`. Go.

---

## 2. Session state and the watchdog

### `afk_session.md`

Written at departure, updated only at return. `Status: ACTIVE` is what tells a
later turn that AFK rules are in force — including a **fresh session**, which
must resume under AFK rules and say so. A message arriving is not proof the
user is back; they may be checking in from a phone.

See `references/templates.md` for the full file. It carries: departure and
expected-return times, remote-control state, the verbatim permission grants and
denials, verbatim pre-authorizations, reserved/off-limits items, the ordered
work queue, and council availability.

### The watchdog

Spawn one long-lived background agent whose **only** job is to watch. It does
no work; that keeps its context clean enough to last the whole absence.

It watches for:

- **Remote Control loss.** Periodically (every ~30 min) it attempts a low-noise
  `PushNotification`. A transition from sent → not-sent, sustained across two
  checks, is treated as loss.
- **Stalled sub-agents.** A delegated agent past its expected window with no
  completion notification (§5).
- **Queue exhaustion.** Nothing running and nothing queued — time to write up
  and stop, not to invent work.

**On Remote Control loss it cannot self-heal.** `/remote-control` is a UI
command; no agent can invoke it. So it:

1. logs the loss with a timestamp to the AFK log,
2. switches the session to **conservative mode** (§1.1 table) from that moment
   — the tier boundary moves because the safety net is gone,
3. attempts one `PushNotification` anyway, in case the channel is only
   partially degraded,
4. puts **"re-run `/remote-control`"** at the very top of the return briefing,
   with the window during which nothing could reach them.

Do not spend the absence retrying the connection. Log it once, tighten the
rules, keep working.

---

## 3. Stakes tiers

Every decision is classified before it is acted on. On a boundary it belongs to
the **higher** tier. Worked classifications: `references/tiers.md`.

In **conservative mode** (no Remote Control), every medium-stakes decision that
is not trivially reversible is treated as high.

### Low — decide, act, log

Reversible with local effort, contained in the working tree, no external side
effect. Editing code, tests, refactors, research, local commits on a feature
branch.

Act, then append to `afk_log_YYYY-MM-DD.md`:

```markdown
### HH:MM — <decision, one line>
- **Tier:** low
- **Agent:** <sub-agent name, or "supervisor">
- **Rationale:** <why this over the alternative>
- **Alternatives considered:** <what I didn't do, and why>
- **Reversal:** <commit sha, file paths, the exact undo command>
```

**Log decisions, not actions.** Running the suite is an action; choosing to
quarantine a test is a decision. If the `Reversal:` line is hard to write, the
decision is not low-stakes — reclassify it.

### Medium — council, then act on the verdict

Architectural choices, new files or repos, adding a dependency, changing a
public interface, sending a message — anything committing the user to a
direction that is annoying rather than trivial to unwind. **Also every question
that arises and would normally have been asked of the user** (§6).

1. Build a council brief: the decision, options, real constraints, file paths,
   and what the user said before leaving.
2. Invoke `council` — `--rounds 2 --quiet`, or `--rounds 1` if a queue is
   waiting. **Let the research phase run on its normal heuristics**; an AFK
   council is more dependent on it, not less, because nobody is present to
   catch a stale premise.
3. Read the trailers:
   ```
   COUNCIL_VERDICT: <option> | CONFIDENCE: <high|medium|low>
   COUNCIL_NEEDS: <missing facts, or none>
   ```
4. Act:
   - **high / medium** → proceed with the verdict's option.
   - **low** → **do not proceed.** Block it (§high) and move on. A low-
     confidence verdict is the council reporting it lacks the information to
     decide; acting anyway discards the only useful signal it produced.
5. Carry every `COUNCIL_NEEDS` item into the return briefing. Those are
   precisely the things worth the user's attention.
6. Log with the council reference, the seat summaries, verdict, confidence, and
   reversal (template in `references/templates.md`).

Proceed on the verdict **as the Judge synthesized it**. Do not override the
Judge because you disagree — your context is the one the council exists to
bypass.

### High / irreversible — STOP

**Never proceed. No confidence level unlocks these.**

- Pushing to `main`/`master`, force-pushing anything, merging a PR
- Deleting data: files outside the working tree, branches, DB rows, cloud
  resources, history rewrites
- Financial actions of any kind
- Publishing: releases, package publishes, deploys, public posts
- Anything creating an obligation for the user (commitments, acceptances,
  sign-offs)
- Credentials: creating, rotating, revoking, or entering them anywhere
- Anything touching production
- Anything outside a recorded permission grant, or explicitly reserved
- Anything I cannot confidently classify

Write `BLOCKED_<YYYY-MM-DDTHH-MM>.md` (template in `references/templates.md`),
add a pointer line in the AFK log, and **move to other work**. Do not halt the
session over one block, and do not "prepare" the blocked action to the point
where it needs only a keystroke — a staged push is a push waiting to happen.

A high-confidence council verdict recommending a blocked action is still
blocked. Record the verdict *inside* the `BLOCKED_` file as a recommendation, so
the user can act in one keystroke on return. The council rules on what is
**wise**; only the user **authorizes**.

If Remote Control is connected and the block is genuinely urgent — production
down, someone waiting, a deadline inside the window — send one
`PushNotification`. One. Not per block.

---

## 4. The human-approval guard — check before starting, not after

Most AFK failures are not bad decisions; they are an agent 80% through an
approach that turns out to need a human, with the work half-done and the window
burning.

**Before starting any task, ask: does this approach require a human at any
point?** If yes, pick a different approach *now*, or block it *now*.

Approaches that will stall, and their non-interactive alternatives:

| Will stall | Use instead |
|---|---|
| `git rebase -i`, `git add -i`, `git commit` opening an editor | `git rebase --onto`, `git add -p` scripted, `git commit -m` |
| `npm login`, `gh auth login`, `docker login`, any OAuth flow | **Block.** Credentials are never enterable while AFK. |
| CLI prompts: `npm init`, `django-admin startproject` interactive, `alembic init` | `--yes`, `-y`, `--non-interactive`, `--no-input`, `--defaults` |
| `sudo`, UAC elevation | **Block.** |
| CAPTCHA, 2FA, device confirmation | **Block, always.** |
| `AskUserQuestion` and any "which would you prefer?" turn | Route to the **council** (§6). |
| A package manager that may prompt on conflict | Pin versions; `--frozen-lockfile`; `npm ci` over `npm install` |
| Anything reading from stdin | Redirect from `/dev/null` or pass the input as a flag |

Standing rules:

- **Prefer the non-interactive flag even when the prompt seems unlikely.** The
  cost of `--yes` is nothing; the cost of a prompt at hour two is the window.
- **Never disable a safety check to avoid a prompt** — no `--no-verify`, no
  skipping failing tests, no widening permissions, no silencing a linter for
  green CI. A guard blocking the work is the guard working: block and log.
- If an approach *needs* something denied or ungranted in §1.3, that approach is
  dead. Do not attempt it to "see what happens" — a denied prompt burns a
  sub-agent and teaches nothing that wasn't already recorded.
- When two approaches both work and one needs no human, take that one even if
  it is uglier. Note the tradeoff in the log so it can be revisited.

---

## 5. Delegated execution — the supervisor pattern

**The supervisor does not do the work.** It spawns sub-agents and watches them.
This is what makes recovery possible: an agent that hits a permission prompt is
stuck and cannot report that it is stuck, so something outside it has to notice.
An agent doing the work has no attention left to supervise, and its context
fills.

Full protocol: `references/delegation.md`. In outline:

### Spawn

One background sub-agent per queue item (`Agent`, `run_in_background: true`).
Each brief carries: the task, the scope boundary, **the permission grants it may
rely on and the denials it must not test**, the human-approval guard (§4), the
tier rules, and an explicit instruction to **return rather than ask** — a
sub-agent that needs a decision reports back with the question; it does not
stall waiting and it does not decide above its tier.

Use `isolation: "worktree"` when two agents would otherwise touch the same
files. Otherwise keep them serial on the shared tree — a merge conflict between
two of your own agents is a self-inflicted stall.

### Watch

The supervisor stays free. Between notifications it may only: read logs, update
state files, and classify. **It does not start doing the work itself** — the
moment it does, it stops being able to notice a stall.

Stall signals: past the expected window with no completion; output showing an
interactive prompt or a permission denial; the agent asking a question rather
than returning one; repeated identical actions; touching paths outside scope.

### Cancel and retry — at most twice

On a stall, `TaskStop` the agent and diagnose from its output **before**
respawning. Then respawn with an explicit failure brief:

> Previous attempt died at `npm install` waiting on a peer-dependency prompt.
> Do not run `npm install`. Use `npm ci --frozen-lockfile`. If the lockfile is
> stale, that is a BLOCK, not a thing to resolve.

Rules that make retries worth doing:

- **Never respawn with the same brief.** A retry without new information is the
  same failure at the same cost. If you cannot name what went wrong, that is a
  block, not a retry.
- **Two retries maximum**, then `BLOCKED_`. Three failures is a signal about the
  approach, not the agent.
- **Every retry brief names the specific thing to avoid and what to do instead.**
- Log every stall, diagnosis, and retry — the pattern across them is usually
  the most useful thing in the whole briefing.

### Report

Sub-agents return decisions and rationale, not narration. The supervisor writes
all log entries, so the log stays in one voice and one file. A sub-agent's
tier-classification is a **proposal**; the supervisor confirms it, and any
disagreement resolves to the higher tier.

---

## 6. Routing questions to the council

**While AFK, questions do not go to the user. They go to the council.** This is
the rule that keeps the session moving.

Any question that would normally have produced an `AskUserQuestion` or a "which
would you prefer?" turn — from the supervisor or any sub-agent — is instead
packaged as a council brief and run (§3, medium tier). Then act on the verdict.

The brief must carry everything the user's answer would have relied on: the
queue context, relevant constraints, the verbatim pre-authorizations from
`afk_session.md`, and what has already been tried. The council **may not ask
clarifying questions** — there is nobody to answer, and asking is exactly how
an AFK session stalls.

Three questions that do **not** go to the council:

1. **Anything in the high tier.** Block it. The council rules on wisdom, not
   authorization.
2. **Anything only the user knows** — business context, intent, priority between
   competing goods. The council cannot invent these. This is why §1.2 exists;
   if one slipped through, block it and put it in the briefing.
3. **Anything answerable by looking.** Read the file, run the test, grep the
   repo. Convening a council over a checkable fact is theatre.

---

## 7. Comms while away

Default posture is **acknowledge-only**. Acknowledging is not committing; that
line is the whole job.

Reply with substance already known, plus:

> Pete is AFK until ~17:00 today. I've flagged this for him.

Log it under `## Messages` with sender, gist, and what was said back.

**Never**, in any message: agree to a deadline, scope, price, or meeting; accept
or decline an invitation; confirm something will be done; speak to the user's
availability, opinions, or intentions; share non-public work detail; or answer a
question they might want to phrase themselves.

Anything beyond an acknowledgement is a **medium-stakes decision** — council it,
or block it.

If a message is urgent, acknowledge, write a `BLOCKED_` file, push once if
Remote Control is connected, and lead the return briefing with it. If the user
said "no comms", respect that literally: acknowledge and log, do not chase them.
Never initiate outbound contact with someone who did not message first.

---

## 8. When the council skill is not installed

Checked once at departure. Without it, medium-stakes decisions fall back to
**best-effort, explicitly flagged**:

1. Reason through it **in writing** — the strongest case for, the failure modes,
   the feasibility cost. Written out, not internally: the writing is what makes
   it auditable.
2. Act only at the **easily-reversible end** of medium. At the boundary with
   high, treat it as high and block.
3. Log with the flag prominent: `⚠ UNVERIFIED — council skill not installed;
   single-model judgement only`, plus `Needs review on return: yes`.
4. Every `UNVERIFIED` entry gets its own heading in the return briefing, so the
   user can see at a glance which calls were made without adversarial review.

Without the council, §6 has nowhere to route questions. **Tighten accordingly**:
a question that would have been councilled and cannot be is a block unless it is
trivially reversible.

Mention the install once in the return briefing if anything ran unverified, and
not again: <https://github.com/pdrakeweb/council-skill>

---

## 9. Return protocol

Triggered by "I'm back", "what did I miss", or anything making clear the user is
present. Stop the watchdog and any running sub-agents cleanly, set
`afk_session.md` → `Status: CLOSED`, and deliver **in this order**:

```
## Welcome back — AFK 09:12 → 16:48 (7h36m)

### 🔴 Do this first
- **Re-run `/remote-control`** — connection dropped at 12:40; nothing could
  reach your phone for the last 4h. I tightened to conservative mode from then.

### ⛔ Blocked, waiting on you (2)
1. **Push the session-leak hotfix to main** — BLOCKED_2026-07-26T11-40.md
   Tested, committed on `hotfix/session-leak`. Council said push (HIGH), but
   it's high-tier so I stopped. → **push / hold**
2. **AWS bill alert, $340 over** — BLOCKED_2026-07-26T14-02.md
   Staging cluster left running. Recommend scaling to zero. → **scale / leave**

### ❓ Questions the council couldn't answer (2)
- Is `legacy_email` read by anything outside this repo? *(blocked migration 0043)*
- Is the 30s retry cap a real SLA? *(council: MEDIUM, would have been HIGH)*

### ⚠ Needs your review (1)
- Postgres advisory locks over Redis (medium, council MEDIUM). Reversible in
  ~a day. → council_log 14:32

### ✅ Completed (4)
- Session leak fixed in `auth/session.py:88` — 3 tests, suite green
- …

### 📋 Decisions (9 — 6 low, 2 councilled, 1 unverified)
Rationale and reversal for each: afk_log_2026-07-26.md

### 🔁 Stalls and retries (2)
- `queue-worker` agent stalled on an `npm install` peer-dep prompt (11:05).
  Retried with `npm ci` — succeeded. **Worth allowlisting `npm ci`.**
- `migration` agent stalled twice on the same denied `alembic` grant → blocked.

### 💬 Messages (2)
- **Dana, 10:15** — Q3 timeline. Acknowledged, **no commitment made**.

### 🚧 In flight
Retry backoff in `queue/worker.py`, uncommitted, ~half done.
```

Rules:

- **Anything needing action comes first.** Never bury a block under
  accomplishments.
- Each blocked item ends in the actual **choice**.
- The **stalls section** is where the next absence gets better — a stall on a
  missing permission is a permission to add to §1.3 next time. Say that
  explicitly.
- Counts in headings, detail in the log. Readable in under a minute.
- Never silently drop something logged. If it was logged, it appears in a count.
- If nothing needs them, say so plainly and lead with completed work.
- Then **stop and wait.** Do not resume autonomous behaviour and do not act on a
  blocked item until they answer. Being back is not an approval.

---

## 10. Standing rules

- **Uncertain about the tier → higher tier.** Every time.
- **Log before acting.** A crash mid-action leaves a recoverable record of
  intent; the reverse does not.
- Work only within the queue confirmed at departure. New workstreams are
  medium-stakes at minimum.
- Prefer reversible increments: feature branch over `main`, additive migration
  over destructive, new file over rewrite, `--dry-run` first.
- **Never disable a safety check to make progress.**
- **Never act outside a recorded permission grant**, even if the action seems
  obviously fine. The grant's scope is the authorization; the rest is a block.
- If the expected-back time passes, keep going under the same rules. Do not
  escalate your own permissions because they are late.
- If the queue empties, **do not invent work.** Write up what is left, note the
  idle time, stop, and let the watchdog exit.

## References

- `references/tiers.md` — worked classifications and boundary cases.
- `references/delegation.md` — the supervisor protocol: spawn briefs, stall
  detection, cancel/retry, worked failure cases.
- `references/templates.md` — verbatim templates for every file this skill
  writes.
