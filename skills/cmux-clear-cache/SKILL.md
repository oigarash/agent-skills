---
name: cmux-clear-cache
description: Clear cmux application cache and state files to fix rendering issues like whiteout, blank screens, or display glitches. Use when the user mentions cmux cache, cmux whiteout, cmux blank screen, cmux display issues, cmux rendering problems, or wants to reset cmux state. Also use when cmux behaves unexpectedly after updates or crashes.
---

# cmux Cache Clear

Fix cmux rendering and display issues (whiteout, blank screens, UI glitches) by clearing application cache and state files.

## Prerequisites

cmux must be fully quit before clearing cache. If it's still running, ask the user to quit it first.

## Cache Locations

cmux stores data in three macOS standard locations:

| Path | Contents | Safe to delete |
|------|----------|----------------|
| `~/Library/Caches/cmux/` | Sentry logs, async logs | Yes |
| `~/Library/Application Support/com.cmuxterm.app/` | PostHog telemetry, browser history, feature flags | Yes |
| `~/Library/Preferences/com.cmuxterm.app.plist` | User preferences and settings | Only if full reset needed |

## Procedure

### Standard cache clear (preserves settings)

```bash
rm -rf ~/Library/Caches/cmux/
rm -rf ~/Library/Application\ Support/com.cmuxterm.app/
```

### Full reset (including settings)

Only do this if the user explicitly asks for a full reset:

```bash
rm -rf ~/Library/Caches/cmux/
rm -rf ~/Library/Application\ Support/com.cmuxterm.app/
rm ~/Library/Preferences/com.cmuxterm.app.plist
```

### Socket cleanup

If cmux won't start or connect after cache clear, also remove the socket file:

```bash
rm -f /tmp/cmux.sock
```

## After clearing

Tell the user to relaunch cmux and verify the issue is resolved.

## Known issues this fixes

- Terminal whiteout when receiving notifications without switching windows
- Blank or frozen panes after sleep/wake
- UI rendering corruption after cmux updates
