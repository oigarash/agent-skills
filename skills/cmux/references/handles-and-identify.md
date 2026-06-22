# Handles and Identify

Use `identify` and short handles for deterministic automation targeting.

## Handle Inputs

Most v2-backed commands accept:
- UUID
- short ref (`window:N`, `workspace:N`, `pane:N`, `surface:N`; `tab:N` for `tab-action`/`rename-tab`)
- index (where legacy/index-based commands still allow it)

`cmux identify --json` returns both `caller` and `focused` blocks, each carrying
`window_ref` / `workspace_ref` / `pane_ref` / `surface_ref` / `tab_ref`. The caller defaults
come from `$CMUX_WORKSPACE_ID`, `$CMUX_SURFACE_ID`, and `$CMUX_TAB_ID` in cmux terminals.

## Self Identify

```bash
cmux identify --json
```

Returns current focused topology plus optional caller resolution.

## Caller Override

```bash
cmux identify --workspace workspace:2
cmux identify --workspace workspace:2 --surface surface:8
```

Useful for agents that need to route relative actions from a known caller anchor.

## Output Shaping

```bash
cmux --json identify                 # refs-first output
cmux --json --id-format both identify
cmux --json --id-format uuids identify
```
