# Templates

Copy these verbatim. A consistent shape is what makes the record skimmable after
a seven-hour absence and greppable afterwards.

---

## `afk_session.md`

Written at departure, updated only at return (and by the watchdog on a
Remote Control state change).

```markdown
# AFK session
- **Departed:** 2026-07-26 09:12
- **Expected back:** 2026-07-26 17:00
- **Remote Control:** connected  (→ lost 12:40, conservative mode from then)
- **Mode:** normal | conservative
- **Comms posture:** acknowledge-only
- **Council available:** yes
- **Watchdog:** armed (agent `afk-watchdog`)
- **Log:** afk_log_2026-07-26.md
- **Status:** ACTIVE

## Work queue (scope boundary — confirmed at departure)
1. Fix the session leak in auth/session.py         — expect ~30 min
2. Add retry backoff to queue/worker.py            — expect ~45 min
3. Triage the flaky test_reconnect                 — expect ~20 min

## Permission grants (verbatim, with scope)
- GRANTED: `pytest`, `pytest -k` — any
- GRANTED: `npm ci` — repo root only
- GRANTED: `alembic upgrade head` — **local dev DB only, NOT staging**
- GRANTED: WebSearch / WebFetch — council research phase
- DENIED:  anything touching `deploy/` — "not while I'm out"
- DENIED:  `docker push` — no response at departure, treat as denied

## Expertise answers (from the departure batch)
- "legacy_email — yes, the billing exporter reads it. Leave it."
- "30s retry cap is a guess, tune it freely."
- "Dana's timeline question — I'll answer that myself, just acknowledge."

## Pre-authorized (verbatim, as the user said it)
- "yes, quarantine the flaky test and open an issue, don't chase it"
- "if CI goes red on the leak fix, just revert it, don't debug at 2am"

## Reserved (explicitly off-limits)
- "don't touch the billing code while I'm out"
```

Record grants, denials, expertise answers, and pre-authorizations **verbatim**.
A paraphrase drifts; the exact words are what a borderline action gets checked
against six hours later. **An unanswered permission request is a denial** —
record it as one.

---

## Departure batches

### Expertise questions (§1.2)

```markdown
**Need your brain before you go — 3 things:**
1. `legacy_email` — is anything outside this repo reading it? *(default: assume yes, keep it)*
2. The 30s retry cap — real SLA or a guess? *(default: treat as guess, tune freely)*
3. Dana's timeline question — answer it yourself? *(default: yes, I'll only acknowledge)*

Reply with numbers to change, or "defaults".
```

Only things **the user knows and I cannot derive**. If I could research it,
council it, or read it out of the repo, it does not belong here.

### Permission sweep (§1.3)

```markdown
**Permissions I'll need — approve now or I'll stall on them:**
- `pytest`, `pytest -k` — running the suite
- `npm ci` — installing deps (never bare `npm install`, it prompts)
- `alembic upgrade head` — **local dev DB only**
- WebSearch / WebFetch — the council's research phase
- Write to `~/.cache/pip` — build cache

Anything here you'd rather I not have, say so now and I'll route that work to
BLOCKED rather than discovering it at hour three.
```

Name the **specific** commands, not "shell access". Say what each is for. Flag
the scope on anything scoped.

---

## `afk_log_YYYY-MM-DD.md`

One file per day. Header once, then append.

````markdown
# AFK log — 2026-07-26
Session: 09:12 → (open) · Remote Control: connected · Council: available

## Decisions

