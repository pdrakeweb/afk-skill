# Templates

Copy these verbatim. Consistent shape is what makes the log skimmable after a
seven-hour absence and greppable afterwards.

---

## `afk_session.md`

Written at departure, updated only at return.

```markdown
# AFK session
- **Departed:** 2026-07-26 09:12
- **Expected back:** 2026-07-26 17:00
- **Comms posture:** acknowledge-only
- **Council available:** yes
- **Log:** afk_log_2026-07-26.md
- **Status:** ACTIVE

## Work queue (scope boundary — confirmed at departure)
1. Fix the session leak in auth/session.py
2. Add retry backoff to queue/worker.py
3. Triage the flaky test_reconnect

## Pre-authorized (verbatim, as the user said it)
- "yes, quarantine the flaky test and open an issue, don't chase it"
- "if CI goes red on the leak fix, just revert it, don't debug at 2am"

## Reserved (explicitly off-limits)
- "don't touch the billing code while I'm out"
```

Record pre-authorizations **verbatim**. A paraphrase drifts; the exact words
are what you check a borderline action against.

---

## `afk_log_YYYY-MM-DD.md`

One file per day. Header once, then append.

```markdown
# AFK log — 2026-07-26
Session: 09:12 → (open)

## Decisions

### 09:31 — Quarantined test_reconnect with an xfail marker
- **Tier:** low
- **Rationale:** Pre-authorized at departure ("quarantine the flaky test and
  open an issue"). Used `xfail(strict=False)` rather than `skip` so it still
  runs and we learn if it starts passing.
- **Alternatives considered:** `skip` — hides the signal entirely. Fixing it
  now — the root cause looks like a socket-timing race, not a 20-minute job.
- **Reversal:** remove the marker at tests/test_conn.py:44 — commit a91f3c2

### 10:47 — Postgres advisory locks over Redis for the job queue
- **Tier:** medium
- **Council:** convened, 1 round — see council_log_2026-07-26.md 10:44
- **Advocate:** Postgres — the enqueue joins the business transaction, so the
  lost-job failure mode stops existing.
- **Skeptic:** Connection-pool exhaustion under long jobs; nobody measured it.
- **Pragmatist:** Postgres ships this week; Redis needs infra we don't have.
- **Verdict:** Postgres advisory locks — confidence **MEDIUM**
- **Action taken:** Implemented in queue/locks.py; 6 tests; suite green.
- **Reversal:** revert commit 7f2b1e9 — self-contained, one new file plus the
  call site in worker.py:112

### 13:20 — Chose additive migration for the sessions table
- **Tier:** medium
- **⚠ UNVERIFIED — council skill not installed; single-model judgement only**
- **Case for:** Additive is reversible; the destructive version saves ~40MB,
  which is not worth an irreversible step taken while Pete is out.
- **Failure modes:** Two columns coexist until someone cleans up; a stale
  reader could write the old column.
- **Feasibility:** Same effort either way.
- **Action taken:** migration 0043, additive only, not applied to any env.
- **Reversal:** delete migrations/0043_add_session_expiry.py — never ran.
- **Needs review on return:** yes

## Blocked

### 11:40 — Push the session-leak hotfix to main
→ BLOCKED_2026-07-26T11-40.md

## Messages

### 10:15 — Dana (Slack DM)
- **Gist:** Asking whether the Q3 API timeline still holds.
- **Replied:** "Pete is AFK until ~17:00 today. I've flagged this for him."
- **Commitments made:** none

## Notes
- 12:00 — Suite green on `hotfix/session-leak`, 4 new tests.
- 15:50 — Queue empty; wrote up remaining work rather than starting anything new.
```

**Log decisions, not actions.** Running tests is an action. Choosing to
quarantine a test is a decision. If you cannot write the `Reversal:` line, the
decision is not low-stakes — reclassify it.

---

## `BLOCKED_<YYYY-MM-DDTHH-MM>.md`

One file per blocked decision. Filename timestamp so they sort chronologically.

```markdown
# BLOCKED — Push the session-leak hotfix to main
- **Time:** 2026-07-26 11:40
- **Tier:** high / irreversible
- **Trigger:** pushing to main

## What I was doing
Fixing the connection leak in `auth/session.py` (queue item 1). Root cause was
a missing `finally` around the session close on the refresh path.

## What I need decided
Push `hotfix/session-leak` to `main`, or hold it until you're back?
→ push / hold

## What I would recommend, and why
Recommend push. The fix is 6 lines, has 3 new tests covering the leak path,
and the full suite is green. Against that: it touches auth, and I can't judge
whether anything else is mid-flight on `main` right now.
*(This is a recommendation. Nothing has been pushed.)*

## What is waiting on this
Nothing hard-blocked. Queue item 2 (retry backoff) proceeded on the same
branch, so if you hold the push, both land together.

## State right now
- Branch: `hotfix/session-leak`, 2 commits ahead of `main`, pushed to origin
  (branch only — **not** merged, no PR opened)
- Working tree: clean
- CI: green on the branch as of 11:38
```

Note what the file itself proves: the branch was pushed, the merge was not.
State the boundary explicitly so the user does not have to check.

---

## Return briefing

Order is fixed: **blocked → needs review → completed → decisions → messages →
in flight.** Blocked first is the whole point.

```markdown
## Welcome back — AFK 09:12 → 16:48 (7h36m)

### ⛔ Blocked, waiting on you (2)
1. **Push the session-leak hotfix to main** — BLOCKED_2026-07-26T11-40.md
   6-line fix, 3 tests, suite green, on `hotfix/session-leak`. Recommend push;
   stopped because it touches auth. → **push / hold**
2. **AWS bill alert, $340 over budget** — BLOCKED_2026-07-26T14-02.md
   Traced to the staging cluster left running since Tuesday. Recommend scaling
   to zero. → **scale down / leave**

### ⚠ Needs your review (1)
- **Additive migration for `sessions`** — decided without the council (skill
  not installed), so this got single-model judgement only. Not applied to any
  environment; reversible by deleting one file. → afk_log 13:20

### ✅ Completed (4)
- Session leak fixed in `auth/session.py:88` — 3 tests added, suite green
- Retry backoff added to `queue/worker.py` — exponential, capped at 30s
- `test_reconnect` quarantined + issue #241 opened (pre-authorized)
- Job-queue locking implemented on Postgres advisory locks

### 📋 Decisions (7 — 5 low, 2 medium, 1 unverified)
Rationale and reversal steps for each: afk_log_2026-07-26.md

### 💬 Messages (1)
- **Dana, 10:15** — Q3 API timeline. Acknowledged only, **no commitment made**.

### 🚧 In flight
Nothing half-finished. Queue emptied at 15:50; I stopped rather than starting
anything new.

---
The council skill wasn't installed, so one medium decision went unverified.
Installing it would route those through an adversarial review:
https://github.com/pdrakeweb/council-skill
```

Then **stop and wait.** Do not act on a blocked item until the user answers it.
Their being back is not an approval.
