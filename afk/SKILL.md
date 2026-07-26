---
name: "afk"
version: "0.1.0"
description: >
  Govern autonomous work while the user is away from keyboard. Triggers on
  "I'm going AFK", "I'm AFK for 3 hours", "heading out", "stepping away",
  "I'll be offline until tomorrow morning", "back in an hour", "no comms until
  Monday", "going dark", "you're on your own for a bit" — extract the duration
  or return time if one is given, otherwise ask once before they leave. Runs a
  pre-departure checklist to clear pending decisions, then applies stakes tiers
  while away: low-stakes acted on and logged, medium-stakes debated via the
  council skill, high-stakes and irreversible actions BLOCKED and deferred.
  Also triggers on "I'm back", "I'm returning", "what did I miss", "status
  while I was out" to run the return briefing — even if they never said AFK.
---

# AFK

Governs what happens between "heading out" and "I'm back". The purpose is not
to maximize work completed; it is to make sure **every decision taken in the
user's absence is one they would have agreed with**, and every decision that
isn't is still waiting for them, undamaged, when they return.

The default when uncertain is **stop and log**, not proceed. Unfinished work is
cheap. An unwanted irreversible action is not.

## Session state

On activation, write `afk_session.md` in the working directory:

```markdown
# AFK session
- **Departed:** YYYY-MM-DD HH:MM
- **Expected back:** YYYY-MM-DD HH:MM  (or "unknown")
- **Comms posture:** acknowledge-only  (or per user instruction)
- **Council available:** yes | no
- **Log:** afk_log_YYYY-MM-DD.md
- **Status:** ACTIVE
```

`Status: ACTIVE` is what tells a later turn that AFK rules are in force. Set it
to `CLOSED` only during the return protocol. If a fresh session finds an
`ACTIVE` file, resume under AFK rules and say so — do not assume the user is
back just because they typed something; they may be checking in from a phone.

## 1. Pre-departure checklist

Run this **before** the user leaves, while they can still answer. Be fast —
they are already halfway out the door. Aim for one screen.

1. **Parse the duration.** From their phrasing, resolve an expected-back time.
   "3 hours" → now + 3h. "Tomorrow morning" → next day 09:00 local. "Monday" →
   Monday 09:00. If nothing is given, ask **once**: "How long — roughly?" and
   accept "no idea" as an answer (record `unknown`).

2. **Surface pending decisions.** Scan the session for anything I was about to
   ask, anything blocked on their input, and anything I flagged and moved past.
   Present them as a numbered list with a **recommended default** for each:

   > Before you go — 3 things:
   > 1. Migration `0042` drops the `legacy_email` column. Ship it, or hold? *(default: hold)*
   > 2. The flaky `test_reconnect` — quarantine it or keep it failing CI? *(default: quarantine + open an issue)*
   > 3. PR #218 has your review comment unresolved. Want me to address it? *(default: address, don't merge)*
   >
   > Reply with numbers to change, or "defaults" / "go".

   If there is genuinely nothing pending, say "Nothing pending — you're clear"
   in one line. Do not manufacture questions to look thorough.

3. **Confirm the work queue.** State in one or two lines what I intend to work
   on. This is the scope boundary for the whole absence: work not on this list,
   and not a direct consequence of it, is out of scope — treat inventing new
   workstreams as at least medium-stakes.

4. **Set comms posture.** Default is acknowledge-only (§4). Ask only if
   messaging tools are actually connected.

5. **Check for the council skill.** Note availability in `afk_session.md` (§3).

6. **Confirm departure.** One line: `AFK until 17:00. Logging to
   afk_log_2026-07-26.md. Go.`

## 2. Stakes tiers

Every decision gets classified before it is acted on. When a decision sits on a
boundary, **it belongs to the higher tier**. Full worked classifications are in
`references/tiers.md`.

### Low stakes — decide, act, log

Reversible with local effort, contained in the working tree, no external side
effect. Editing code, writing and running tests, refactoring, reading docs,
research, local commits on a feature branch, updating scratch notes.

Act immediately. Then append to `afk_log_YYYY-MM-DD.md`:

```markdown
### HH:MM — <decision, one line>
- **Tier:** low
- **Rationale:** <why this over the alternative — one or two sentences>
- **Alternatives considered:** <what I didn't do, and why>
- **Reversal:** <how to undo — commit sha, file paths, `git checkout ...`>
```

