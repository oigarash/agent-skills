---
name: skill-feedback
description: |
  Turn friction you noticed while using another Agent Skill into actionable feedback — either an
  upstream GitLab/GitHub Issue/PR when the fix is generic, or a local SKILL.md patch when the
  fix is environment-specific. Use this skill whenever the user says things like
  「このスキルおかしい」「skill の説明が足りない」「skill を改善したい」「skill issue 立てて」
  「skill にフィードバック」「/skill-feedback」, or any mention of fixing, reporting, or
  improving an Agent Skill. Also trigger proactively whenever you (Claude) notice during a
  session that a skill misled you, lacked a needed step, referenced a wrong path, fought with
  tool output, or its description did/did-not trigger when it should have — surface this skill
  and offer to capture the finding. It also covers deciding whether a finding is generic
  (upstream) or env-specific (local), locating the upstream repo from SKILL.md metadata or
  `git remote`, and filing via `glab` / `gh` from a pre-written template with dry-run and
  duplicate checks.
metadata:
  author: oigarash
  version: "0.1.0"
  repository: https://github.com/oigarash/agent-skills
---

# Skill Feedback

Capture a concrete improvement you noticed about an Agent Skill in the current session, and
route it correctly: upstream Issue/PR if it is a generic defect, or a local SKILL.md patch if
it is env-specific. Defer all *skill authoring details* to the `skill-creator` skill when it
is available — this skill focuses on **triage, routing, and filing**.

## Mental Model

Every finding is one of three things. Decide which before touching any tool.

| Finding kind | Example | Destination |
|---|---|---|
| Generic defect in SKILL.md / reference / script | description never triggers on the natural phrasing; command example is wrong for everyone | Upstream Issue (default) or PR |
| Env- or project-specific gap | path `/Users/<me>/work/X` baked into instructions; an internal URL only you can reach | Local/user SKILL.md patch |
| Not-actually-a-skill-problem | user prompt was ambiguous; a different skill should have handled it | Not filed. Note it back to the user and stop |

If unsure, prefer treating as generic and draft an Issue, but include a "is this just my
environment?" note in the body so the maintainer can redirect.

## Core Workflow

Work through these steps in order. Don't skip Step 0 — filing against the wrong skill wastes
maintainer time.

### Step 0. Confirm the target skill

State your best guess at which skill the finding is about, plus the observed location, and ask
the user to confirm before going further. Typical sources of the guess:

- The most recently invoked skill in this session (check tool-call history / the `<command-name>`
  tags you've seen).
- Explicit mention by the user.
- A matching name/keyword in the list of available skills.

If there are multiple candidates, list them. Never silently pick.

### Step 1. Resolve the skill directory and read its SKILL.md

Agent Skills may live in several places and are often symlinked. Always resolve to the real
path before reading or editing:

```bash
# Common locations to probe, in priority order
ls -la ~/.claude/skills/<name>/SKILL.md                # user-level
ls -la ~/.claude/plugins/**/skills/<name>/SKILL.md     # plugin-provided
ls -la ./.claude/skills/<name>/SKILL.md                # project-local

# Then resolve the real file (handles symlinks)
readlink -f ~/.claude/skills/<name>/SKILL.md
```

Read the resolved SKILL.md once so you can quote the *current* description/instructions in
both Issue bodies and local diffs. Feedback referencing stale content is worse than no
feedback.

### Step 2. Write down the finding, precisely

Before worrying about routing, pin the finding down in 3 lines:

- **Observed**: what the skill told you to do / how it described itself.
- **Actual**: what happened, or what should have happened.
- **Impact**: who is affected and when. This is what decides generic vs env-specific.

Reject vague findings ("the skill is confusing"). Push back on yourself until you have a
reproducible, specific delta. If you can't, tell the user and stop — unactionable Issues
aren't helpful.

### Step 3. Classify: generic vs env-specific

Use the table above. Concrete signals:

- **Generic** (file upstream): the finding would reproduce for *any* user who installs this
  skill from the repo — wrong flag name, broken reference link, description that misfires on
  common phrasing, missing prerequisite note, outdated API example.
- **Env-specific** (local patch): finding depends on the user's installed tools/versions,
  company intranet, personal paths, project conventions, or a non-upstreamed customization.

If generic, continue to Step 4. If env-specific, jump to Step 6.

### Step 4. Locate the upstream repository

Try these in order and stop at the first success. If all fail, the skill has no upstream →
degrade to Step 6 (local patch) and tell the user.

1. **SKILL.md frontmatter `metadata.repository`**. Preferred. Also check common aliases:
   `repository`, `repo`, `upstream`, `source`.
2. **`git remote -v` from the skill directory.** Run at the resolved path, not the symlink:
   ```bash
   git -C "$(readlink -f ~/.claude/skills/<name>)" remote -v 2>/dev/null
   ```
   If it's a git repo, pick `origin` (or `upstream` if present) and take its URL.
3. **Ask the user.** Verbatim: "このスキルの upstream リポジトリを教えてください（無ければ
   ローカル SKILL.md の修正提案に切り替えます）。"

Normalize whatever you find to `host + owner/repo`. Host decides the CLI:
- `github.com` → `gh`
- `gitlab.com` / any GitLab host → `glab`
- Anything else → ask the user which CLI to use, or file manually.

### Step 5. File upstream (Issue default, PR only when appropriate)

**Default to Issue.** PRs are only appropriate when:

1. The change is a small, self-contained text tweak in SKILL.md or a reference doc.
2. It is not touching the `description:` field (triggering is sensitive; let the maintainer
   tune — open an Issue describing the miss instead).
3. The user has explicitly asked for a PR, or you have clear authorization from the user.

Otherwise, file an Issue and let the maintainer decide.

#### 5a. Duplicate check

Before drafting, search for an existing Issue on the same topic. If found, propose
*commenting on it* instead of opening a new one.

```bash
# GitHub
gh issue list --repo <owner/repo> --state all --search "<3-5 keywords>"

# GitLab
glab issue list --repo <owner/repo> --search "<3-5 keywords>"
```

If a likely-duplicate Issue exists, show it to the user (number, title, state), offer to
post a short "+1 / additional repro" comment under the same Safety rails (preview →
confirm), and skip opening a new Issue. Only open a new one if the user confirms the
existing Issue is about a different problem.

#### 5b. Draft from the template

Load `references/issue-template.md` and fill it in. Keep it tight — reviewers skim.

#### 5c. Sanitize before sending

Strip, or ask the user to confirm keeping:
- absolute paths containing a username (`/Users/<me>/...`, `/home/<me>/...`)
- API tokens, cookies, session IDs, email addresses, coworker names
- internal URLs or company-confidential product names, unless the upstream is also internal
- long prompt excerpts — quote the shortest snippet that reproduces the finding

#### 5d. Preview → confirm → file

Follow the Safety rails section below — preview every time, proceed only on explicit OK,
honor dry-run requests by stopping after the preview.

Only after the user confirms, run:

```bash
# GitHub Issue
gh issue create --repo <owner/repo> --title "<title>" --body "$(cat <<'EOF'
<body>
EOF
)" --label "skill-feedback"

# GitLab Issue
glab issue create --repo <owner/repo> --title "<title>" --description "$(cat <<'EOF'
<body>
EOF
)" --label "skill-feedback"
```

Return the created Issue URL to the user. If a PR path was chosen, see
`references/pr-template.md` — branch off upstream, apply the text edit, push, open the MR/PR,
and link back to any related Issue.

### Step 6. Env-specific path: propose a local SKILL.md patch

When the finding is genuinely env-specific, don't file upstream — patch the local copy. The
user explicitly asked for this routing in the initial design.

1. **Resolve the real file**: `readlink -f <path-to-SKILL.md>`. If it points outside
   `~/.claude/skills/` (e.g., into `~/work/<project>/skills/<name>/`), that is the file you
   will edit; warn the user so they know the change lands in that project repo, not in
   `~/.claude/`.
2. **Prepare a unified diff** and show it to the user before touching anything.
3. **Wait for explicit confirmation**, then apply with `Edit`. Do not `Edit` first and
   apologize later — SKILL.md is load-bearing for future sessions.
4. **Summarize the applied change in one line** and suggest reloading the skill list if your
   harness caches it.

If the skill is a symlink pointing to a git repo with a non-pushed branch, note that the
change will appear in `git status` in that repo — give the user a heads-up so they can commit
or stash intentionally.

### Step 7. Close the loop in this session

Add one line to the session notes / TODO telling the user what was filed or patched and
where. This keeps the finding from being lost if the conversation continues.

## Safety rails (applies to every filing or edit)

These apply to Step 5 (upstream filing) and Step 6 (local patch) alike. They exist because
every action in this skill leaves a visible artifact outside the current session — an Issue
someone gets notified about, or a file future sessions will read.

- **Always preview first.** Before any `gh`/`glab`/`Edit` call, show the user: target (repo
  or file path), title, full body or unified diff, and labels/branch. The preview is the
  default; confirmation turns it into an action.
- **Require explicit confirmation.** Proceed only on an unambiguous yes ("OK", "go", "はい",
  「出して」). Silence or ambiguous replies ("looks interesting", "maybe") do not authorize
  filing. Restate and re-ask.
- **Honor dry-run / preview-only requests.** If the user says「dry run」「プレビューだけ」
  「まだ投げないで」, print the preview and stop — do not ask for confirmation, do not file.
- **One artifact per finding.** Don't batch multiple findings into one Issue/PR or one diff.
  If the user has N findings, run the workflow N times.
- **No stealth edits.** For local SKILL.md changes, never `Edit` before showing the diff and
  getting a yes — even for "tiny" fixes. SKILL.md is read by every future session.
- **Respect symlink realities.** If the target file resolves into a git repo outside
  `~/.claude/`, tell the user *before* editing so they know which repo will show a diff.
- **Duplicate check is mandatory for upstream.** Always run the search in Step 5a; never
  skip because "it's probably fine."
- **Sanitize.** Apply the scrub list in Step 5c to both Issue bodies and commit messages.
  When in doubt, ask the user.

If any rail can't be satisfied (e.g., you can't reach the upstream to search for duplicates),
stop and tell the user — don't file blind.

