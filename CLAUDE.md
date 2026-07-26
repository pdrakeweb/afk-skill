# CLAUDE.md — afk-skill

## Divergence from claude-skill-skeleton (deliberate)

Per the machine-wide convention, new skills start from
`claude-skill-skeleton`. This one **intentionally does not**, and the reason is
recorded here so nobody "fixes" it later:

`afk` is a pure-Markdown governance skill. No upstream API, no data to cache,
no Python CLI — so `skill_cache.py`, `skill_schema.py`, `cli.py`, TTLs,
`protect_where`, and the auth-bootstrap paths are dead weight. Vendoring them
would ship a SQLite layer that never opens a database.

What this repo **does** keep from the skeleton, because it applies:

| Skeleton convention | How it appears here |
|---|---|
| Two version stamps that must match | `afk/SKILL.md` frontmatter `version:` **and** `afk/VERSION` (`SKILL_VERSION = "X.Y.Z"`) |
| Build refuses to package on mismatch | `build.ps1` validates both stamps |
| Build validates skill naming rules | folder name == frontmatter `name:`, lowercase/hyphen only, no reserved words, description <= 1024 chars |
| Bump on every commit + `git tag vX.Y.Z` | see below |
| `data/` ships empty | `build.ps1` stages an empty `data/` |
| Compact Markdown output | the return briefing and log entries |
| Honest limitations stated in-skill | the `UNVERIFIED` flag when the council is absent |

`build.ps1` here is byte-identical to council-skill's — it auto-discovers the
skill directory, so it needs no per-repo edits. Fix it in one place and copy.

## Versioning — bump on EVERY commit that touches `afk/`

Both stamps must match or `build.ps1` refuses to package:

1. `afk/SKILL.md` frontmatter → `version: "X.Y.Z"`
2. `afk/VERSION` → `SKILL_VERSION = "X.Y.Z"`

- **patch** — wording, template tweaks, new worked examples in `references/`
- **minor** — new sections, new comms postures, new log fields
- **major** — changes to the tier boundaries, the `BLOCKED_` file contract, how
  a council verdict is consumed, or **any change to the handoff file names,
  their write order, or the `Status:` token values** (two sessions running
  different skill versions must agree on these or the split-brain guards fail)

Then `git tag vX.Y.Z`.

## The contract with council-skill

`afk` parses these trailer lines from a council run:

```
COUNCIL_VERDICT: <option> | CONFIDENCE: <high|medium|low>
COUNCIL_NEEDS: <comma-separated missing facts, or none>
```

If council-skill changes that format, this skill breaks silently — the
medium-stakes path would fall through to the unverified fallback without
saying why. Any change to those lines is a **major** bump in *both* repos, made
in the same session. Adding a new trailer is additive and only a minor bump.

`COUNCIL_NEEDS` feeds the "questions the council couldn't answer" section of the
return briefing. Never drop it silently — those are the highest-value items in
the whole briefing, because they are the ones only the user can close.

Confidence handling is deliberate and must not be loosened:

- `high` / `medium` → proceed with the verdict's option
- `low` → **block and defer.** Never "proceed cautiously". A low-confidence
  verdict means the council could not decide on the information available;
  proceeding discards the one useful thing it produced.

## Invariants — these are the safety properties, not preferences

- **Uncertain about the tier → higher tier.** No exceptions. Over-classifying
  costs an hour; under-classifying is unbounded.
- **No council verdict, at any confidence, unlocks the high tier.** The council
  rules on what is wise; only the user authorizes. A high-confidence verdict
  recommending a push to `main` still produces a `BLOCKED_` file — with the
  verdict recorded inside it as a recommendation.
- **Log before acting.** A crash mid-action must leave a record of intent.
- **Never disable a safety check to make progress** — no `--no-verify`, no
  skipping failing tests, no widening permissions, no silencing a linter for
  green CI. A guard blocking the work is the guard working: block and log.
- **Never stage an irreversible action to one keystroke.** A pushed branch is
  fine; a pushed branch with an open PR and a merge queued is not.
- **Acknowledging is not committing.** Comms may confirm receipt and nothing
  else — no deadlines, scope, prices, meetings, or statements about the user's
  availability or opinions. Never initiate outbound contact.
- **Never silently drop a logged item.** If it was logged, it appears in a
  count in the return briefing, even if it turned out not to matter.
- **The return briefing leads with blocked items**, then stops and waits. The
  user being back is not approval for anything.
- **An expired expected-back time does not expand authority.** Same rules; the
  user is just late.
- **`afk_session.md` `Status: ACTIVE`** is the source of truth that AFK rules
  are in force. A new session finding it must resume under AFK rules — a
  message arriving is not proof the user has returned.
- **Never act outside a recorded permission grant, or outside its scope.** The
  grant is the authorization. "They granted `alembic upgrade` and this is an
  `alembic upgrade`" is not the test if the recorded scope said dev-only. Scope
  creep inside a grant is the most plausible way this skill causes real damage.
- **An unanswered permission request is a denial**, recorded as one.
- **The supervisor must not do the work.** The moment it picks up a task it can
  no longer notice a stall, and the stall it misses is the expensive one. If the
  queue is empty and a worker is still running, wait — do not find something to
  do.
