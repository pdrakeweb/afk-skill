# afk-skill

A Cowork / Claude Code skill that governs autonomous work **while you are away
from keyboard**.

It is not a productivity maximizer. Its purpose is narrower and more useful:
make sure every decision taken in your absence is one you'd have agreed with —
and that every decision that isn't is still waiting for you, undamaged, when
you get back.

Two failure modes matter equally, and most of the skill is about the second:

1. **Doing something unwanted** — guarded by the stakes tiers.
2. **Getting stuck and doing nothing** — guarded by the departure sequence, the
   human-approval guard, the delegation supervisor, and routing questions to the
   council instead of to you.

> The default when uncertain is **stop and log**, not proceed. Unfinished work
> is cheap. An unwanted irreversible action is not. But an agent frozen for six
> hours on a permission prompt is a *wasted* absence, and that is preventable.

## Triggering

Say any of:

```
I'm going AFK for 3 hours
AFK mode
heading out, back around 5
I'll be offline until tomorrow morning
going dark until Monday — no comms
```

The skill extracts the duration if you give one and asks once if you don't.
On return, `"I'm back"` / `"what did I miss"` runs the return briefing.

## What it does

**1 — Departure sequence.** Everything here can *only* be done while you're
present, so it runs first and fast:

- **Remote Control preflight.** Checks whether notifications can reach your
  phone. It **cannot enable Remote Control** — `/remote-control` is a UI
  command no agent can invoke — so it tells you to run it before you go. The
  answer changes behaviour: with no channel, the skill runs in **conservative
  mode**, where medium decisions that aren't trivially reversible get blocked,
  because an error would compound unwatched for the whole window.
- **Expertise questions, asked immediately.** Everything needing *your specific
  knowledge* — business context, intent behind existing code, whether a
  constraint is real — batched into one list with defaults. Not things it could
  research or council; things only you know.
