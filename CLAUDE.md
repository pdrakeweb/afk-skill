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
- **major** — changes to the tier boundaries, the `BLOCKED_` file contract, or
  how a council verdict is consumed

Then `git tag vX.Y.Z`.

## The contract with council-skill

`afk` parses the last line of a council run:

```
COUNCIL_VERDICT: <option> | CONFIDENCE: <high|medium|low>
```

If council-skill changes that format, this skill breaks silently — the
medium-stakes path would fall through to the unverified fallback without
saying why. Any change to that line is a **major** bump in *both* repos, made
in the same session.

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