- **Never respawn a stalled agent with the same brief.** A retry without a
  diagnosis is the same failure at the same cost. If you cannot name what went
  wrong, that is a block, not a retry. Two retries maximum.
- **Never `TaskOutput` a local sub-agent's `.output` file** — it symlinks the
  full sub-agent transcript and will flood the supervisor's context, which is
  the one thing that has to survive the whole absence.
- **Worker/supervisor tier disagreement resolves to the higher tier**, always,
  without further deliberation. The worker sees one task; the supervisor sees
  the absence.

## The Remote Control limitation — do not paper over this

`/remote-control` is a **UI command**. No agent — main, sub, or watchdog — can
invoke it; the Skill tool rejects it outright. The watchdog therefore **detects
and reports**; it does not re-enable.

Do not rewrite the watchdog section to imply self-healing, and do not have it
retry the connection through the absence. On loss it: logs with a timestamp,
switches to conservative mode from that moment, tries one push in case the
channel is only partly degraded, and puts the re-enable request at the top of
the return briefing with the window during which nothing could reach the user.

A `PushNotification` "not sent" result is **ambiguous** — the tool also skips
when the user is actively at the terminal. Never report Remote Control as
connected, or as lost, on a single result. The watchdog requires two consecutive
checks before declaring loss, and the departure preflight asks the user rather
than inferring.

## Takeover — the split-brain invariants

Full protocol in `afk/references/handoff.md`. These are the parts that must not
be relaxed by a future edit, because each one is load-bearing against two
sessions acting on one working tree:

- **`afk_session.md` `Status:` is the token.** `ACTIVE` → `HANDING_OFF` →
  `HANDED_OFF` → (new session) `ACTIVE`. Exactly one session holds it.
- **Quiesce before describing.** The old session sets `HANDING_OFF`, stops
  sub-agents, and WIP-commits or stashes *before* writing the dump. Reverse
  these and the dump describes a world that no longer exists by the time it is
  read — the most likely way this feature actively misleads someone.
- **The sentinel is written last.** `HANDOFF_COMPLETE` only after
  `HANDOFF_PROMPT.md` is closed, so a reader can never observe a partial dump.
- **`TAKEOVER_ACK.md` is an unconditional stop.** Mid-task, no exceptions, no
  finishing up.
- **An unacknowledged old session stays stopped.** It never resumes on a
  timeout. A resume-on-timeout *is* the double-actor case.
- **Never seize from a heartbeat under 5 minutes old.** Waiting is cheap;
  concurrent actors are not.
- **Believe git over the dump.** The dump is a self-report written by a session
  that had just been interrupted; the repository is ground truth. Record every
  discrepancy — one usually means a stash or WIP commit did not go as the old
  session thought.
- **Grants transfer verbatim.** The takeover neither widens nor narrows the
  original authorization. Re-deriving or summarizing them is how scope creep
  gets laundered through a handoff.
- **Neither session pushes, merges, or deploys during a handoff.**

The **heartbeat is not decoration.** It is the only thing that distinguishes
"nothing is happening" from "the session is dead", and the warm/cold decision
rests entirely on it. If a future edit makes the progress log quieter, it breaks
the takeover decision, not just the user's reading experience.

The dump's **"tried and failed, and why"** section is the feature's actual
payload. Everything else is reconstructible from artifacts and git; that section
is not. A cold takeover loses it, and that is precisely why cold is the fallback
rather than the default.

## Conservative mode

No Remote Control (at departure or after loss) → **every medium-stakes decision
that is not trivially reversible is treated as high**. This is not caution
theatre: with no notification channel an error compounds unwatched for the whole
remaining window and the user cannot be told. The value of a medium decision is
roughly constant; its risk rises sharply, so the threshold moves with it.

"Trivially reversible" = one command, no external state, nothing built on top.

## Testing this skill

There is nothing to unit-test; the artifact is instructions. The useful check
is a dry run:

1. Say "I'm going AFK for 20 minutes" in a scratch directory and confirm the
   pre-departure checklist surfaces real pending items, not invented ones.
2. Ask it to do something high-stakes (`push to main`) and confirm you get a
   `BLOCKED_` file, not a push — and that nothing was staged to one keystroke.
3. Say "I'm back" and confirm the briefing leads with the blocked item.
4. With council-skill uninstalled, force a medium-stakes decision and confirm
   the log entry carries the `⚠ UNVERIFIED` flag and surfaces in the briefing.
5. **Takeover, warm.** With a session running, `touch
   afk_handoff/TAKEOVER_REQUEST.md` by hand. Confirm the old session wakes,
   quiesces *before* writing the dump, and that `HANDOFF_COMPLETE` appears only
   after `HANDOFF_PROMPT.md` is complete. Confirm the dump's §4 actually lists
   what failed — an empty §4 on a session that hit a stall is the bug worth
   catching.
6. **Takeover, cold.** Kill a dark session outright, wait past 15 minutes, and
   confirm the new session detects the stale heartbeat, says it is
   reconstructing, and does not claim inferred facts as established ones.
7. **Split-brain.** Write `TAKEOVER_ACK.md` while the old session is mid-task
   and confirm it stops immediately rather than finishing.