## Repository URL normalization

Upstream URLs come in many shapes. Normalize before feeding to `gh` / `glab`:

- `git@github.com:owner/repo.git` → `owner/repo`, host `github.com`
- `https://github.com/owner/repo` / `.git` → same
- `git@gitlab.example.com:group/subgroup/repo.git` → `group/subgroup/repo`, host
  `gitlab.example.com` (needs `glab auth` pre-configured for that host)
- A bare `owner/repo` is ambiguous — ask.

## Triggering proactively during a session

Watch for these signals while helping the user. When you spot one, *pause* and offer to
invoke this skill — don't derail the task mid-flight, but mention it before moving on:

- A skill's description fired but the instructions didn't actually cover the user's ask.
- A skill's description *didn't* fire on a task it is clearly about.
- A referenced command/path/flag in SKILL.md doesn't exist on the user's machine in the way
  the skill describes, and a simple version-skew fix would help everyone.
- The user expresses frustration with a skill ("また違うじゃん", "これ間違ってない？").

Phrasing: "いまの件、`<skill-name>` の SKILL.md に起因しそうです。後でまとめて
skill-feedback で Issue 化しておきますか？" Don't pivot immediately unless the user agrees.

## Authoring / improving skills

Detailed guidance on writing or rewriting a SKILL.md (frontmatter design, progressive
disclosure, test-driven iteration) lives in the `skill-creator` skill. If `skill-creator` is
available in this environment, delegate to it for any *drafting* or *major rewrite* work;
this skill only files the feedback.

If `skill-creator` is **not** available, read `references/authoring-quickref.md` — it carries
the minimum you need to write or patch a SKILL.md defensibly without the full creator loop.

## Non-goals

- Running automated test suites / eval harnesses against a skill. If you need that, use
  `skill-creator` (it has an eval viewer and benchmark flow).
- Bulk-auditing all installed skills. This skill is per-finding, driven by something you
  actually observed in-session.
- Filing Issues on skills without a known upstream. Always degrade to a local patch or stop.

## Quick reference: files in this skill

- `SKILL.md` — this file; triage and routing logic.
- `references/issue-template.md` — Issue body template for upstream filings.
- `references/pr-template.md` — PR body template (opt-in path only).
- `references/authoring-quickref.md` — minimum SKILL.md authoring guide for when
  `skill-creator` isn't available.
