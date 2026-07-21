---
name: customize-claude-code
description: >
  Practical reference for customizing Claude Code: hooks (PreToolUse, PostToolUse, Stop, SessionStart, etc.),
  settings.json configuration, CLAUDE.md conventions, and environment variables.
  Use this skill when creating or debugging hooks, configuring settings.json permissions or hooks,
  writing CLAUDE.md files, or troubleshooting any Claude Code customization.
  Trigger on: "hook", "PreToolUse", "PostToolUse", "settings.json", "CLAUDE.md setup",
  "customize Claude Code", "Claude Code configuration", "permission settings",
  "hookが動かない", "hookのデバッグ", "settings.jsonの設定".
---

# Customize Claude Code

Practical tips and gotchas for customizing Claude Code, distilled from real implementation experience.

## Hook System Overview

Hooks are shell commands that Claude Code executes at specific lifecycle events. They let you intercept, validate, or augment Claude's actions without modifying Claude itself.

### Hook Event Types

**Tool execution events** (most commonly used):

| Event | When it fires | Can block? | Key use case |
|-------|--------------|------------|-------------|
| `PreToolUse` | Before a tool executes | Yes (`deny`) | Gate/validate tool calls, auto-allow known-safe calls |
| `PostToolUse` | After a tool executes successfully | Yes (`block`) | Audit, trigger side-effects, inject feedback |
| `PostToolUseFailure` | After a tool fails | Yes (`block`) | Error handling, retry logic |

**Session & turn events**:

| Event | When it fires | Can block? | Key use case |
|-------|--------------|------------|-------------|
| `Stop` | When Claude is about to stop responding | Yes (`block`) | Quality gates, force continuation |
| `SubagentStop` | When a subagent is about to stop | Yes (`block`) | Subagent quality gates |
| `UserPromptSubmit` | After user submits a prompt | Yes (`block`) | Pre-processing, context injection |
| `SessionStart` | When a session begins | No | Environment setup, context injection (stdout → context) |
| `SessionEnd` | When a session ends | No | Cleanup, reporting |

**Other events**:

| Event | When it fires | Matcher on | Key use case |
|-------|--------------|-----------|-------------|
| `Notification` | When notification generated | type (`permission_prompt`, `idle_prompt`, etc.) | External alerting |
| `PreCompact` / `PostCompact` | Before/after context compaction | `manual` / `auto` | Save/restore context |
| `FileChanged` | Watched file changes on disk | filenames (e.g., `.envrc\|.env`) | Hot-reload config |
| `ConfigChange` | settings.json changes mid-session | scope (`user_settings`, `project_settings`, etc.) | Dynamic reconfiguration |
| `CwdChanged` | Working directory changes | — | Re-evaluate environment |

### Critical Behavioral Differences

**PreToolUse deny vs PostToolUse block** — this is the most important distinction:

- **PreToolUse `deny`**: Tool execution is **prevented**. The current mode (e.g., plan mode) is **preserved**. Claude receives the reason and can retry.
- **PostToolUse `block`**: Tool **already executed**. The block message is sent to Claude, but the action happened. Mode transitions (like leaving plan mode) are **irreversible**.

Real-world consequence: if you need to gate `ExitPlanMode` and keep Claude in plan mode during iteration, you **must** use PreToolUse. PostToolUse fires too late — plan mode is already exited.

## Hook I/O Contract

### Input (stdin JSON)

Every hook receives a JSON object on stdin:

```json
{
  "tool_name": "ExitPlanMode",
  "tool_input": { "plan": "..." },
  "session_id": "abc-123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/Users/you/project"
}
```

Fields vary by event type:
- `tool_name` / `tool_input`: Present for `PreToolUse` and `PostToolUse`
- `tool_output` / `execution_time_ms`: `PostToolUse` only
- `session_id`: Always present (also available as `CLAUDE_SESSION_ID` env var)
- `transcript_path`: Path to the session's JSONL transcript
- `permission_mode`: Current mode (`default`, `plan`, `auto`, etc.)
- `prompt`: `UserPromptSubmit` only

### Output (stdout JSON)

**PreToolUse** — permission decision:
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "review passed"
  }
}
```

Valid `permissionDecision` values:
- `"allow"` — tool executes
- `"deny"` — tool blocked, reason shown to Claude
- `"ask"` — prompt user for manual decision

PreToolUse can also **rewrite tool input** (e.g., sanitize a Bash command):
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "updatedInput": {
      "command": "npm test -- --safe-mode"
    }
  }
}
```

Any hook can inject context visible to Claude via `additionalContext`:
```json
{
  "decision": "allow",
  "additionalContext": "Reminder: the user prefers TypeScript strict mode"
}
```