- **Permission pre-authorization sweep.** A permission prompt while you're away
  is a stalled session. It enumerates the specific commands, network access, and
  MCP servers the queued work will hit and gets them approved in one batch.
  Grants are recorded **verbatim with their scope** — a grant of "`alembic
  upgrade` on dev" is not a grant on staging. An unanswered request is a denial.
- Then the ordinary pending decisions, the work queue (the scope boundary for
  the whole absence), and the watchdog.

**2 — Stakes tiers.** Every decision is classified before it is acted on. On a
boundary, it goes to the **higher** tier.

| Tier | Examples | Behaviour |
|---|---|---|
| **Low** | editing code, tests, refactors, research, local commits | Decide, act, log with rationale + reversal steps |
| **Medium** | architecture, new files/repos, new dependency, sending a message | Convene the [council skill](https://github.com/pdrakeweb/council-skill); act on the verdict; log the debate |
| **High / irreversible** | pushing to `main`, deleting data, money, publishing, credentials, production | **STOP.** Write `BLOCKED_<timestamp>.md` and move on |

There is **no confidence level that unlocks the high tier** — not even a
high-confidence council verdict. The council rules on what is *wise*, not on
what you have *authorized*. The verdict goes *inside* the `BLOCKED_` file as a
recommendation, so you can act on it in one keystroke when you're back.

**3 — Questions go to the council, not to you.** Any question that would
normally have produced a "which would you prefer?" turn — from the supervisor or
any sub-agent — is packaged as a council brief and run instead. That's the rule
that keeps the session moving. Three exceptions: high-tier things (blocked),
things only you know (blocked, and they should have been caught at departure),
and things answerable by just looking (go look).

**4 — Human-approval guard.** Most AFK failures aren't bad decisions; they're an
agent 80% through an approach that turns out to need a human. So before starting
anything: *does this approach require a human at any point?* If yes, pick a
different approach now or block it now. Ships with a table of the usual
offenders — `git rebase -i`, `npm login`, OAuth, `sudo`, CAPTCHAs, bare `npm
install` — and their non-interactive alternatives. Never disables a safety check
to dodge a prompt.

**5 — Delegated execution.** The main agent **doesn't do the work**. It spawns a
background sub-agent per queue item and supervises: watching for stalls,
`TaskStop`-ing stuck workers, and respawning with an explicit failure brief
("previous attempt died at `npm install` waiting on a peer-dep prompt — use
`npm ci`; if the lockfile is stale that's a BLOCK, not a thing to resolve").

An agent stuck on a permission prompt *cannot report that it's stuck* — that's
what being stuck means — so something outside it has to notice. An agent busy
doing the work has no attention left to notice, and its context fills. Two
retries maximum, each with new information; never the same brief twice. Three
failures is a signal about the approach, not the agent.

Sub-agents are briefed to **return, not ask**: "returning with a question is a
successful outcome and costs the session nothing; stalling costs it hours."

**6 — Decision log.** Every decision (not every action) goes to
`afk_log_YYYY-MM-DD.md` with rationale, alternatives considered, and how to
reverse it. Logged *before* acting, so a crash mid-action leaves a recoverable
record of intent.

**7 — Comms.** Acknowledge-only by default: *"The user is AFK until ~17:00. I've
flagged this for them."* Never agrees to a deadline, scope, price, or meeting;
never speaks to your availability or opinions; never initiates outbound contact.
Anything beyond an acknowledgement is a medium-stakes decision — councilled or
blocked.

**8 — Return briefing.** Ordered so anything needing action sits above anything
that's merely news: do-this-first → blocked → questions the council couldn't
answer → needs review → completed → decisions → **stalls and retries** →
messages → in flight. Then it stops and waits: you being back is not an
approval.

The stalls section is where the *next* absence gets better — a stall on a
missing permission is a permission to pre-authorize next time, and it says so
explicitly rather than leaving you to infer it from a symptom.

## The watchdog, and what it honestly can't do

Saying "AFK mode" arms a long-lived background agent whose only job is to watch
— no work, so its context stays clean enough to last the absence. It watches for
Remote Control loss, stalled sub-agents, and queue exhaustion.

**On Remote Control loss it cannot self-heal.** `/remote-control` is a UI
command; no agent, main or sub, can invoke it. So it logs the loss, switches to
conservative mode from that moment (the tier boundary moves because the safety
net is gone), tries one push anyway in case the channel is only partly degraded,
**goes dark**, and puts *"re-run `/remote-control`"* at the top of your return
briefing with the window during which nothing could reach you.

## Going dark, and taking over

A session that loses Remote Control isn't broken — it's **unobservable**, which
is worse, because it looks exactly like a session that died. Going dark turns
that into two things you can actually do.

### Follow along

It starts `afk_progress.md` — append-only, tailable from anywhere:

```bash
tail -f afk_progress.md
```

One line per state change, plus a **heartbeat every 2 minutes even when nothing
happens**. The heartbeat is the load-bearing part: without it, "no entries" is
ambiguous between *nothing is happening* and *the session is dead*, and those
demand opposite responses from you. It runs detached, so it costs the supervisor
no context.

### Or take over

Start a new Claude Code session in the same directory with Remote Control
working and say **"take over the AFK session"**. The two sessions share a
filesystem and nothing else, so every signal is a file:

| File | Written by | Meaning |
|---|---|---|
| `afk_handoff/TAKEOVER_REQUEST.md` | **new** | "I have Remote Control. Dump and stand down." |
| `afk_handoff/HANDOFF_PROMPT.md` | old | The knowledge dump, written as an ingestible prompt |
| `afk_handoff/HANDOFF_COMPLETE` | old | Sentinel, written **last** — until it exists the dump may be partial |
| `afk_handoff/TAKEOVER_ACK.md` | **new** | "I have it. Stop." |

The old session can't poll — it's an agent, not a daemon — so it arms a one-shot
background `until` loop watching for the request file. That loop's *completion
notification* is what wakes it.

On waking it **quiesces before describing**: stops sub-agents, WIP-commits or
stashes in-flight work, *then* writes the dump. That order is the correctness
property — a dump written before quiescing describes a world that no longer
exists by the time it's read.

The dump's most valuable section is **"tried and failed, and why"**. Without it
the new session's first act is to repeat a failure the old one already paid for,
and the takeover is a net loss.

### Warm, or cold

The new session decides on evidence, not a guess — the heartbeat age:

| Last heartbeat | Old session | Action |
|---|---|---|
| < 5 min | **alive** | Warm takeover — request the dump. **Never seize.** |
| 5–15 min | ambiguous | Request, wait the full 10-minute timeout |
| > 15 min / no log | dead | Cold takeover — reconstruct from artifacts |

Cold takeover is a *reconstruction*, and everything it produces is labelled as
such. It trusts git over every other artifact, and the "tried and failed"
knowledge is simply gone — which is the real cost, and the briefing says so.

The new session verifies the dump against `git` before acting (**believe git** —
the dump is a self-report from a session that had just been interrupted), and
carries the permission grants over **verbatim**: the original authorization
transfers exactly, neither widened nor narrowed. Then it leads your briefing
with the **open questions**, because you're finally there to answer them.

### Split-brain

Two sessions acting on one working tree corrupts state in ways no log can
reconstruct. `afk_session.md` `Status:` is the token — `ACTIVE` → `HANDING_OFF`
→ `HANDED_OFF` — and exactly one session holds it. The old session stops the
instant it writes `HANDING_OFF`, *before* the dump. The new one doesn't act
until it has written the ack. `TAKEOVER_ACK.md` stops the old session
unconditionally, mid-task, no finishing up. An unacknowledged old session stays
stopped forever rather than resuming on a timeout — a resume-on-timeout is
exactly the double-actor case.

## Council integration, and the fallback

Medium-stakes decisions **and every question that arises** go to the
[`council`](https://github.com/pdrakeweb/council-skill) skill — debaters in
clean parallel contexts who then cross-examine each other, and a Judge who rules
with a confidence level. AFK reads the machine-readable verdict line and:

- **high / medium** confidence → proceed with the verdict's option
- **low** confidence → **do not proceed.** Block and defer. A low-confidence
  verdict is the council saying it lacks the information to decide; acting
  anyway throws away the only useful signal it produced.

Every `COUNCIL_NEEDS` item — the facts the council lacked — is carried into your
return briefing as "questions the council couldn't answer". Those are precisely
the things worth your attention.

**If the council skill is not installed**, medium-stakes decisions fall back to
best-effort: reason through the case-for, failure modes, and feasibility *in
writing*, act only at the easily-reversible end of medium, and log the entry
flagged `⚠ UNVERIFIED — single-model judgement only`. Every unverified decision
gets its own heading in the return briefing, so you can see at a glance exactly
which calls were made without adversarial review. Questions then have nowhere to
route, so the threshold tightens: a question that would have been councilled and
can't be is a block unless it's trivially reversible.

## Files it writes

Into the working directory, not the repo:

| File | Purpose |
|---|---|
| `afk_session.md` | Departure/return times, Remote Control state, mode, work queue, **permission grants and denials verbatim with scope**, expertise answers, pre-authorizations, and the takeover token `Status:` |
| `afk_log_YYYY-MM-DD.md` | Decisions (tier, rationale, alternatives, reversal), **stalls and retries**, infrastructure events, messages |
| `BLOCKED_<YYYY-MM-DDTHH-MM>.md` | One per blocked decision: what's needed, the council's recommendation, exact current state |
| `afk_progress.md` | Only when dark. Append-only, tailable, heartbeated |
| `afk_handoff/` | Only during a takeover. Request, dump, sentinel, ack |

`afk_session.md` staying `ACTIVE` is what lets a fresh session resume under AFK
rules rather than assuming you're back because something got typed — you may
just be checking in from a phone.

## Build and deploy

```powershell
.\build.ps1              # → afk.skill
.\build.ps1 -Zip         # → afk.skill + afk.zip
.\deploy.ps1             # INSTALL into Claude Desktop on this machine
.\publish.ps1            # build, then copy to <Dropbox>\Skills
.\deploy.ps1  -DryRun    # either script; show what would happen, write nothing
.\publish.ps1 -DryRun
```

| Script | Target | Use when |
|---|---|---|
| `deploy.ps1` | Claude Desktop's local skills-plugin dir on **this machine** | You want to *use* the skill here |
| `publish.ps1` | `<Dropbox>\Skills` | You want the package on your other machines, or to upload it |

```bash
./build.sh               # same, on macOS / Linux / Git Bash
./build.sh --zip
```

`build.sh` and `build.ps1` produce **byte-identical archive contents**, so it
doesn't matter which one runs. Both validate before packaging and refuse to
build on a folder/frontmatter name mismatch, an illegal name, an empty or
>1024-char description, or the two version stamps disagreeing.

Both scripts build first, so a failed validation aborts them and a package that
doesn't validate never reaches Claude Desktop or the shared folder.

`deploy.ps1` is the **skeleton's** installer, adopted here: it auto-detects the
nested `skills-plugin\<guid>\<guid>\skills\` layout, mirrors with robocopy
`/MIR` **excluding `data\`** so runtime state survives a redeploy, keeps 5
backups, and registers the skill in `manifest.json` — Claude Desktop only loads
skills listed there, so a bare file copy is invisible. **Restart Claude Desktop
afterward.**

`publish.ps1` resolves Dropbox from `%LOCALAPPDATA%\Dropbox\info.json` and keeps
the last 5 packages in `Skills\.backups\`, since Dropbox syncs deletions too.
Override with `-Destination`.

For web and mobile, upload `afk.skill` via **Settings → Capabilities → Skills**
— that creates an account-synced copy which then takes precedence over local
files everywhere, including desktop. Install
[`council-skill`](https://github.com/pdrakeweb/council-skill) alongside it for
the medium-stakes path.

## Relationship to the skeleton

Built from scratch rather than from
[claude-skill-skeleton](https://github.com/pdrakeweb/claude-skill-skeleton) —
this is a pure-Markdown skill with no API, no cache, and no Python CLI, so the
skeleton's SQLite machinery has nothing to do. It keeps the skeleton
conventions that apply: two matching semver stamps validated at build time
(`SKILL.md` frontmatter + `afk/VERSION`), build-time name/description
validation, bump-on-every-commit, `git tag vX.Y.Z`. See [CLAUDE.md](CLAUDE.md).

## Prior art

Surveyed before writing. The existing "AFK" skills solve a **different**
problem — remote-controlling a session from your phone — and none of them
implement stakes tiers, a blocked-decision protocol, or a return briefing:

- [wickdninja/afk](https://github.com/wickdninja/afk) — drive sessions from
  Slack/iMessage. Complementary, not overlapping: pair it with this skill if
  you want a comms channel while away.
- [gmotyl/afk-claude-telegram-bridge-skill](https://github.com/gmotyl/afk-claude-telegram-bridge-skill) — same category, via Telegram.
- [allanninal/claude-code-skills](https://github.com/allanninal/claude-code-skills) `autonomous-skill` — multi-session task decomposition; no AFK semantics or decision governance.

## License

MIT — see [LICENSE](LICENSE).
