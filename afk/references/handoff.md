# Takeover protocol

When Remote Control drops, the AFK session keeps working but goes **dark**: it
can still read and write the filesystem, but it can no longer reach the user,
and the user can no longer see it. The session is not broken — it is
*unobservable*, which is worse, because it looks identical to a session that
died.

This protocol turns that into two recoverable options for the user:

1. **Follow along** — a running progress log on disk they can `tail` from
   anywhere, without touching the session.
2. **Take over** — start a *new* Claude Code session that has working Remote
   Control, tell the old session to dump everything it knows, and resume from
   that dump.

The whole design rests on one constraint: **the two sessions share a filesystem
and nothing else.** They cannot call each other. Every signal is a file.

---

## The files

All handoff state lives in `afk_handoff/` beside the other AFK artifacts.

| File | Written by | Meaning |
|---|---|---|
| `afk_progress.md` | old (dark) session | Append-only running log. The follow-along artifact **and** the liveness signal. |
| `afk_handoff/TAKEOVER_REQUEST.md` | **new** session | "I have Remote Control. Dump your state and stand down." |
| `afk_handoff/HANDOFF_PROMPT.md` | old session | The knowledge dump, written as an ingestible prompt. |
| `afk_handoff/HANDOFF_COMPLETE` | old session | Sentinel, written **last**. Until it exists, the prompt may be half-written. |
| `afk_handoff/TAKEOVER_ACK.md` | new session | "I have it. You are done. Stop." |

Two ordering rules that make this safe on a filesystem with no locking:

- **The sentinel is always written last**, after `HANDOFF_PROMPT.md` is closed.
  A reader that checks for `HANDOFF_COMPLETE` can never observe a partial dump.
- **`TAKEOVER_ACK.md` is written last of all**, and is the old session's
  unconditional stop signal. If it exists, the old session stops — mid-sentence,
  mid-task, no exceptions.

---

## Part 1 — Going dark: the progress log

The moment the watchdog declares Remote Control lost (two consecutive failed
checks — SKILL.md §2), start `afk_progress.md`.

### What it is for

Two readers, and the format has to serve both:

- **The user**, on a phone or another terminal, running
  `tail -f afk_progress.md` to see whether anything is happening.
- **The new session**, reconstructing state during a cold takeover if the old
  session never answers.

Write for the first; the second is what you get for free if you do it honestly.

### Rules

- **Append-only.** Newest at the bottom. Never rewrite history in this file —
  someone may be tailing it, and a rewrite makes the tail lie.
- **One line per event**, timestamped, UTC, ISO-8601. Prose, not JSON — a human
  is reading this on a phone.
- **Heartbeat every 2 minutes even when nothing happens.** This is the important
  one. Without it, "no entries" is ambiguous between *nothing is happening* and
  *the session is dead*, and those need completely different responses from the
  user. A heartbeat collapses the ambiguity.
- **Every state change gets a line**: task started, sub-agent spawned, sub-agent
  returned, decision logged, stall detected, block written, council convened.
- **Never put a secret, token, or credential in it.** It is a file the user may
  well paste into another window.

The heartbeat runs as a detached background loop so it costs the supervisor no
context:

```bash
# Bash tool, run_in_background: true
while [ ! -f afk_handoff/TAKEOVER_ACK.md ]; do
  printf -- '- `%s` — heartbeat · supervisor alive · %s\n' \
    "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "${AFK_PHASE:-working}" >> afk_progress.md
  sleep 120
done
```

It self-terminates when the takeover is acknowledged, so a handed-off session
leaves no orphan process writing to a log the new session now owns.

### Arm the takeover listener at the same time

The old session cannot poll — it is an agent, not a daemon. So it arms a
one-shot background watcher whose *completion notification* is what wakes it:

```bash
# Bash tool, run_in_background: true
until [ -f afk_handoff/TAKEOVER_REQUEST.md ]; do sleep 15; done
echo "TAKEOVER REQUESTED"
head -20 afk_handoff/TAKEOVER_REQUEST.md
```

A single `until` loop that exits when the condition is true is exactly right
here: one notification, no polling cost, and the exit *is* the signal. Do not
use a `Monitor` for this — monitors are for streams of recurring events; an
unbounded watcher would sit armed after the event had already fired.

### Tell the user how to reach it

Put this in the progress log's header, because at that point it is the only
channel left:

```markdown
# AFK progress — dark since 12:40 UTC
Remote Control dropped. I cannot reach you and cannot re-enable it
(`/remote-control` is a UI command no agent can invoke).

**Follow along:**   tail -f afk_progress.md
**Take over:**      start a new Claude Code session in this directory with
                    Remote Control working, and say "take over the AFK session".
```

---

## Part 2 — The new session requests takeover

Triggered by "take over the AFK session", "resume the AFK session", "the other
session lost connection", or the user landing in a directory with an `ACTIVE`
`afk_session.md` and a dark marker.

### Step 1 — Establish what you are taking over from

Read, in this order: `afk_session.md`, the tail of `afk_progress.md`,
`afk_log_*.md`, any `BLOCKED_*.md`, and `git status` + `git log --oneline -10`.

