# Upstream Issue Template

Use this when filing a *generic* improvement about an Agent Skill to its upstream repo.
Keep the filled Issue short — maintainers skim. Every section below has a rough word budget;
respect it.

## Title

`[skill: <skill-name>] <one-line observed → expected>`

Example: `[skill: webex-cli] description never triggers on 「Webex スペース一覧」`

If the skill lives in a monorepo (i.e. SKILL.md declares `metadata.path`), prefix with the
subpath so the maintainer can jump straight to the file:
`[skill: skills/webex-cli] …`.

## Body

```markdown
### Summary
<1–2 sentences. What breaks, for whom.>

### Observed behavior
<Quote the exact line(s) from SKILL.md / references that caused the problem, or describe how
the skill's description failed/over-triggered. Paste the minimal excerpt — not the whole
file.>

### Expected behavior
<What the skill should have said or done instead. Be specific enough that a reviewer can
diff it in their head.>

### Proposed change to SKILL.md (optional)
<If you have concrete wording, show a small diff. Leave blank if you'd rather let the
maintainer decide the phrasing.>

~~~diff
- old line
+ new line
~~~

### Session context
- Date: YYYY-MM-DD
- Claude variant / model: <e.g., Claude Opus 4.7 in Claude Code>
- Skill version (from SKILL.md metadata, if present): <version>
- Host environment (only if relevant): <OS / tool versions>

### Reproduction
<Shortest sequence another user could run to see the same issue. Scrub personal paths,
internal URLs, and session-only state. Skip this section if the finding is purely about
SKILL.md wording (no reproduction needed).>

### Is this maybe env-specific?
<Short self-check: any chance this only hits my setup? If yes, note it so the maintainer
can redirect the Issue without guessing.>
```

## Labels

Suggest these when supported; drop any the repo doesn't use:

- `skill-feedback` (so the maintainer can filter)
- `docs` if the fix is a SKILL.md/reference text change
- `triage` if you want the maintainer to confirm scope first

## What *not* to include

- Full chat transcripts. Quote only the line that reproduces the issue.
- Tokens, cookies, session IDs, internal URLs, coworker names.
- Speculation about unrelated skills.
- A demand for a specific implementation unless you have strong evidence the maintainer
  wants a PR — Issue first, PR only if invited or the fix is trivial text.