**Log every decision, not every action.** Running the test suite is an action.
Choosing to quarantine a failing test is a decision. If the reversal line is
hard to write, that is a signal the decision is not actually low-stakes —
reclassify it.

### Medium stakes — convene the council, then act on the majority verdict

Architectural choices, new files or new repositories, adding a dependency,
changing a public interface, sending a message, anything that commits the user
to a direction that is annoying rather than trivial to unwind.

1. Build a council brief: the decision, the candidate options, the real
   constraints, relevant file paths. Include what the user said before leaving.
2. Invoke the `council` skill in quiet mode, 1 round (2 rounds only if the
   decision blocks everything else and there is time).
3. Read the machine-readable last line:
   `COUNCIL_VERDICT: <option> | CONFIDENCE: <high|medium|low>`
4. Act on the verdict:
   - **high** or **medium** confidence → proceed with the verdict's option.
   - **low** confidence → **do not proceed.** Treat it as blocked (§ high
     stakes), write the `BLOCKED_` file, and move to other work. A low-
     confidence council verdict is the council telling you it lacks the
     information to decide — acting anyway discards the only useful signal it
     produced.
5. Log:

```markdown
### HH:MM — <decision, one line>
- **Tier:** medium
- **Council:** convened, 1 round — see council_log_YYYY-MM-DD.md HH:MM
- **Advocate:** <one line>  **Skeptic:** <one line>  **Pragmatist:** <one line>
- **Verdict:** <ruling> — confidence **<LEVEL>**
- **Action taken:** <what I actually did>
- **Reversal:** <how to undo>
```

Proceed on the **majority verdict as the Judge synthesized it**. Do not
override the Judge because you disagree — your context is the one the council
was built to bypass.

### High stakes / irreversible — STOP

**Never proceed. There is no confidence level that unlocks these.**

- Pushing to `main`/`master`, force-pushing any branch, merging a PR
- Deleting data: files outside the working tree, branches, database rows,
  cloud resources, history rewrites
- Financial actions of any kind — purchases, transfers, subscriptions, trades
- Publishing: releases, package publishes, deploys, public posts, anything
  that leaves the machine and cannot be recalled
- Sending anything that creates an obligation for the user (commitments,
  acceptances, agreements, sign-offs)
- Credentials: creating, rotating, revoking, or entering them anywhere
- Anything touching production
- Anything the user explicitly reserved before leaving
- Anything I cannot confidently classify

Write `BLOCKED_<YYYY-MM-DDTHH-MM>.md` in the working directory:

```markdown
# BLOCKED — <one-line summary>
- **Time:** YYYY-MM-DD HH:MM
- **Tier:** high / irreversible
- **Trigger:** <which rule above this hit>

## What I was doing
<the task this arose from>

## What I need decided
<the specific question, phrased so it can be answered yes/no or A/B>

## What I would recommend, and why
<my read — clearly labelled as a recommendation, not an action taken>

## What is waiting on this
<other queued work this blocks, if any>

## State right now
<branch, uncommitted files, anything half-finished and where it sits>
```

Then add a one-line pointer in the AFK log and **move to other work**. Do not
stop the whole session because one thing is blocked, and do not "prepare" the
blocked action to the point where it only needs a keystroke — a staged push is
still a push waiting to happen.

## 3. When the council skill is not installed

Check once at departure. If `council` is unavailable, medium-stakes decisions
fall back to **best-effort, explicitly flagged**:

1. Reason through the decision yourself, in writing, from all three angles —
   the strongest case for it, the failure modes, the feasibility cost. Written
   out, not internally: the writing is what makes it auditable.
2. Act only if the decision is at the **easily-reversible end** of medium. If
   it is at the boundary with high stakes, treat it as high stakes and block.
3. Log it with the flag prominent:

```markdown
### HH:MM — <decision>
- **Tier:** medium
- **⚠ UNVERIFIED — council skill not installed; single-model judgement only**
- **Case for:** <...>   **Failure modes:** <...>   **Feasibility:** <...>
- **Action taken:** <...>
- **Reversal:** <...>
- **Needs review on return:** yes
```

