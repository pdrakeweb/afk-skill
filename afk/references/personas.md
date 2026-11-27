# Personas

A persona is the voice and style the session uses for outbound replies while
the user is away (§7). It is communication style only.

**What a persona is not:**

- It never changes a tier. High-tier blocks still apply exactly as written
  (`references/tiers.md`); no persona, and no council verdict, authorizes
  anything above its tier.
- It never speaks as the user, and never as the person a corpus persona was
  derived from. Every reply still comes from the agent acting for the user,
  and still states the user is away.
- It applies to outbound messages only. Logs, the return briefing, sub-agent
  briefs, and `BLOCKED_` files stay in the skill's plain voice.

## Where personas live

`personas/` in the skill directory, one file per persona. Filename is the
persona name, lowercase with dashes: `personas/concise.md`.

Schema — every persona file carries these fields (see
`personas/_template.md` for the blank form):

```markdown
# Persona: <name>
- **Description:** <one line — who this voice is for>
- **Tone:** <register, formality, warmth>
- **Length:** <short / medium — default short>
- **Openings:** <how replies open>
- **Closings:** <how replies close>
- **May acknowledge:** <which message kinds it may answer>
- **Never says:** <hard limits — commitments, opinions, specifics>
- **When it can't help:** <the fallback line shape>
- **Derived from:** <interview | corpus: what, where from | example>
```

Files starting with `_` are not personas (`_template.md` is the blank form).
A persona file must not contain credentials, verbatim private content, or
anything beyond a first name identifying a corpus author. It is style
guidance, not quoted mail.

## Onboarding

Onboarding runs while the user is present — standalone, when they say "set
up a persona", "add a persona", or "onboard a persona", or as part of
departure (§1.8) if they accept the offer. The user picks one path per
persona. Either path ends the same way: a persona file, read back to the
user, saved only after they approve it.

### Path A — interview

Three batches, same shape as the departure sequence (§1.2/§1.3): one block,
recommended defaults, answerable with "defaults".

**Batch 1 — identity:**

```markdown
**Persona setup, 1 of 3 — who is speaking:**
1. Name for this persona? *(default: concise)*
2. Role — whose assistant is replying, and to whom? *(default: the user's
   assistant, replying to anyone who messages while they're out)*
3. How should replies sign off, if at all? *(default: no sign-off)*

Reply with numbers to change, or "defaults".
```

**Batch 2 — voice:**

```markdown
**Persona setup, 2 of 3 — how it sounds:**
1. Tone — plain, warm, formal, dry? *(default: plain and direct)*
2. Length — one sentence, two sentences, short paragraph? *(default: one or
   two sentences)*
3. Openings and closings — any fixed phrases? *(default: none — lead with
   the acknowledgement)*

Reply with numbers to change, or "defaults".
```

**Batch 3 — reply rules:**

```markdown
**Persona setup, 3 of 3 — what it may and may not do:**
1. Which messages may it acknowledge? *(default: any inbound message that
   isn't a commitment request)*
2. What must it never say? *(default: no deadlines, prices, decisions, or
   opinions — and nothing about the user's availability beyond "away until
   ~<time>")*
3. When it can't help, what shape does the fallback take? *(default:
   acknowledge + "I've flagged this for them — they'll pick it up when
   they're back.")*

Reply with numbers to change, or "defaults".
```

Then draft the persona file from the three batches, read it back in full,
and save it only on approval. Corrections are edits to the draft, not a
restart — re-ask only the batch the correction touches.

### Path B — corpus

The user supplies a corpus to build the persona from: emails from a
particular individual, docs they have written, public articles, or similar.
Record what was supplied and where it came from.

1. **Read the corpus.** Derive: vocabulary, sentence shape, tone/register,
   stance toward requests, typical reply patterns (openings, closings,
   hedging), and what the author never does.
2. **Draft the persona file** from the derivation. Mark it
   `Derived from: corpus — <what, where from>`.
3. **Present the draft for approval or correction before saving.** Do not
   save an unapproved draft. Corrections are edits to the draft.
4. **Thin corpus — say so.** If the corpus is too small or too uniform to
   derive a voice (under ~10 substantive messages, or a single short doc),
   say so plainly and fall back to the interview for the missing pieces.
   Never pad a thin derivation with invented traits.
5. **Corpus hygiene.** Derive style only — never copy verbatim private
   content into the persona file. If the corpus is someone else's writing,
   the persona borrows their manner, never their identity: replies still
   come from the agent acting for the user, and never claim to be that
   person. Only use a corpus the user provides; never go looking for
   someone's writing unasked.

If the derivation and the user's corrections disagree, the user's corrections
win. The file is theirs; the corpus is evidence, not authority.

## Selection at departure

At departure (§1.8): if personas exist, ask which one this session uses, or
none. Default is none — plain acknowledge-only, the current behaviour. The
choice is fixed for the session and recorded in `afk_session.md`. A persona
does not change the comms posture (§1.6): "no comms" from the user still
means acknowledge and log, in whatever voice is active.

## Add, edit, remove

- **Add:** run onboarding again; a new file lands in `personas/`.
- **Edit:** the user says what to change; edit the file; read it back.
  An edit takes effect from the next session, never mid-session.
- **Remove:** delete the file. If it was the only persona, the next departure
  offers onboarding again. A session never references a deleted persona.

## Guardrails (summary)

- Persona is style only. Tiers, blocks, and the human-approval guard (§3, §4)
  are unchanged and unchangeable by any persona.
- Never present the session as the user, or as the corpus author. The first
  line of any reply states the user is away.
- Never make a commitment in any voice. A persona that would agree to a
  deadline is misconfigured — fix the persona, do not send the reply.
- Never write corpus content into a reply. The persona file holds style, not
  source material.
