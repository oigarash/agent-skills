# Minimal SKILL.md Authoring Quick-Reference

This is a fallback for when the `skill-creator` skill is **not** available. If `skill-creator`
is installed, use it instead — it has the full drafting/eval/optimization loop.

The point of this file is: you should still be able to draft or patch a SKILL.md well enough
to be useful, using only the notes below.

## 1. Anatomy

```
<skill-name>/
├── SKILL.md             (required; frontmatter + markdown instructions)
├── references/          (optional; deeper docs, loaded on demand)
├── scripts/             (optional; executable helpers the skill calls)
└── assets/              (optional; templates / static files the skill emits)
```

Progressive disclosure: the *frontmatter* is always in Claude's context; the *body* loads
when the skill triggers; `references/` and friends load only when SKILL.md points to them.

Keep SKILL.md under ~500 lines. If it grows beyond that, split into `references/*.md` and
link from SKILL.md with "read this when X" pointers.

## 2. Frontmatter (required)

```yaml
---
name: <skill-name>                 # required, kebab-case, matches directory
description: |                     # required; the primary triggering signal
  One-paragraph "what it does + when to use it". Be specific about the phrases / contexts
  that should trigger it. Skills tend to *under*-trigger, so write this slightly pushy.
  Include natural-language examples in whatever languages your user speaks.
metadata:                          # optional but recommended
  author: <name or handle>
  version: "0.1.0"
  repository: <url>                # optional; enables upstream feedback routing
---
```

The `description` field is doing ~80% of the work. If users complain "the skill didn't fire
when I asked X", that's almost always a `description` problem, not a body problem.

## 3. Writing the description

Good descriptions:

- State the concrete capability first ("Configure ghostty terminal via ~/.config/ghostty/config").
- Enumerate trigger phrases: the actual words a user would type. Include synonyms and
  multi-lingual variants if relevant.
- Include a negative clause when the skill overlaps with another ("Skip when the user is
  asking about Webex Calling — that is out of scope for this CLI.")
- Push a little: "Make sure to use this skill whenever the user mentions X, even if they
  don't explicitly ask for Y."

Bad descriptions are generic ("A skill for X"), under-specific, or only list the happy path.

## 4. Body shape that works

A typical SKILL.md flows as:

1. One short paragraph repeating the purpose (so the skill is self-contained when read in
   isolation).
2. **Mental model** / key concepts — give Claude a frame before procedure.
3. **Core workflow** as numbered steps, imperative voice ("Resolve the real path", not
   "You should resolve the real path").
4. **Templates / examples** pulled out into `references/` if they are long.
5. **Non-goals** — explicitly list what this skill does *not* do. Prevents over-triggering.

Prefer explaining *why* a step matters over heavy-handed `MUST` / `NEVER` caps. The model is
smart; an unexplained rule is a brittle rule. If you find yourself writing three `NEVER`s in
a row, step back and reframe as reasoning.

## 5. When to bundle a script vs inline instructions

Bundle a script in `scripts/` when:

- The step is deterministic and repeated across invocations.
- Doing it inline would cost tokens every time (e.g., parsing a large known file format).
- The script has been battle-tested.

Keep it inline when the logic shifts per invocation or you want the model to reason about it.

## 6. Minimum sanity check before shipping

Without a full eval loop, at least:

1. Read the SKILL.md top-to-bottom as if you've never seen it. Does the description tell you
   exactly when to use it? Do the steps make sense without outside context?
2. Mentally walk the skill through 2–3 realistic prompts, including one that is a near-miss
   (should *not* trigger).
3. If a step references an external command/path, verify the command exists and the flag
   names are current.
4. If the skill has a `references/` file, confirm SKILL.md actually points to it with a
   "read this when X" hint.

That's the floor. For anything higher-stakes, use `skill-creator`.

## 7. Common pitfalls

- **Stale examples**: command flags rename faster than you update docs. Mark examples with
  the version they were verified against, or link to the tool's own help.
- **Path leakage**: `/Users/<you>/...` baked into instructions makes the skill env-specific
  by accident. Use `$HOME` or describe the location abstractly.
- **Triggering on fumes**: if the description is just "<tool name>", it will fire on
  adjacent topics that happen to name the tool. Add concrete phrasings and a negative
  clause.
- **Over-bundled references**: every `references/*.md` you add is context you're asking
  Claude to pull in later. Only split out what's genuinely heavyweight.
