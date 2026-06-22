---
name: cmux
description: End-user control of cmux topology and routing (windows, workspaces, panes/surfaces, focus, moves, reorder, identify, trigger flash). Use when automation needs deterministic placement and navigation in a multi-pane cmux layout.
---

# cmux Core Control

Use this skill to control non-browser cmux topology and routing.

## Core Concepts

- Window: top-level macOS cmux window.
- Workspace: tab-like group within a window.
- Pane: split container in a workspace.
- Surface: a tab within a pane (terminal or browser panel).

"Panel" is an alias for "Surface". The `--panel` flag and the `focus-panel` /
`list-panels` / `send-panel` commands all operate on surfaces and take a
`surface:N` ref — never a `pane:N` ref.

## Fast Start

```bash
# identify current caller context
cmux identify --json

# list topology
cmux list-windows
cmux workspace list            # canonical; `list-workspaces` still works (legacy alias)
cmux list-panes
cmux list-pane-surfaces --pane pane:1

# create/focus/move
cmux workspace create          # canonical; `new-workspace` still works (legacy alias)
cmux new-split right           # split the focused pane
cmux new-split right --surface surface:7   # split from a specific surface (--panel is an alias for --surface)
cmux move-surface --surface surface:7 --pane pane:2 --focus true
cmux split-off --surface surface:7 right
cmux reorder-surface --surface surface:7 --before surface:3

# attention cue
cmux trigger-flash --surface surface:7
```

Workspace operations now have a canonical noun form: `cmux workspace <list|create|close|
rename|select|env|reconnect|disconnect|group>`. The flat verbs (`new-workspace`,
`list-workspaces`, `close-workspace`, `rename-workspace`, `select-workspace`) keep working
but print a one-time deprecation hint pointing at the noun form; set `CMUX_QUIET=1` to
silence it. Window, pane, and tab operations are still flat (`list-windows`, `list-panes`,
`focus-pane`, `new-split`, …). Note `--panel` is an alias for `--surface`, so it takes a
`surface:N` ref — passing a `pane:N` ref fails with `not_found: Surface not found`.

## External App Callers

Most `cmux` topology commands are authenticated to the current cmux caller context. When an
agent or external app was not launched from inside cmux, RPC-style commands such as
`cmux identify`, `cmux workspace list`, and `cmux workspace create` can fail with:

```text
Access denied -- only processes started inside cmux can connect
```

In that situation, do not keep retrying caller-scoped RPC commands. To open a repository or
directory in cmux from an external app on macOS, use LaunchServices instead:

```bash
open -a cmux /path/to/project
```

After running it, verify the workspace through visible cmux UI state or the saved session
file if needed. Once work continues inside a terminal launched by cmux, use normal RPC
commands again because `CMUX_WORKSPACE_ID` and related caller context variables will be set.

## Settings and Docs

Use `cmux docs settings` before changing cmux-owned settings. It prints the docs URL, schema URL, raw GitHub resources, cmux.json paths, and reload command.

```bash
cmux docs settings
cmux settings path
```

cmux-owned settings live in `~/.config/cmux/cmux.json`. Legacy `~/.config/cmux/settings.json` and `~/Library/Application Support/com.cmuxterm.app/settings.json` files are read only as fallback for missing keys. Before editing, copy any existing `cmux.json` file to a timestamped `.bak` next to it so the user can revert. Edit the user file, then reload:

```bash
cmux reload-config
```

`cmux reload-config` reloads BOTH `cmux.json` and Ghostty config (`~/.config/ghostty/config`) and refreshes terminals in place. No app restart needed.

Use cmux settings for app behavior, sidebar, notifications, browser behavior, automation, workspace colors, and cmux-owned shortcuts. Terminal rendering settings such as font, cursor style, theme, scrollback, background transparency (`background-opacity`), and blur (`background-blur`) belong in Ghostty config at `~/.config/ghostty/config`.

Open the UI when useful:

```bash
cmux settings
cmux settings cmux-json
cmux settings shortcuts
```

## Handle Model

- Default output uses short refs: `window:N`, `workspace:N`, `pane:N`, `surface:N`.
- UUIDs are still accepted as inputs.
- Request UUID output only when needed: `--id-format uuids|both`.

## Deep-Dive References

| Reference | When to Use |
|-----------|-------------|
| [references/handles-and-identify.md](references/handles-and-identify.md) | Handle syntax, self-identify, caller targeting |
| [references/windows-workspaces.md](references/windows-workspaces.md) | Window/workspace lifecycle, the `cmux workspace` noun, reorder/move |
| [references/panes-surfaces.md](references/panes-surfaces.md) | Splits, surfaces, move/reorder, focus routing, surface resume |
| [references/trigger-flash-and-health.md](references/trigger-flash-and-health.md) | Flash cue and surface health checks |

For areas outside core topology — browser automation, the markdown viewer, settings schema,
notifications/status — use the built-in docs and per-command help instead of separate skills:
`cmux docs [settings|browser|agents|dock|sidebars]` and `cmux <command> --help`.
