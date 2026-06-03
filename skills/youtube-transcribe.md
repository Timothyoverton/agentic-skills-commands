---
name: youtube-transcript
description: Download YouTube video transcripts when user provides a YouTube URL or asks to download/get/fetch a transcript from YouTube. Also use when user wants to transcribe or get captions/subtitles from a YouTube video.
allowed-tools:
  - Bash
  - Read
  - Write
---

# YouTube Transcript Downloader

Download transcripts (subtitles/captions) from YouTube videos using yt-dlp.

## When to Use This Skill

- User provides a YouTube URL and wants the transcript
- "download transcript from YouTube", "get captions", "get subtitles", "transcribe a YouTube video"

## How It Works

1. Resolve yt-dlp — install via `uv tool` if missing
2. Try manual subtitles first (`--write-sub`) — highest quality
3. Fallback to auto-generated (`--write-auto-sub`) — usually available
4. If neither available, tell the user and stop
5. Convert VTT to plain text and clean up

## Complete Workflow

```bash
VIDEO_URL="YOUTUBE_URL"
OUTPUT_NAME="transcript_temp"

# STEP 1: Resolve yt-dlp (uv tool installs to ~/.local/bin, may not be on PATH)
if ! command -v yt-dlp &> /dev/null && ! [ -f ~/.local/bin/yt-dlp ]; then
    ~/.local/bin/uv tool install yt-dlp 2>/dev/null \
    || pip3 install yt-dlp 2>/dev/null \
    || python3 -m pip install yt-dlp
fi
YT_DLP=$(command -v yt-dlp 2>/dev/null || echo ~/.local/bin/yt-dlp)

# Get sanitised title for output filename
OUTPUT_TITLE=$("$YT_DLP" --print "%(title)s" "$VIDEO_URL" | tr '/' '_' | tr ':' '-' | tr '?"' '  ')

# STEP 2: Try manual subtitles, fallback to auto-generated
if "$YT_DLP" --write-sub --sub-langs en --skip-download --output "$OUTPUT_NAME" "$VIDEO_URL" 2>/dev/null; then
    echo "✓ Manual subtitles downloaded"
elif "$YT_DLP" --write-auto-sub --sub-langs en --skip-download --output "$OUTPUT_NAME" "$VIDEO_URL" 2>/dev/null; then
    echo "✓ Auto-generated subtitles downloaded"
else
    echo "⚠ No subtitles available for this video." && exit 1
fi

# STEP 3: Print transcript to stdout and clean up VTT
VTT_FILE=$(ls ${OUTPUT_NAME}*.vtt 2>/dev/null | head -n 1)
if [ -f "$VTT_FILE" ]; then
    python3 -c "
import re
seen = set()
with open('$VTT_FILE') as f:
    for line in f:
        line = line.strip()
        if line and not line.startswith(('WEBVTT','Kind:','Language:')) and '-->' not in line:
            clean = re.sub('<[^>]*>', '', line).replace('&amp;','&').replace('&gt;','>').replace('&lt;','<')
            if clean and clean not in seen:
                print(clean)
                seen.add(clean)
"
    rm "$VTT_FILE"
fi
```

## Error Handling

**yt-dlp not found after install** — all three install methods failed; direct user to https://github.com/yt-dlp/yt-dlp#installation

**No subtitles available** — report it clearly; don't attempt any further download

**Private / geo-blocked / age-restricted video** — report the yt-dlp error message verbatim

**Multiple languages available** — default to `--sub-langs en`; if English isn't available, run `--list-subs` and ask the user which language they want
