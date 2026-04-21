# Transcript Grabber — Claude Code Plugin

A `/transcript` slash command for Claude Code. Works on two kinds of input:
- **Instagram video URLs** — scrapes the post and transcribes the video
- **Local video/audio files** — transcribes a file already on your machine (mp4, mov, mkv, webm, mp3, wav, m4a, anything ffmpeg can decode)

```
# Instagram URL
/transcript https://www.instagram.com/p/DWiDUi8jIuw/

# Local file (Windows)
/transcript C:/Users/you/Downloads/interview.mp4

# Local file (macOS/Linux)
/transcript /Users/you/recordings/podcast.mp3
```

Returns:
- Source + duration (creator handle for IG, filename for local files)
- Timestamped transcript (each segment with `[start - end]`)
- Clean flowing transcript (no timestamps)

**Note:** The Apify API token is only needed for Instagram URLs. Local file transcription works with zero tokens — just Python + faster-whisper.

---

## Installation (one-time, ~3 minutes)

### 1. Install the plugin
In Claude Code, run:
```
/plugin marketplace add mathewr1993/transcript-plugin
/plugin install transcript-grabber@transcript-plugin
```

### 2. Get your own free Apify API token
This skill uses [Apify](https://apify.com) to scrape Instagram. Each person needs their own key (free tier is plenty).

1. Sign up at https://console.apify.com/sign-up — free tier includes ~$5/month credit, which covers hundreds of transcripts.
2. Go to **Settings → API & Integrations**: https://console.apify.com/settings/integrations
3. Copy your **Personal API token** (starts with `apify_api_...`).

### 3. Set the token as an environment variable

**Windows (PowerShell):**
```powershell
setx APIFY_TOKEN "apify_api_your_token_here"
```
Then **close and reopen your terminal** — `setx` only applies to new terminals.

**macOS / Linux:**
Add this line to `~/.zshrc` (Mac default) or `~/.bashrc` (Linux):
```bash
export APIFY_TOKEN="apify_api_your_token_here"
```
Then run `source ~/.zshrc` (or `source ~/.bashrc`), or open a new terminal.

**Verify it worked:**
```bash
# Windows:
echo %APIFY_TOKEN%
# macOS/Linux:
echo $APIFY_TOKEN
```
Should print your token.

### 4. Install the Python dependencies
```bash
pip install requests faster-whisper
```
(First `/transcript` run will also download the Whisper "base" model — about 150MB, one-time.)

### 5. Test it
In Claude Code:
```
/transcript https://www.instagram.com/p/DWiDUi8jIuw/
```

---

## Updates

When a new version is released, run:
```
/plugin marketplace update transcript-plugin
```
Or turn on auto-update in `/plugin` → Marketplaces tab.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `APIFY_TOKEN environment variable is not set` | Your terminal hasn't picked up the new variable. Close and reopen your terminal, or run `echo $APIFY_TOKEN` / `echo %APIFY_TOKEN%` to verify it's set. |
| `No video found. Post type: ...` | The link is a photo or carousel post, not a video. Only videos/reels have transcripts. |
| Transcription is slow on first run | Normal — Whisper is downloading its model (~150MB, one-time). Subsequent runs are fast. |
| `ModuleNotFoundError: No module named 'faster_whisper'` | Run `pip install faster-whisper` (or `pip3 install faster-whisper`). |

---

## What it uses

- **[Apify](https://apify.com)** — scrapes the Instagram post and gets the video URL
- **[faster-whisper](https://github.com/SYSTRAN/faster-whisper)** — local CPU transcription (no API cost, runs on your machine)