### 09:31 — Quarantined test_reconnect with an xfail marker
- **Tier:** low
- **Agent:** flaky-triage
- **Rationale:** Pre-authorized at departure ("quarantine the flaky test and
  open an issue"). Used `xfail(strict=False)` not `skip`, so it still runs and
  we learn if it starts passing.
- **Alternatives considered:** `skip` — hides the signal. Fixing it now — root
  cause looks like a socket-timing race, not a 20-minute job.
- **Reversal:** remove the marker at tests/test_conn.py:44 — commit a91f3c2

### 10:47 — Postgres advisory locks over Redis for the job queue
- **Tier:** medium (routed from a sub-agent question)
- **Agent:** queue-worker (returned the question); supervisor councilled it
- **Council:** panel classic · rounds 2 · research ran (5 findings)
  → council_log_2026-07-26.md 10:44
- **Advocate:** Postgres — enqueue joins the business transaction, so the
  lost-job failure mode stops existing.
- **Skeptic:** connection-pool exhaustion under long jobs; unmeasured.
- **Pragmatist:** Postgres ships this week; Redis needs infra we don't have.
- **Round 2:** Skeptic conceded the pool concern is bounded at current volume.
- **Verdict:** Postgres advisory locks — confidence **MEDIUM**
- **COUNCIL_NEEDS:** actual p99 job duration under load — carried to briefing
- **Action taken:** implemented in queue/locks.py; 6 tests; suite green
- **Reversal:** revert 7f2b1e9 — one new file plus the call site at worker.py:112

### 13:20 — Additive migration for the sessions table
- **Tier:** medium
- **⚠ UNVERIFIED — council skill not installed; single-model judgement only**
- **Case for:** additive is reversible; the destructive version saves ~40MB,
  not worth an irreversible step taken while Pete is out.
- **Failure modes:** two columns coexist until cleanup; a stale reader could
  write the old column.
- **Feasibility:** same effort either way.
- **Action taken:** migration 0043, additive only, **not applied to any env**.
- **Reversal:** delete migrations/0043_add_session_expiry.py — never ran.
- **Needs review on return:** yes

## Stalls and retries

### 11:05 — STALL: migration-agent
- **Symptom:** silent 20 min past window; output ends at a permission request
- **Diagnosis:** `alembic upgrade` granted for dev only; agent targeted staging
- **Action:** stopped; blocked (out-of-scope grant — not a retry candidate)
- **Retry:** none — see BLOCKED_2026-07-26T11-25.md
- **Next time:** decide at departure whether staging migrations are ever in scope

### 11:30 — STALL: queue-worker (recovered)
- **Symptom:** silent 25 min past window; output ends at an `npm install` prompt
- **Diagnosis:** peer-dependency conflict prompt — §4 guard missed
- **Action:** stopped; retried with `npm ci --frozen-lockfile`
- **Retry:** 1 of 2 — succeeded in 4 min
- **Next time:** allowlist `npm ci`; never allowlist bare `npm install`

## Blocked

### 11:40 — Push the session-leak hotfix to main
→ BLOCKED_2026-07-26T11-40.md

## Infrastructure

### 12:40 — Remote Control lost
- Two consecutive PushNotification checks returned not-sent.
- Switched to **conservative mode**: medium-tier decisions that aren't trivially
  reversible now block.
- Cannot re-enable — `/remote-control` is a UI command. Queued for the briefing.

## Messages

### 10:15 — Dana (Slack DM)
- **Gist:** does the Q3 API timeline still hold?
- **Replied:** "Pete is AFK until ~17:00 today. I've flagged this for him."
- **Commitments made:** none

## Notes
- 12:00 — suite green on `hotfix/session-leak`, 4 new tests
- 15:50 — queue empty; wrote up remaining work rather than starting anything new
````

**Log decisions, not actions.** Running tests is an action; choosing to
quarantine a test is a decision. If you cannot write the `Reversal:` line, the
decision is not low-stakes — reclassify it.

---

## `BLOCKED_<YYYY-MM-DDTHH-MM>.md`

One per blocked decision. Timestamped filename so they sort chronologically.

```markdown
# BLOCKED — Push the session-leak hotfix to main
- **Time:** 2026-07-26 11:40
- **Tier:** high / irreversible
- **Trigger:** pushing to main

## What I was doing
Fixing the connection leak in `auth/session.py` (queue item 1). Root cause was
a missing `finally` around the session close on the refresh path.

## What I need decided
Push `hotfix/session-leak` to `main`, or hold until you're back?
→ **push / hold**

## Council said
Convened at 11:36 (panel classic, 2 rounds). Verdict: push — confidence **HIGH**.
Skeptic's only objection (auth blast radius) was conceded in round 2 given the
test coverage.
*This is a recommendation, not authorization. High tier is blocked regardless of
verdict — the council rules on what is wise, only you authorize.*

## What I would recommend, and why
Push. 6-line fix, 3 new tests covering the leak path, full suite green. Against
it: it touches auth, and I can't see what else is mid-flight on `main`.

## What is waiting on this
Nothing hard-blocked. Queue item 2 (retry backoff) proceeded on the same branch,
so holding the push lands both together.

## State right now
- Branch: `hotfix/session-leak`, 2 commits ahead of `main`, **pushed to origin
  as a branch only — not merged, no PR opened**
- Working tree: clean
- CI: green on the branch as of 11:38
```

Note what that last section proves: the branch was pushed, the merge was not.
State the boundary explicitly so the user doesn't have to go check — and never
stage the blocked action down to one keystroke.

---

## Sub-agent spawn brief

All seven elements from `delegation.md`. Item 7 goes in verbatim.

```markdown
TASK: Add exponential retry backoff to queue/worker.py, capped at 30s.
DONE WHEN: backoff implemented, tests cover the cap and the first three retries,
full suite green.

SCOPE: queue/worker.py, queue/tests/. Nothing else. Anything outside this →
return to supervisor, do not explore.

PERMISSIONS YOU HAVE (verbatim, with scope):
- `pytest`, `pytest -k` — any
- `npm ci` — repo root only

PERMISSIONS EXPLICITLY DENIED — do not test these:
- anything under `deploy/`
- `docker push`
- bare `npm install` (it prompts; use `npm ci`)

HUMAN-APPROVAL GUARD: never run an interactive command. Use non-interactive
flags always (`--yes`, `-y`, `--non-interactive`, `--no-input`). Never disable a
safety check to avoid a prompt (`--no-verify`, skipping tests, silencing a
linter). If the only way forward needs a human, return — do not attempt it.

CONTEXT: Pete said the 30s cap is a guess, tune freely. Don't touch billing code.

TIER RULES: act on low-stakes decisions (reversible, local, no external side
effect) and report them with rationale. Anything medium or above — architecture,
new files, public interfaces, dependencies, anything leaving this machine —
RETURN IT, do not decide it.

RETURN, DO NOT ASK: If you need a decision you cannot make at your tier, return
immediately with the question and everything you learned. Do not stall waiting
for an answer — nobody is there. Do not decide above your tier. Do not "try
something and see". Returning with a question is a SUCCESSFUL outcome and costs
the session nothing; stalling costs it hours.

REPORT: decisions and rationale, not narration. The supervisor writes the log.
```

---

## Return briefing

Order is fixed: **do-this-first → blocked → open questions → review → completed
→ decisions → stalls → messages → in flight.** Anything needing action is above
anything that is merely news.

```markdown
## Welcome back — AFK 09:12 → 16:48 (7h36m)

### 🔴 Do this first
- **Re-run `/remote-control`** — dropped at 12:40; nothing could reach your
  phone for 4h08m. I tightened to conservative mode from then, so two medium
  decisions were blocked that would otherwise have proceeded.

### ⛔ Blocked, waiting on you (2)
1. **Push the session-leak hotfix to main** — BLOCKED_2026-07-26T11-40.md
   Tested, committed on `hotfix/session-leak`. Council: push (HIGH). Blocked
   because high tier is blocked regardless. → **push / hold**
2. **AWS bill alert, $340 over** — BLOCKED_2026-07-26T14-02.md
   Staging cluster running since Tuesday. Recommend scaling to zero.
   → **scale down / leave**

### ❓ Questions the council couldn't answer (2)
- Is `legacy_email` read outside this repo? *(blocked migration 0043)*
- Actual p99 job duration under load? *(council: MEDIUM — would have been HIGH)*

### ⚠ Needs your review (1)
- **Additive migration for `sessions`** — decided without the council (not
  installed), single-model judgement only. Not applied anywhere; reversible by
  deleting one file. → afk_log 13:20

### ✅ Completed (4)
- Session leak fixed in `auth/session.py:88` — 3 tests, suite green
- Retry backoff in `queue/worker.py` — exponential, capped at 30s
- `test_reconnect` quarantined + issue #241 opened (pre-authorized)
- Job-queue locking on Postgres advisory locks

### 📋 Decisions (9 — 6 low, 2 councilled, 1 unverified)
Rationale and reversal for each: afk_log_2026-07-26.md

### 🔁 Stalls and retries (2)
- `queue-worker` stalled on an `npm install` peer-dep prompt (11:05); retried
  with `npm ci` — succeeded. **Worth allowlisting `npm ci` next time.**
- `migration-agent` targeted staging under a dev-only grant → blocked, no retry.
  **Worth deciding whether staging is ever in scope for AFK.**

### 💬 Messages (1)
- **Dana, 10:15** — Q3 API timeline. Acknowledged only, **no commitment made**.

### 🚧 In flight
Nothing half-finished. Queue emptied at 15:50; I stopped rather than starting
anything new.

---
The council skill wasn't installed for the first two hours, so one medium
decision went unverified: https://github.com/pdrakeweb/council-skill
```

Then **stop and wait.** Do not act on a blocked item until they answer it. Their
being back is not an approval.