**PostToolUse / Stop** — block decision:
```json
{
  "decision": "block",
  "reason": "Findings require revision"
}
```

**No stdout + exit 0** = allow (implicit pass-through for PostToolUse/Stop).

### Exit Codes

| Exit | Meaning |
|------|---------|
| 0 | Success — stdout JSON is processed |
| 2 | Blocking error — shown to user as hook failure |
| Other | Non-blocking error — logged but execution continues |

## settings.json Configuration

### File Locations (priority order)

1. **Project-level**: `<project-root>/.claude/settings.json` (highest priority)
2. **User-level**: `~/.claude/settings.json`
3. **Enterprise**: managed externally

**Common mistake**: putting project settings in `~/.claude/projects/<sanitized-path>/settings.json`. That path is for Claude's internal project state — **your project hooks go in `<project>/.claude/settings.json`**.

### Hook Configuration Format

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "ExitPlanMode",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/my-gate.sh",
            "timeout": 300
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/my-stop-gate.sh",
            "timeout": 300
          }
        ]
      }
    ]
  }
}
```

- `matcher`: regex matched against tool name. Empty string = match all. Case-sensitive. For MCP tools: `mcp__github__.*`.
- `timeout`: seconds. Default is 600 (10 min). Set appropriately for hooks that call external APIs.
- `if`: optional permission-rule filter (e.g., `"if": "Bash(git *)"` — only fires for git commands, not all Bash).

### Hook Types (Beyond Command)

Besides `"type": "command"`, there are three more types:

**Prompt hooks** (single-turn LLM evaluation):
```json
{
  "type": "prompt",
  "prompt": "Check if the plan addresses all edge cases. Respond with JSON: {\"ok\": true} or {\"ok\": false, \"reason\": \"...\"}",
  "model": "haiku"
}
```

**Agent hooks** (multi-turn with tool access, up to 50 turns):
```json
{
  "type": "agent",
  "prompt": "Run the test suite and verify all tests pass. $ARGUMENTS",
  "model": "haiku",
  "timeout": 120
}
```

**HTTP hooks** (webhook POST):
```json
{
  "type": "http",
  "url": "https://hooks.example.com/api/hook",
  "headers": { "Authorization": "Bearer $MY_TOKEN" },
  "allowedEnvVars": ["MY_TOKEN"]
}
```

### Hook Execution Rules

- Multiple matching hooks **run in parallel** (not sequential)
- Identical hook commands are deduplicated (same command + same matcher = runs once)
- When multiple hooks return decisions, **most restrictive wins** (deny > ask > allow)

## Shell Script Gotchas

These are real bugs encountered in production hook scripts.

### 1. `set -e` kills pipelines silently

`grep` exits 1 when it finds no match. With `set -e`, your script dies:

```bash
# BAD: script aborts if no match
set -euo pipefail
PLAN_FILE=$(grep '"plan_mode"' "$TRANSCRIPT" | jq -r '.attachment.planFilePath')

# GOOD: use set -uo pipefail (no -e), handle errors manually
set -uo pipefail
PLAN_FILE=$(grep '"plan_mode"' "$TRANSCRIPT" 2>/dev/null \
  | jq -r '.attachment.planFilePath // empty' 2>/dev/null \
  | tail -1 || true)
[ -n "$PLAN_FILE" ] || exit 0  # graceful skip
```

### 2. `|| true` zeroes `$?`

When you need the actual exit code of a command:

```bash
# BAD: $? is always 0
OUTPUT=$("$CODEX" exec ... || true)
CODEX_EXIT=$?  # always 0!

# GOOD: without set -e, $? captures the real exit code
OUTPUT=$("$CODEX" exec ...)
CODEX_EXIT=$?
```

This works because without `set -e`, a non-zero exit from command substitution does not abort the script.

### 3. stdout pollution breaks JSON output

Any stray stdout corrupts the hook's JSON response. Common offenders:

```bash
# BAD: cmux notify outputs "OK" to stdout
cmux notify --title "Review" --body "done"

# GOOD: redirect everything
cmux notify --title "Review" --body "done" >/dev/null 2>&1
```

Same applies to `echo` statements used for debugging — they go to stdout and break the JSON contract. Use a log file or stderr instead:

```bash
LOG_FILE="$HOME/.claude/plans/.hook.log"
log() { echo "[$(date '+%H:%M:%S')] $*" >> "$LOG_FILE"; }
```

### 4. jq emit helpers keep output clean

Define emit functions once and reuse them:

```bash
emit_allow() {
  jq -cn --arg r "$1" '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "allow",
      permissionDecisionReason: $r
    }
  }'
}

