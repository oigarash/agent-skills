# Upstream PR Template

Open a PR/MR only when **all** of the following hold:

- The fix is a small, self-contained text change in SKILL.md or a reference file.
- The change does **not** touch the `description:` frontmatter field (triggering is sensitive
  — let the maintainer tune).
- You have explicit user authorization to push and open the PR/MR.
- No existing Issue covers it, or there is an Issue and you are linking to it.

Otherwise, file an Issue from `issue-template.md` and stop.

## Branch, commit, push

```bash
# From the resolved upstream clone (not a symlink into ~/.claude)
git -C "$REPO" switch -c skill-feedback/<short-slug>
# ... Edit ...
git -C "$REPO" commit -am "skill(<skill-name>): <imperative one-line>"
git -C "$REPO" push -u origin HEAD
```

Never force-push. Never commit with `--no-verify`.

## Title

`skill(<skill-name>): <imperative one-line summary>`

## Body

```markdown
### Motivation
<1–2 sentences. Why this change is needed. Link the Issue if one exists.>

Closes #<issue-number>   <!-- or: Refs #<issue> -->

### Change
<Plain-English description of the text diff. One short paragraph.>

### Why this is safe / scoped
- Touches only <files>
- Does not modify `description:` frontmatter
- No change to executable scripts or assets

### Verified
- Read through SKILL.md after the edit and confirmed the quoted sections are consistent.
- <If applicable> Re-ran the skill on the original task and it now behaves as expected.

### Out of scope
<Anything adjacent you noticed but deliberately did NOT fix in this PR.>
```

## After opening

- Paste the PR/MR URL back to the user.
- If you opened an Issue first, post a short comment on the Issue linking the PR.
- Do **not** merge. Wait for the maintainer.
