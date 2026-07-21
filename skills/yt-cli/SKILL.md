---
name: yt-cli
description: >
  YouTube Studio CLI (`yt`) for managing YouTube channels from the terminal.
  Use when: (1) uploading, listing, updating, or deleting videos,
  (2) managing playlists, captions, comments, or thumbnails,
  (3) viewing channel analytics or revenue data,
  (4) automating YouTube operations from AI agents or scripts,
  (5) searching YouTube content, (6) managing i18n/localization,
  (7) handling authentication with YouTube API.
  Triggers: any mention of "yt", "youtube cli", "youtube management",
  video upload, channel analytics, playlist management, caption management,
  or YouTube API automation.
---

# yt - YouTube Studio CLI

Manage a YouTube channel entirely from the terminal. Designed for both human operators and AI agent automation.

## Installation

```bash
cd /path/to/youtubu_manager
uv pip install -e .
```

## Global Options

- `-o / --output {table|json|csv|yaml}` — Output format (default: `table`). Use `json` for piping to other tools or AI agents.
- `--version` — Show version.

## Authentication

Authenticate before using any command:

```bash
yt auth login                        # Browser-based OAuth
yt auth login --device               # Device flow for headless/AI environments
yt auth status                       # Check token validity
yt auth refresh                      # Force token refresh
yt auth revoke -y                    # Revoke credentials
```

## Command Reference

See [references/commands.md](references/commands.md) for the full command reference with all arguments, options, and quota costs.

## Common Workflows

### Upload and Schedule a Video

```bash
yt video upload video.mp4 \
  --title "My Video" \
  --description "Description here" \
  --tags "tag1,tag2" \
  --privacy private \
  --schedule "2026-03-01T18:00:00Z" \
  --thumbnail thumb.jpg
```

### Multilingual Video Setup

```bash
# Upload captions
yt caption upload VIDEO_ID subs_ja.srt --language ja --name "日本語"
yt caption upload VIDEO_ID subs_en.srt --language en --name "English"

# Set localized metadata
yt i18n set-video VIDEO_ID --lang en --title "English Title" --description "Desc"
```

### Channel Analytics Pipeline

```bash
# Get top videos as JSON for further processing
yt analytics top-videos --period 90d --limit 10 -o json

# Revenue overview
yt analytics revenue --period 28d -o json
```

### Comment Moderation

```bash
yt comment list VIDEO_ID --order time --limit 100 -o json
yt comment moderate COMMENT_ID --action rejected -y
```

### Playlist Management

```bash
yt playlist create "My Playlist" --privacy public
yt playlist add PLAYLIST_ID VIDEO_ID
yt playlist items PLAYLIST_ID
```

## Key Design Patterns

- **dry-run**: Most write operations support `--dry-run` to preview changes without executing.
- **Confirmation prompts**: Destructive operations (delete, revoke) require confirmation. Use `-y` to skip.
- **Quota awareness**: Each API operation has a documented quota cost. High-cost operations: `video upload` (1600), `search query` (100).
- **JSON output**: Use `-o json` for machine-readable output in automation pipelines.
