# yt CLI — Full Command Reference

## auth — Manage YouTube API authentication

| Subcommand | Arguments | Options | Quota |
|---|---|---|---|
| `login` | — | `--client-secrets PATH`, `--device` | — |
| `status` | — | — | — |
| `refresh` | — | — | — |
| `revoke` | — | `--yes/-y` | — |

## video — Manage videos

| Subcommand | Arguments | Options | Quota |
|---|---|---|---|
| `upload` | `FILE` | `--title` (req), `--description`, `--tags`, `--category` (def:22), `--privacy` (def:private), `--schedule` (ISO 8601), `--language`, `--thumbnail`, `--dry-run` | 1600 |
| `list` | — | `--privacy`, `--limit` (def:25) | 100+ |
| `get` | `VIDEO_ID` | — | 1 |
| `update` | `VIDEO_ID` | `--title`, `--description`, `--tags`, `--category`, `--privacy`, `--dry-run` | 50 |
| `delete` | `VIDEO_ID` | `--yes/-y`, `--dry-run` | 50 |

## channel — Manage channel settings

| Subcommand | Arguments | Options | Quota |
|---|---|---|---|
| `info` | — | — | — |
| `update` | — | `--description`, `--country`, `--keywords`, `--dry-run` | 50 |
| `branding` | `IMAGE` | — | 50 |
| `watermark-set` | `IMAGE` | `--position` (corner\|bottom) | 50 |
| `watermark-remove` | — | — | 50 |

## playlist — Manage playlists

| Subcommand | Arguments | Options | Quota |
|---|---|---|---|
| `list` | — | `--limit` (def:25) | — |
| `create` | `TITLE` | `--description`, `--privacy` (def:private), `--dry-run` | 50 |
| `update` | `PLAYLIST_ID` | `--title`, `--description`, `--privacy` | 50 |
| `delete` | `PLAYLIST_ID` | `--yes/-y` | 50 |
| `add` | `PLAYLIST_ID VIDEO_ID` | — | 50 |
| `remove` | `PLAYLIST_ID ITEM_ID` | `--yes/-y` | 50 |
| `items` | `PLAYLIST_ID` | `--limit` (def:25) | — |

## comment — Manage video comments

| Subcommand | Arguments | Options | Quota |
|---|---|---|---|
| `list` | `VIDEO_ID` | `--limit` (def:25), `--order` (time\|relevance) | — |
| `post` | `VIDEO_ID` | `--text/-t` (req), `--dry-run` | 50 |
| `reply` | `COMMENT_ID` | `--text/-t` (req) | 50 |
| `moderate` | `COMMENT_ID` | `--action` (held\|published\|rejected, req), `--yes/-y` | 50 |
| `delete` | `COMMENT_ID` | `--yes/-y` | 50 |

## caption — Manage video captions

| Subcommand | Arguments | Options | Quota |
|---|---|---|---|
| `list` | `VIDEO_ID` | — | 50 |
| `upload` | `VIDEO_ID FILE` | `--language` (req), `--name`, `--format` (srt\|vtt\|sbv), `--dry-run` | 400 |
| `download` | `CAPTION_ID` | `--format` (srt\|vtt\|sbv), `--output-file/-f` | — |
| `update` | `CAPTION_ID FILE` | — | 450 |
| `delete` | `CAPTION_ID` | `--yes/-y` | 50 |

## thumbnail — Manage video thumbnails

| Subcommand | Arguments | Options | Quota |
|---|---|---|---|
| `set` | `VIDEO_ID IMAGE` | — | 50 |
| `get` | `VIDEO_ID` | — | 1 |

## analytics — View channel analytics

All analytics commands support `--period` (7d\|28d\|90d\|365d, default: 28d).

| Subcommand | Arguments | Extra Options |
|---|---|---|
| `overview` | — | — |
| `top-videos` | — | `--limit` (def:25) |
| `traffic` | — | — |
| `demographics` | — | — |
| `geography` | — | — |
| `revenue` | — | — |
| `video` | `VIDEO_ID` | — |

## search — Search YouTube content

| Subcommand | Arguments | Options | Quota |
|---|---|---|---|
| `query` | `QUERY` | `--type` (video\|channel\|playlist), `--order` (date\|rating\|relevance\|viewCount), `--limit` (def:25), `--dry-run` | 100 |

## i18n / localize — Manage internationalization

| Subcommand | Arguments | Options | Quota |
|---|---|---|---|
| `languages` | — | — | — |
| `regions` | — | — | — |
| `set-video` | `VIDEO_ID` | `--lang` (req), `--title`, `--description` | 50 |

## reporting — Manage YouTube reporting jobs

| Subcommand | Arguments | Options | Quota |
|---|---|---|---|
| `types` | — | — | — |
| `create` | `REPORT_TYPE` | — | — |
| `list` | — | — | — |
| `download` | `REPORT_URL` | `--output-file/-f` (req) | — |