Do not skip the git state. It is the only source here that cannot lie.

### Step 2 — Decide: warm or cold

Compare the last heartbeat in `afk_progress.md` against now.

| Last heartbeat | Old session is | Do |
|---|---|---|
| < 5 min ago | **alive** | **Warm takeover.** Request the dump. Do not seize. |
| 5–15 min ago | ambiguous | Request the dump, wait the full timeout. Assume alive. |
| > 15 min ago, or no progress log | **dead or never went dark** | **Cold takeover** (Part 5). |

**Never seize from a live session.** Two sessions acting on one working tree is
the failure this whole protocol exists to prevent, and it is much worse than
waiting four minutes. The heartbeat exists precisely so this call can be made on
evidence instead of a guess.

### Step 3 — Confirm with the user

They are present — that is the entire point of the new session. Show them what
you found and get one confirmation before writing anything:

> Found an AFK session, dark since 12:40 (4h12m), last heartbeat 40s ago — so
> it's alive and still working. It's mid-task on `queue/worker.py` with 2
> blocked items waiting.
>
> Take over? I'll ask it to dump its state and stand down. It stops working the
> moment it sees the request. → **take over / just watch**

"Just watch" is a legitimate answer: `tail -f afk_progress.md` costs nothing and
lets the old session finish. Do not push them toward takeover.

### Step 4 — Write the request

`afk_handoff/TAKEOVER_REQUEST.md` — template in `templates.md`. It carries the
requesting session's identity, timestamp, and how long it will wait.

### Step 5 — Wait for the sentinel

```bash
# Bash tool, run_in_background: true
t=0
until [ -f afk_handoff/HANDOFF_COMPLETE ]; do
  sleep 10; t=$((t+10))
  [ $t -ge 600 ] && { echo "HANDOFF TIMEOUT after ${t}s"; exit 1; }
done
echo "HANDOFF COMPLETE"
```

**Ten minutes**, because the old session has real work to do first: it must stop
sub-agents and commit or stash in-flight work before it can honestly describe
its state. A dump written before quiescing describes a world that no longer
exists by the time it is read.

On timeout, fall through to cold takeover (Part 5) — and say so explicitly,
because a timed-out warm takeover is strictly less trustworthy than a cold one
that knew it was cold from the start.

---

## Part 3 — The old session hands off

The watcher fires. **Everything below happens before any further work**, and in
this order. The order is the correctness property.

### Step 1 — Quiesce. Do this first.

1. Set `afk_session.md` → `Status: HANDING_OFF`. From this line on, **this
   session may not start anything new.** Not a task, not a sub-agent, not a
   council, not a commit that wasn't already in flight.
2. `TaskStop` every running sub-agent. Note what each was doing and how far it
   got — that goes in the dump.
3. Get the working tree into a **describable** state: commit in-flight work to
   the feature branch with a `WIP:` message, or stash it. Either is fine;
   leaving a dirty tree undescribed is not.
4. Record the resulting `git status`, branch, and HEAD sha.

**Describe second, quiesce first.** Reversing these two produces a dump that is
already stale on arrival, which is the single most likely way this feature
misleads someone.

### Step 2 — Write the dump

`afk_handoff/HANDOFF_PROMPT.md`. It is a **prompt**, not a report: it opens by
addressing the new session in the imperative, so pasting it verbatim works. Full
template in `templates.md`. Nine sections, and the fourth is the one that
justifies the whole feature:

1. **Identity** — repo, branch, HEAD, working directory, when the old session
   started, when it went dark.
2. **The original mandate** — verbatim from `afk_session.md`: the work queue,
   permission grants **with scope**, denials, expertise answers,
   pre-authorizations, reserved items. Verbatim, not summarized. A paraphrase of
   a permission grant is how scope creep happens.
3. **Done** — completed items with commit shas.
4. **Tried and failed, and why** — every stall, every dead end, every approach
   abandoned and the reason. *This is the highest-value section in the file.*
   Without it the new session's first act is to repeat a failure the old session
   already paid for, and the takeover is a net loss.
5. **In flight** — exactly where the current task stopped, what the sub-agent
   was doing, what is committed vs stashed vs uncommitted.
6. **Decisions** — the load-bearing ones inline with rationale; the rest by
   pointer to `afk_log_*.md`.
7. **Blocked** — pointers to every `BLOCKED_*.md`, with one line each.
8. **Open questions** — `COUNCIL_NEEDS` items and anything only the user knows.
   **The user is present in the new session** — these can finally be asked.
9. **Next action** — one specific thing, and why it is next.

### Step 3 — Sentinel, then stand down

Write `afk_handoff/HANDOFF_COMPLETE` **after** the prompt file is closed. Then:

- `afk_session.md` → `Status: HANDED_OFF`
- Append a final line to `afk_progress.md`: handed off, at what time, to whom.
- **Stop.** Do no further work. Not "one small thing while I wait" — that one
  small thing is the split-brain.
