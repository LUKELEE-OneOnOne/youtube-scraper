# youtube-scraper

An OpenClaw skill that scrapes YouTube video metadata and downloads subtitles — no YouTube API key required.

---

## What It Does

Given a YouTube URL, this skill:

1. **Verifies you are logged into YouTube** via the OpenClaw browser before doing anything
2. **Scrapes full video metadata** — title, video ID, thumbnail, publish date, view count, like count, channel name, subscriber count, channel country/region
3. **Downloads subtitles** using a three-level fallback chain:
   - Creator-uploaded subtitles (highest quality)
   - YouTube auto-generated subtitles
   - Local Whisper transcription from downloaded audio (when subtitles are fully disabled)

Everything runs locally. No YouTube Data API key, no OpenAI API key.

---

## Requirements

| Tool | Install | Role |
|------|---------|------|
| `yt-dlp` | `brew install yt-dlp` | Metadata scraping, subtitle download, audio download |
| `youtube-transcript-api` | `pip install youtube-transcript-api` | Fast subtitle fetching (no video download needed) |
| `whisper` | `pip install openai-whisper` | Last-resort transcription when subtitles are disabled |
| `ffmpeg` | `brew install ffmpeg` | Required by Whisper for audio processing |
| Python 3 | pre-installed on macOS | Runs conversion scripts |

`whisper` and `ffmpeg` are optional — only needed if you expect to encounter videos with subtitles fully turned off.

---

## Browser Mode

This skill supports two browser configurations. It detects which one is available automatically.

### Mode A — OpenClaw Managed Chromium (preferred)

A dedicated, isolated Chromium instance managed by OpenClaw. The agent controls it directly.

- Log into YouTube inside this browser once
- The skill reads cookies from it via `openclaw browser cookies --json` and converts them to a format yt-dlp understands

### Mode B — Chrome Extension Relay (fallback)

Controls your existing Chrome window via the OpenClaw Browser Relay extension.

- Log into YouTube in your normal Chrome
- Click the **OpenClaw Browser Relay** extension icon on the YouTube tab (badge shows **ON**)
- yt-dlp reads your Chrome cookie database directly — no export step needed

---

## Subtitle Fallback Chain

```
youtube-transcript-api
  ├─ creator-uploaded subtitles   ──→ SUCCESS: save .srt
  └─ auto-generated subtitles     ──→ SUCCESS: save .srt
         │
         └─ (if ERROR) yt-dlp --write-subs
                ├─ creator-uploaded subtitles  ──→ SUCCESS: save .srt/.vtt
                └─ auto-generated subtitles    ──→ SUCCESS: save .srt/.vtt
                         │
                         └─ (if DISABLED) Whisper
                                  ├─ download audio via yt-dlp
                                  └─ transcribe locally → save .srt
```

---

## Metadata Fields Collected

| Field | Source |
|-------|--------|
| Video ID | `yt-dlp` (`id`) |
| Title | `yt-dlp` (`title`) |
| Publish date | `yt-dlp` (`upload_date`) |
| Duration | `yt-dlp` (`duration_string`) |
| View count | `yt-dlp` (`view_count`) |
| Like count | `yt-dlp` (`like_count`) |
| Thumbnail | `yt-dlp` (`thumbnail`) — downloaded as `.jpg` |
| Channel name | `yt-dlp` (`channel`) |
| Subscriber count | `yt-dlp` (`channel_follower_count`) |
| Channel country/region | `yt-dlp` (`uploader_country`) — browser scrape of About page as fallback |
| Channel URL | `yt-dlp` (`channel_url`) |

---

## Output

All files are saved to `/tmp/youtube-<VIDEO_ID>/`:

```
/tmp/youtube-dQw4w9WgXcQ/
├── Video Title.jpg          # thumbnail
└── subtitle_dQw4w9WgXcQ.srt # subtitles (source noted in summary)
```

The skill also returns a structured text summary:

```
📹  Video Info
────────────────────────────────────────
Title          : Never Gonna Give You Up
Video ID       : dQw4w9WgXcQ
Published      : 2009-10-25
Duration       : 3:33

📊  Stats
────────────────────────────────────────
Views          : 1,600,000,000
Likes          : 16,000,000

📺  Channel
────────────────────────────────────────
Channel name   : Rick Astley
Subscribers    : 4,200,000
Country/Region : GB
Channel URL    : https://www.youtube.com/@RickAstleyYT

📁  Local Files
────────────────────────────────────────
Thumbnail      : /tmp/youtube-dQw4w9WgXcQ/Never Gonna Give You Up.jpg
Subtitle file  : /tmp/youtube-dQw4w9WgXcQ/subtitle_dQw4w9WgXcQ.srt
Subtitle source: creator-uploaded (en)
```

---

## Installation

**Option 1 — Place in your workspace skills folder** (single agent):

```bash
# Already in place if you cloned openclaw-PlayGround
~/Downloads/openclaw-PlayGround/skills/youtube-scraper/SKILL.md
```

Restart the OpenClaw Gateway or tell the agent to "refresh skills" to pick it up.

**Option 2 — Install from the `.skill` package** (global, all agents):

```bash
clawhub install ~/Downloads/youtube-scraper.skill
```

**Option 3 — Place in the shared local skills folder**:

```bash
cp -r ~/Downloads/openclaw-PlayGround/skills/youtube-scraper ~/.openclaw/skills/
```

---

## Triggering the Skill

Once installed, give the agent a YouTube URL with any natural language request:

```
Scrape this video: https://www.youtube.com/watch?v=dQw4w9WgXcQ
```

```
Download the subtitles and get the metadata for https://youtu.be/dQw4w9WgXcQ
```

```
Get the transcript and channel info for this video: https://www.youtube.com/shorts/abc123
```

---

## Whisper Model Size Guide

Only relevant when Method C (Whisper) is triggered.

| Model | Size | Speed | Accuracy | Best for |
|-------|------|-------|----------|---------|
| `tiny` | 75 MB | fastest | lowest | quick drafts |
| `base` | 142 MB | fast | low | short clips |
| `small` | 244 MB | moderate | good | **default** |
| `medium` | 769 MB | slow | high | long videos |
| `large` | 1.5 GB | slowest | best | high-accuracy needs |

Models are downloaded automatically on first use and cached at `~/.cache/whisper/`.

---

## Troubleshooting

**yt-dlp reports not logged in (Mode A)**
Revisit YouTube in the OpenClaw managed browser to refresh the session, then re-run the cookie export step.

**`--cookies-from-browser chrome` fails (Mode B)**
Grant Chrome "Full Disk Access" in System Settings → Privacy & Security, or use the extension relay cookie export as a fallback.

**Extension badge shows `!`**
The OpenClaw Gateway is not running on this machine. Start the Gateway; the extension reconnects automatically.

**youtube-transcript-api blocked**
This only happens when running behind a datacenter proxy. Disable the proxy and retry — local direct connections are unaffected.

**Members-only video**
The logged-in account must hold the required membership. The exported cookies carry the membership credentials to yt-dlp.

---

## Related Projects

| Project | Platform | Focus |
|---------|----------|-------|
| [jerrykuku/youtube-video-analyzer](https://github.com/jerrykuku/youtube-video-analyzer) | Claude Code skill | Transcripts + comments + sentiment analysis, zero dependencies |
| [jdepoix/youtube-transcript-api](https://github.com/jdepoix/youtube-transcript-api) | Python library | Subtitle fetching via YouTube internal API |
| [yt-dlp/yt-dlp](https://github.com/yt-dlp/yt-dlp) | CLI tool | Video/audio/metadata download |
| [openai/whisper](https://github.com/openai/whisper) | Python library | Local speech-to-text transcription |

This skill focuses on **metadata scraping**, **login-aware subtitle download**, and **Whisper fallback** for the OpenClaw platform. For comment analysis and sentiment reporting, see [jerrykuku/youtube-video-analyzer](https://github.com/jerrykuku/youtube-video-analyzer).
