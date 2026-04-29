---
name: bilibili-content
description: "Extract Bilibili video subtitles (AI + uploader + ASR fallback), danmaku analysis, and top comments. Output as summaries, chapter notes, threads, or blog posts."
version: 1.0.0
author: Hermes Agent + community
license: MIT
metadata:
  hermes:
    tags: [bilibili, video, subtitle, transcript, danmaku, chinese, asr, whisper]
    homepage: https://github.com/NousResearch/hermes-agent
---

# Bilibili Content Tool

Extract and transform Bilibili video content — the B站 equivalent of `youtube-content`. Grabs subtitles via B站's AI ASR, uploader-provided captions, or local Whisper fallback. Also captures danmaku (弹幕) highlights and top comments.

## Features

- **Three-tier subtitle extraction** — B站 AI auto-generated (`ai-zh`), uploader captions (`zh`), or local `faster-whisper` ASR when neither is available
- **Danmaku analysis** — top keywords, peak-density time segments with sample texts
- **Top comments** — sorted by likes, useful for corrections and supplementary insights
- **Multi-page support** — handles 分P videos via `--page` or auto-detects `p=` in URL
- **Multiple output formats** — chapter summaries, full transcripts, X/Twitter threads, blog posts, and tutorial-style study notes

## When to use

User shares a Bilibili link, wants a video summarized, needs the transcript, or asks for study notes from a tutorial video. Works with `bilibili.com/video/`, `b23.tv` short links, BV号, and AV号.

## Setup

```bash
# Core dependencies
pip install bilibili-api-python httpx

# Optional: local ASR fallback (when no subtitles available)
pip install faster-whisper
# macOS: brew install ffmpeg
```

### Bilibili AI Subtitles (Cookie Required)

B站's AI auto-generated subtitles have ~90% coverage but require a login session:

```bash
# 1. Log into B站 in your browser
# 2. F12 → Application → Cookies → bilibili.com → copy SESSDATA value
# 3. Set the environment variable
export BILIBILI_SESSDATA="your_sessdata_value"
```

Without a cookie, only uploader-submitted captions are available (rare — <5% of videos).

### Local ASR Fallback

When both AI and uploader subtitles are unavailable:

```bash
python3 SKILL_DIR/scripts/fetch_content.py "URL" --asr-fallback
python3 SKILL_DIR/scripts/fetch_content.py "URL" --asr-fallback --asr-model small  # better accuracy
```

Model sizes: `tiny` (fastest) < `base` (default) < `small` < `medium` < `large` (best).

## Usage

`SKILL_DIR` is the directory containing this SKILL.md file.

```bash
# Full extraction (subtitle + danmaku + comments as JSON)
python3 SKILL_DIR/scripts/fetch_content.py "https://www.bilibili.com/video/BV1xx411c7m2"

# Plain subtitle text (for piping into further processing)
python3 SKILL_DIR/scripts/fetch_content.py "BV1xx411c7m2" --subtitle-only

# Subtitle with timestamps
python3 SKILL_DIR/scripts/fetch_content.py "URL" --subtitle-only --timestamps

# Skip danmaku and comments (fastest)
python3 SKILL_DIR/scripts/fetch_content.py "URL" --no-danmaku --no-comments

# Specify page for multi-P videos
python3 SKILL_DIR/scripts/fetch_content.py "URL" --page 17  # 0-based, P18

# Language preference
python3 SKILL_DIR/scripts/fetch_content.py "URL" --language zh-CN
```

URLs with `?p=18` are auto-detected — no need for `--page`.

### JSON Output Structure

```json
{
  "video_id": "BV1xx411c7m2",
  "title": "视频标题",
  "description": "视频简介",
  "duration": "12:34",
  "duration_seconds": 754,
  "pages": 1,
  "current_page": 1,
  "url": "https://www.bilibili.com/video/BV1xx411c7m2",
  "subtitle": {
    "language": "ai-zh",
    "language_display": "中文",
    "source": "ai",
    "segment_count": 519,
    "segments": [
      {"text": "好的各位同学", "start": 0.82, "duration": 0.86},
      ...
    ],
    "full_text": "完整字幕文本...",
    "timestamped_text": "0:00 好的各位同学\n0:01 ..."
  },
  "danmaku": {
    "count": 380,
    "top_keywords": [{"text": "太细了", "count": 6}, ...],
    "hot_segments": [
      {"time_range": "5:30-6:00", "count": 22, "samples": ["太细了", ...]},
      ...
    ]
  },
  "comments": {
    "total_ac": 1234,
    "top_comments": [
      {"user": "用户名", "content": "...", "likes": 233, "timestamp": 1700000000},
      ...
    ]
  }
}
```

## Output Formats

After fetching, transform based on user request. For **tutorial/course videos**, use structured study notes with code blocks, tables, tips, and pitfalls. For other videos, select from:

- **Study Notes** (教程类) — layered knowledge points, code blocks, ⚠️ pitfalls, 💡 tips
- **Chapters** — timed chapter list grouped by topic shifts
- **Summary** — 5–10 sentence overview
- **Blog Post** — full article with title, sections, and key takeaways
- **Thread** — X/Twitter-style numbered posts, each ≤280 chars
- **Danmaku Report** — keyword frequency, peak moments, audience sentiment
- **Comment Highlights** — top-liked comments with supplementary context

Default: **Study Notes** for tutorial/course content, **Summary** otherwise.

## Workflow

1. **Fetch** — run `fetch_content.py` with the URL
2. **Validate** — confirm output is non-empty and in the expected language. If subtitle is empty but danmaku/comments exist, note this and use what's available.
3. **Chunk if needed** — if subtitle exceeds ~50K characters, split into overlapping chunks (~40K with 2K overlap), summarize each, then merge.
4. **Transform** — format into the requested output type. Match the video's category (tutorial → study notes, documentary → blog post, etc.).
5. **Verify** — re-read the output to check coherence, correct timestamps, and completeness.

## Subtitle Strategy (Priority Order)

```
1. get_player_info(cid) → AI auto-generated (ai-zh)   coverage: ~90%  needs: Cookie
2. get_subtitle(cid)    → Uploader-provided (zh)       coverage: <5%   needs: none
3. fetch_subtitle_asr() → Local Whisper transcription  coverage: 100%  needs: ffmpeg + faster-whisper
```

Strategy 3 only activates with `--asr-fallback` flag.

## Error Handling

- **No subtitles**: tell the user; note that danmaku and comments are still available
- **Private/deleted video**: relay the API error message
- **Missing dependency**: run `pip install bilibili-api-python httpx`
- **No cookie / AI subtitles unavailable**: suggest setting `BILIBILI_SESSDATA` or using `--asr-fallback`
- **Large danmaku volume**: script samples first 20 segments (120 minutes) — sufficient for analysis

## Files

```
bilibili-content/
├── SKILL.md                        # This file
├── scripts/
│   └── fetch_content.py            # Core extraction script
└── references/
    └── output-formats.md           # Detailed format examples
```
