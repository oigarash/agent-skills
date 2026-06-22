# Panes and Surfaces

Split layout, surface creation, focus, move, and reorder.

## Inspect

```bash
cmux list-panes
cmux list-pane-surfaces --pane pane:1
```

## Create Splits/Surfaces

```bash
cmux new-split right                                 # split the focused pane
cmux new-split right --surface surface:7             # split from a specific surface
cmux new-surface --type terminal --pane pane:1
cmux new-surface --type browser --pane pane:1 --url https://example.com
cmux new-surface --type agent-session --provider claude --pane pane:1 --focus true
```

`new-split`'s `--panel` is an alias for `--surface`; it expects a `surface:N` ref. Passing a
`pane:N` ref fails with `not_found: Surface not found`. Omit it to split the focused pane.
`new-surface --type agent-session` also accepts `--provider <codex|claude|opencode>` and
`--renderer <react|solid>`.

## Focus and Close

```bash
cmux focus-pane --pane pane:2
cmux focus-panel --panel surface:7                   # "panel" == surface; takes a surface:N ref
cmux close-surface --surface surface:7
```

## Move/Reorder Surfaces

```bash
cmux move-surface --surface surface:7 --pane pane:2 --focus true
cmux move-surface --surface surface:7 --workspace workspace:2 --window window:1 --after surface:4
cmux split-off --surface surface:7 right
cmux reorder-surface --surface surface:7 --before surface:3
```

Surface identity is stable across move/reorder/split-off operations. Layout commands are focus-neutral by default; pass `--focus true` only when you want the moved or created surface selected.

## Surface Resume

Attach restart/attach metadata to a terminal surface so it can be restored later:

```bash
cmux surface resume set --kind tmux --shell "tmux attach -t work"
cmux surface resume set --kind opencode --checkpoint ses_123 -- opencode --session ses_123
cmux surface resume show --json
cmux surface resume get
cmux surface resume clear
```
