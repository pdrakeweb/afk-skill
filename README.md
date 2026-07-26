# afk-skill

A Cowork / Claude Code skill that governs autonomous work **while you are away
from keyboard**.

It is not a productivity maximizer. Its purpose is narrower and more useful:
make sure every decision taken in your absence is one you'd have agreed with —
and that every decision that isn't is still waiting for you, undamaged, when
you get back.

> The default when uncertain is **stop and log**, not proceed. Unfinished work
> is cheap. An unwanted irreversible action is not.

## Triggering

Say any of:

```
I'm going AFK for 3 hours
heading out, back around 5
I'll be offline until tomorrow morning
going dark until Monday — no comms
```

The skill extracts the duration if you give one and asks once if you don't.
On return, `"I'm back"` / `"what did I miss"` runs the return briefing.

## What it does

**1 — Pre-departure checklist.** Surfaces every pending decision that needs you
*before* you leave, each with a recommended default so you can clear them with
one word. Confirms the work queue (which becomes the scope boundary for the
whole absence), records any pre-authorizations **verbatim**, and notes what
you've reserved as off-limits.

**2 — Stakes tiers.** Every decision is classified before it is acted on. On a
boundary, it goes to the **higher** tier.

| Tier | Examples | Behaviour |
|---|---|---|
| **Low** | editing code, tests, refactors, research, local commits | Decide, act, log with rationale + reversal steps |
| **Medium** | architecture, new files/repos, new dependency, sending a message | Convene the [council skill](https://github.com/pdrakeweb/council-skill); act on the verdict; log the debate |
| **High / irreversible** | pushing to `main`, deleting data, money, publishing, credentials, production | **STOP.** Write `BLOCKED_<timestamp>.md` and move on |

There is **no confidence level that unlocks the high tier** — not even a
high-confidence council verdict. The council rules on what is *wise*, not on
what you have *authorized*.

**3 — Decision log.** Every decision (not every action) goes to
`afk_log_YYYY-MM-DD.md` with rationale, alternatives considered, and how to
reverse it. Logged *before* acting, so a crash mid-action leaves a recoverable
record of intent.

**4 — Comms.** Acknowledge-only by default: *"Pete is AFK until ~17:00. I've
flagged this for him."* Never agrees to a deadline, scope, price, or meeting;
never speaks to your availability or opinions; never initiates outbound contact.

**5 — Return briefing.** Blocked items first, always — never buried under a
list of accomplishments. Then items needing review, completed work, decision
counts, messages, and anything left in flight. Then it stops and waits: you
being back is not an approval.

## Council integration, and the fallback

Medium-stakes decisions are delegated to the
[`council`](https://github.com/pdrakeweb/council-skill) skill — three debaters
in clean parallel contexts (Advocate, Skeptic, Pragmatist) and a Judge who
rules with a confidence level. AFK reads the machine-readable verdict line and:

- **high / medium** confidence → proceed with the verdict's option
- **low** confidence → **do not proceed.** Block and defer. A low-confidence
  verdict is the council saying it lacks the information to decide; acting
  anyway throws away the only useful signal it produced.

**If the council skill is not installed**, medium-stakes decisions fall back to
best-effort: reason through the case-for, failure modes, and feasibility *in
writing*, act only at the easily-reversible end of medium, and log the entry
flagged `⚠ UNVERIFIED — single-model judgement only`. Every unverified decision
gets its own heading in the return briefing, so you can see at a glance exactly
which calls were made without adversarial review.

## Files it writes

Into the working directory, not the repo:

| File | Purpose |
|---|---|
| `afk_session.md` | Departure time, expected return, queue, pre-authorizations, `ACTIVE`/`CLOSED` |
| `afk_log_YYYY-MM-DD.md` | Every decision: tier, rationale, alternatives, reversal |
| `BLOCKED_<YYYY-MM-DDTHH-MM>.md` | One per blocked decision: what's needed, recommendation, current state |

`afk_session.md` staying `ACTIVE` is what lets a fresh session resume under AFK
rules rather than assuming you're back because something got typed.

## Install

```powershell
.\build.ps1          # → afk.skill
.\build.ps1 -Zip     # → afk.skill + afk.zip
```

Upload `afk.skill` via **Settings → Capabilities → Skills**, or drop the `afk/`
directory into your Cowork skills directory. Install
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