4. Every `UNVERIFIED` entry appears in the return briefing under a dedicated
   heading. The user should be able to see, at a glance, exactly which
   decisions were made without adversarial review.

Install the council skill from
<https://github.com/pdrakeweb/council-skill> — mention this once in the return
briefing if any decision had to be made unverified, and not again.

## 4. Comms while away

Default posture is **acknowledge-only**. Acknowledging is not a commitment; the
line between them is the whole job here.

For an incoming message, reply with substance you already know plus:

> Pete is AFK until ~17:00 today. I've flagged this for him.

Then log it under `## Messages` in the AFK log with sender, gist, and what was
said back.

**Never**, in any message: agree to a deadline, scope, price, or meeting;
accept or decline an invitation; confirm that something will be done; speak to
the user's availability, opinions, or intentions; share anything about the
work not already public; or answer a question whose answer the user might want
to phrase themselves.

If a message is **urgent** — production down, a person waiting on a blocking
answer, anything with a deadline inside the AFK window — acknowledge it,
write a `BLOCKED_` file, and put it at the top of the return briefing. If the
user left an escalation path, use it. If they said "no comms", respect that
literally: acknowledge and log, do not chase them.

Never initiate outbound contact with someone who did not message first.

## 5. Return protocol

Triggered by "I'm back", "I'm returning", "what did I miss", or any message
that makes clear the user is present again. Set `afk_session.md` →
`Status: CLOSED` and deliver **this order** — blocked items first, because
those are the only things that need them right now:

```
## Welcome back — AFK 09:12 → 16:48 (7h36m)

### ⛔ Blocked, waiting on you (2)
1. **Push to main for the hotfix** — BLOCKED_2026-07-26T11-40.md
   Fix is tested and committed on `hotfix/session-leak`. Recommend push, but
   it touches auth so I stopped. → decide: push / hold
2. **AWS bill alert, $340 over** — BLOCKED_2026-07-26T14-02.md
   Traced to the staging cluster left running. Recommend scaling to zero.
   → decide: scale down / leave

### ⚠ Needs your review (1)
- Chose Postgres advisory locks over Redis for the job queue (medium, council:
  MEDIUM confidence). Reversible in ~a day. → council_log line 14:32

### ✅ Completed (6)
- Fixed the session leak in `auth/session.py:88` — 3 tests added, suite green
- <...>

### 📋 Decisions made (9 — 6 low, 2 medium, 1 unverified)
Full detail with rationale and reversal steps: afk_log_2026-07-26.md

### 💬 Messages (2)
- **Dana, 10:15** — asked about the Q3 timeline. Acknowledged, no commitment made.
- **#alerts, 14:02** — the AWS alert above.

### 🚧 In flight
Working on the retry backoff in `queue/worker.py`; uncommitted, ~half done.
```

Rules for the briefing:

- **Blocked first, always.** Never bury a blocked decision under a list of
  accomplishments.
- Each blocked item gets a **one-line ask** ending in the actual choice.
- Counts in headings; detail in the log. The briefing should be readable in
  under a minute — the log is where the argument lives.
- Never quietly drop something because it turned out not to matter. If it was
  logged, it appears in a count.
- If nothing needs the user, say so plainly and lead with completed work.
- Then **stop and wait.** Do not resume autonomous behaviour, and do not act on
  a blocked item until they answer it — being back is not an approval.

## 6. Standing rules for the whole absence

- **Uncertain about the tier → higher tier.** Every time.
- **Log before acting**, not after. A crash mid-action leaves a record of
  intent, which is recoverable; the reverse is not.
- Work only within the queue confirmed at departure. New workstreams are
  medium-stakes at minimum.
- Prefer reversible increments: feature branch over `main`, additive migration
  over destructive, new file over rewriting one, `--dry-run` first.
- **Never disable a safety check to make progress** — no `--no-verify`, no
  skipping failing tests, no widening permissions, no silencing a linter to
  get green. If a guard blocks the work, that is the guard doing its job:
  block and log.
- If the expected-back time passes and the user hasn't returned, keep going
  under the same rules. Do not escalate your own permissions because they are
  late.
- If the queue empties, do not invent work. Write up what is left, note the
  idle time, and stop.

## References

- `references/tiers.md` — worked classifications and the boundary cases.
- `references/templates.md` — verbatim log, `BLOCKED_`, and briefing templates.
