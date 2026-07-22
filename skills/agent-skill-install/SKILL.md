---
name: agent-skill-install
description: >
  Rules for installing and organizing global Agent Skills without duplicating skill bodies.
  Use whenever adding, installing, or placing an Agent Skill so it is available across all AI
  CLIs, deciding where a new skill's source of truth should live, or auditing the global skills
  directory — e.g. "install this skill globally", "add a skill", "make this skill global",
  "スキルをグローバルに入れる", "どこにスキルを置く", or symlink-vs-copy questions. Enforces one
  rule: author the skill in its home repo, then surface it globally via `npx skills` or a
  symlink — never a hand-copied body.
metadata:
  type: reference
  repository: https://github.com/oigarash/agent-skills
  version: 0.1.0
---

# Installing & organizing global Agent Skills

The one principle: **a globally-available skill must have exactly one source of truth, and the
global skills directory must never hold a hand-copied body of it.** Everything below follows
from that. Duplicated bodies drift, have no update path, and rot silently — so we only ever put
a *reference* (an npx-managed install or a symlink) into the global directory.

## How the setup works (read this first)

- **Source of truth for global skills:** `~/mac_setup/ai-unified-config/skills/`. This directory
  IS the global skill set.
- **Distribution:** every per-tool skills dir is a *whole-directory symlink* to that source:
  `~/.claude/skills`, `~/.agents/skills`, `~/.codex/skills`, `~/.cursor/skills` all point at it.
  So anything present in the source is instantly visible to Claude Code, Codex, Cursor, and the
  generic agents runtime — there is no per-skill sync step to run after a change.
- Therefore "make a skill global" == "have an entry for it in the source directory", and that
  entry must be one of the two allowed forms below.

## Two allowed forms for an entry in the source directory

| Form | What it is | When | Updated by |
|---|---|---|---|
| **npx-managed install** | A body placed by `npx skills add` from a registry or git URL | The skill's home repo is reachable by npx (public registry, or a git URL you can clone) | `npx skills update <name>` |
| **symlink** | A symlink to the skill's home-repo checkout on disk | The skill lives inside a tool/monorepo checkout you already have locally, or in a path npx can't/shouldn't install from | editing the home repo (reflected immediately) |

**Never** a plain, hand-copied directory with no upstream link. That is the one thing this skill
exists to prevent. An npx-managed body is a copy too, but it is allowed because it is *tracked* —
`npx skills update` re-fetches it. A symlink is preferred when you want edits in the home repo to
show up with zero extra steps.

Prefer **npx** when the home repo has a clone-able git URL (it stays updatable and portable to a
fresh machine). Fall back to a **symlink** when npx would force a redundant re-clone of something
you already have as part of a larger checkout, or the source is a local/private path.

## Where a skill's source of truth should live (author here)

Before installing, make sure the skill actually lives somewhere with an upstream. Route by what
the skill is:

- **General / personal skill** → `github.com/oigarash/agent-skills` (this repo), under
  `skills/<name>/`.
- **Cisco-internal skill** → `gitlab-cxj.cisco.com/oigarash/agent-skills`, under `skills/<name>/`.
  (See that repo's `AGENTS.md`/`CLAUDE.md` for its conventions: English-only content, required
  `metadata.repository` + `metadata.version`, short descriptions.)
- **A skill that documents or ships with a specific tool/system you built** → keep it *inside that
  tool's own repo* under `skills/<name>/`, so it is versioned and released with the tool. Examples
  in use: `csone-cli`, `sherlock-cli`, `webex-cli`, `circuit-cli`, `bdb-cli`, `cdets-cli`,
  `topic-cli`, and `yt-cli` (in `youtube-manager`). These reach the global set via **symlink**.
- **Someone else's public skill** → do not author or copy it; install it with `npx skills` and let
  the upstream own it (e.g. `find-skills`, `grill-me`, `grill-with-docs`, `playwright-cli`).

If a skill has no home yet, give it one first (usually this personal repo) rather than dropping a
loose copy into the global directory.

## Procedures

### Install a skill globally via npx (preferred)

The skill must be pushed to its home repo first (npx fetches from the remote).

```bash
# From a git URL (note the trailing .git). -g = global.
npx skills add -g git@github.com:oigarash/agent-skills.git --skill <name> -a claude-code

# Cisco-internal repo over SSH
npx skills add -g git@gitlab-cxj.cisco.com:oigarash/agent-skills.git --skill <name> -a claude-code
```

Because the per-tool dirs are one symlinked source, a single global install covers every tool.
Passing a plain HTTPS repo *page* (without `.git`) makes npx look for a
`/.well-known/agent-skills/index.json` endpoint and fail with "No skills found" — always use the
git URL form. Update with `npx skills update <name>`, remove with `npx skills remove <name>`.

### Install via symlink (fallback)

Use when the skill already exists in a local checkout (a tool's repo, `~/ghq/...`, `~/work/...`)
and npx would just re-clone it. Symlink the home-repo copy into the source directory:

```bash
SRC="$HOME/mac_setup/ai-unified-config/skills"
TARGET="<absolute path to the skill dir in its home repo, e.g. ~/ghq/.../<tool>/skills/<name>>"

# Guard: never remove an existing entry unless the target really has the skill.
[ -e "$TARGET/SKILL.md" ] || { echo "target missing SKILL.md: $TARGET"; exit 1; }
ln -s "$TARGET" "$SRC/<name>"
[ -e "$SRC/<name>/SKILL.md" ] && echo "linked OK" || echo "dangling — check the path"
```

Use absolute paths for the symlink target (matches the existing entries). No `npx skills sync`
is needed afterward — the whole-dir distribution symlinks pick it up immediately.

### Audit the global directory (catch orphan copies)

Every entry should be a symlink or an npx-managed install — never a plain, untracked copy.

```bash
SRC="$HOME/mac_setup/ai-unified-config/skills"
for n in "$SRC"/*/; do
  name=$(basename "$n")
  if [ -L "$SRC/$name" ]; then
    echo "symlink  $name -> $(readlink "$SRC/$name")"
  else
    echo "PLAIN    $name   # ok only if npx-managed (find-skills/grill-*/playwright-cli); otherwise it should be a symlink to its home repo"
  fi
done
```

A `PLAIN` entry that is *not* one of the known npx-managed public skills is a smell: it is likely
a hand-copied body that has drifted from — or has no — home. Fix it by finding/creating its home
repo, then replacing the copy with a symlink (or an npx install). To confirm a skill is available
via npx, search the registry: `curl -s "https://skills.sh/api/search?q=<name>"` and look for an
exact `skillId` match.

### Reconcile a copy that diverged from its home

If a `PLAIN` copy and its home repo differ, decide which is authoritative (usually the home repo /
tool project — it is where the skill is developed and often newer). Move any copy-only edits into
the home repo, commit them there, then replace the copy with a symlink. Do not silently discard
local edits without checking.

## Quick decision flow

```
Need a skill available globally?
│
├─ Is it someone else's public skill?
│    └─ npx skills add -g <registry/git-url> --skill <name>     (upstream owns it)
│
├─ Does it belong to a tool/system you built?
│    └─ keep it in that tool's repo (skills/<name>) → symlink into the source dir
│
└─ Is it your own general/Cisco skill?
     ├─ author it in oigarash/agent-skills (github = personal, gitlab-cxj = Cisco)
     ├─ push it
     └─ npx skills add -g <that repo .git> --skill <name>       (preferred)
          └─ or symlink if you already have the checkout and don't want a re-clone

In every branch: the source directory gets a reference, never a hand-copied body.
```