emit_deny() {
  jq -cn --arg r "$1" '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: $r
    }
  }'
}
```

Using `jq -cn` ensures valid JSON escaping. Never hand-craft JSON with `echo`.

## Transcript JSONL

The transcript is a JSONL file (one JSON per line). Plan mode entries look like:

```json
{
  "slug": "plan-slug-name",
  "attachment": {
    "type": "plan_mode",
    "planFilePath": "/Users/you/.claude/plans/plan-slug-name.md"
  }
}
```

Extract the plan file path:
```bash
PLAN_FILE=$(grep '"plan_mode"' "$TRANSCRIPT" 2>/dev/null \
  | jq -r '.attachment.planFilePath // empty' 2>/dev/null \
  | tail -1 || true)
```

## Plan Mode Constraints

- `ExitPlanMode` can **only** be called from within plan mode
- Claude **cannot** programmatically re-enter plan mode once it exits
- If you need Claude to iterate (edit plan, re-submit), use **PreToolUse deny** to keep plan mode active
- PostToolUse on ExitPlanMode fires too late — plan mode is already exited

## Codex CLI Integration Tips

When calling Codex from hooks:

- Use `--skip-git-repo-check` — hooks run from a different context than the target repo
- Use `--output-schema <file>` for structured JSON output (first call only)
- `codex exec resume <session_id>` does NOT support `--output-schema` — embed schema guidance in the prompt instead
- Use `-o <file>` to write output to a file (cleaner than capturing stdout which includes token counts)
- Capture session ID from stderr: `grep -oE 'session id: [a-f0-9-]+' | awk '{print $NF}'`
- JSON schema strict mode requires `"additionalProperties": false` on all object types

## Stop Hook Loop Prevention

Stop hooks can cause infinite loops (block → Claude responds → stop → block → ...). Guard against this:

```bash
INPUT=$(cat)
if [ "$(echo "$INPUT" | jq -r '.stop_hook_active')" = "true" ]; then
  exit 0  # This is already a "forced stop" after a previous block — let Claude stop
fi
```

## Permissions in settings.json

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(npm run test)",
      "Read(src/**)"
    ],
    "deny": [
      "Read(.env)",
      "Bash(curl *)"
    ]
  }
}
```

Evaluation order: **deny → ask → allow** (first match wins). Use glob patterns for matching.

## Testing Strategy

1. **Create a test project** with project-level `settings.json` — never test on global settings
2. **Use `bash -n script.sh`** to syntax-check before deploying
3. **Log everything** to a dedicated log file for debugging
4. **Test bypass mechanisms first** (`SKIP_CODEX_REVIEW=1`, frontmatter flags)
5. **Check hook input** by logging `$INPUT` to understand what fields are actually available
6. **Verify matcher** — if hooks don't fire, the matcher regex might not match the tool name
7. **Test manually**: `echo '{"tool_name":"Bash","tool_input":{"command":"test"}}' | ./hook.sh; echo "exit=$?"`
8. **Use `/hooks`** in Claude Code to see all configured hooks and their sources

## Debugging & Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Hook never fires | Matcher doesn't match tool name | Run `/hooks`, check case-sensitivity |
| "JSON validation failed" | Shell profile (`.zshrc`) outputs text unconditionally | Wrap `echo` in `if [[ $- == *i* ]]; then ... fi` |
| Hook fires but has no effect | Wrong output JSON structure | Check event-specific output format (PreToolUse vs PostToolUse) |
| "command not found" | Non-absolute path | Use full path or `$CLAUDE_PROJECT_DIR` |
| Hook timeout | Default too short for external API | Set `"timeout": 300` in hook config |

Enable debug logging: `claude --debug-file /tmp/claude.log` or `/debug` mid-session.

## Environment Variables Available in Hooks

| Variable | Description |
|----------|------------|
| `CLAUDE_SESSION_ID` | Current session identifier |
| `CLAUDE_PROJECT_DIR` | Project root directory |
| `HOME` | User home directory |
| Custom env vars | Set via `claude` launch (e.g., `SKIP_CODEX_REVIEW=1 claude`) |

## CLAUDE.md Conventions

- Project-level: `<project>/CLAUDE.md` — checked into git, shared with team
- User-level: `~/.claude/CLAUDE.md` — personal preferences
- Project user-level: `~/.claude/projects/<sanitized>/CLAUDE.md` — per-project personal notes
- Local overrides: `CLAUDE.local.md` — gitignored, per-developer
- CLAUDE.md is loaded into every conversation — keep it concise
- Use for: coding conventions, project context, behavioral preferences
- Don't use for: ephemeral state, task tracking, file listings (these go stale)
- Precedence: Managed > Local > Project > User
