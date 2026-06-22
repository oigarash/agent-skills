# Windows and Workspaces

Window/workspace lifecycle and ordering operations.

Workspace operations have a canonical noun form: `cmux workspace <subcommand>`. The flat verbs
(`new-workspace`, `list-workspaces`, `close-workspace`, `rename-workspace`, `select-workspace`)
still work but print a one-time deprecation hint; set `CMUX_QUIET=1` to silence it. Window
operations remain flat (no `cmux window` noun yet).

## Inspect

```bash
cmux list-windows
cmux current-window
cmux workspace list            # legacy alias: cmux list-workspaces
cmux current-workspace
```

## Create/Focus/Close

```bash
cmux new-window
cmux focus-window --window window:2
cmux close-window --window window:2

cmux workspace create          # legacy alias: cmux new-workspace
cmux workspace select workspace:4
cmux workspace rename workspace:4 --title "build"
cmux workspace close workspace:4
```

`workspace create` accepts the same flags as `new-workspace`, plus `--env KEY=VALUE` and
`--env-file <path>`. Inspect a workspace's configured environment with `cmux workspace env
[workspace] [--mask]`.

## Reorder and Move

`reorder-workspace` and `move-workspace-to-window` are not part of the `workspace` noun yet —
they stay flat.

```bash
cmux reorder-workspace --workspace workspace:4 --before workspace:2
cmux move-workspace-to-window --workspace workspace:4 --window window:1
```

## Remote (SSH) Workspaces and Groups

```bash
cmux workspace reconnect workspace:4     # reconnect a remote (SSH) workspace
cmux workspace disconnect workspace:4    # stop a remote workspace's connection

cmux workspace-group list                # collapsible sidebar groups (alias: cmux workspace group)
cmux workspace-group create --name api --from workspace:2,workspace:4
cmux workspace-group ungroup <group>     # dissolve, preserving member workspaces
```