- Wait for `TAKEOVER_ACK.md`, then terminate the heartbeat and exit.

If the ack never comes, stay stopped. A session that resumes because nobody
confirmed is exactly the double-actor scenario the protocol prevents. Say so in
the progress log and stop.

---

## Part 4 — The new session takes over

### Step 1 — Ingest, then verify against reality

Read `HANDOFF_PROMPT.md`. Then **check it against the filesystem**:

```bash
git status --short && git log --oneline -5 && git rev-parse HEAD
```

If the dump's claimed HEAD, branch, or tree state disagrees with what git says,
**believe git.** The dump is a self-report written by a session that had just
been interrupted; the repository is ground truth. Note every discrepancy in the
new log — a discrepancy usually means the old session's stash or WIP commit did
not go the way it thought, and that is worth knowing before touching anything.

### Step 2 — Acknowledge

Write `afk_handoff/TAKEOVER_ACK.md`. This releases the old session and stops its
heartbeat. **Do not begin work before writing it** — the window between reading
the dump and acknowledging is the only interval where both sessions could
plausibly act, and it should be as short as possible.

### Step 3 — Re-establish the session

- New `afk_session.md`: `Status: ACTIVE`, Remote Control `connected`, mode back
  to normal, **grants and reserved items copied verbatim from the dump** — the
  original authorization carries over exactly, neither widened nor narrowed.
- New `afk_log_YYYY-MM-DD.md`, or continue the existing one with a clear
  `## Session 2 — took over at HH:MM` header. Cross-reference the old log by
  name so the record stays continuous.
- Arm a fresh watchdog.

### Step 4 — Report to the user, then ask the questions

The user is *present* now. This is the first chance in hours to close the open
questions, so lead with them rather than with a status report:

```
Took over the AFK session (dark 12:40 → 16:52, 4h12m).

**Questions that blocked work — can you answer now?**
1. Is `legacy_email` read outside this repo? *(blocked migration 0043)*
2. Is the 30s retry cap a real SLA? *(council said MEDIUM on it)*

**Blocked, waiting on you (2)** — BLOCKED_2026-07-26T11-40.md, …
**Done while dark:** 4 items, 9 decisions logged
**Tried and failed:** `npm install` stalls on a peer-dep prompt — using
`npm ci`. Don't re-try the former.
**Next:** finish the retry backoff in `queue/worker.py` (WIP commit a3f9c21)

Continuing AFK rules — you said back at 17:00. Or say "I'm back" and I'll
close the session out.
```

Then continue under normal AFK rules.

---

## Part 5 — Cold takeover (no dump is coming)

The old session is dead, wedged, or timed out. There will be no dump. Proceed
from artifacts alone: `afk_session.md`, `afk_log_*.md`, `afk_progress.md`,
`BLOCKED_*.md`, and git.

Cold takeover is a **reconstruction**, and the whole risk is forgetting that:

- **Label every reconstructed fact as reconstructed** in the new log. Do not let
  an inference from the progress log harden into a stated fact one turn later.
- **Trust git over every other artifact.** The logs describe intent; the
  repository records what happened.
- **The "tried and failed" knowledge is gone.** That is the real cost, and it
  means the first thing you do is likely to be something the old session already
  proved doesn't work. Re-derive from `afk_progress.md` where you can, and say
  in the briefing that you are flying without it.
- **Treat the first action as medium-stakes minimum**, whatever it looks like.
  You do not yet know what was half-done. Check the tree before changing it.
- **Never assume the old session is dead because it is quiet.** Before writing
  anything, re-check the heartbeat one final time. If it has ticked since you
  decided it was dead, it is alive — abort and go warm.

If uncommitted changes exist that no artifact explains, **stop and ask the
user** — they are present. Do not clean, stash, or commit work whose provenance
you cannot establish.

---

## Split-brain: the one thing that must not happen

Two sessions acting on one working tree corrupts state in ways no log can
reconstruct. Every rule below exists for that, and none of them is negotiable.

| Rule | Why |
|---|---|
| `afk_session.md` `Status:` is the **token**. `ACTIVE` → `HANDING_OFF` → `HANDED_OFF`. Exactly one session holds it. | Single source of truth for who may act. |
| The old session stops acting the instant it writes `HANDING_OFF` — before writing the dump. | The dump takes minutes; acting during them makes it false. |
| The new session does not act until it has written `TAKEOVER_ACK.md`. | Bounds the overlap window to file I/O. |
| `TAKEOVER_ACK.md` existing stops the old session **unconditionally** — mid-task, no exceptions, no finishing up. | The one signal that needs no interpretation. |
| Never seize from a session whose heartbeat is under 5 minutes old. | Waiting is cheap; concurrent actors are not. |
| An unacknowledged old session stays stopped. It never resumes on its own. | A resumed-on-timeout session is the double-actor case. |
| Neither session pushes, merges, or deploys during a handoff. | High tier is blocked anyway — and this is the worst possible moment for it. |

**If you are ever unsure which session holds the token: stop and ask the user.**
They are present in at least one of the two sessions, which is the entire reason
a takeover was possible at all.
