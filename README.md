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

---

## Installation

**If you're non-technical, just do step 1.** The plugin will auto-detect anything else that's missing and walk you through it in chat.

### 1. Install the plugin

In Claude Code, run:
```
/plugin marketplace add mathewr1993/transcript-plugin
/plugin install transcript-grabber@transcript-plugin
```

Then try it:
```
/transcript https://www.instagram.com/p/DWiDUi8jIuw/
```

**That's it.** On first run, the plugin will:
- Auto-install the Python packages it needs (`requests`, `faster-whisper`) — no action from you
- Check if FFmpeg is installed. If not, Claude will offer to run the install command for you
- Check if your Apify API token is set. If not, Claude will walk you through getting one and set the environment variable for you

For local file transcription, you don't even need the Apify token — just pass a file path.

---

### Manual setup (if you prefer to do it yourself)

#### Prerequisites
- **Python 3.9+** — https://www.python.org/downloads/
- **FFmpeg** (for video/audio decoding):
  - Windows: `winget install --id=Gyan.FFmpeg -e --source winget`
  - macOS: `brew install ffmpeg`
  - Linux (Debian/Ubuntu): `sudo apt install ffmpeg`

#### Apify API token (only needed for Instagram URLs)
1. Sign up at https://console.apify.com/sign-up (free tier includes ~$5/month credit — covers hundreds of transcripts).
2. Go to https://console.apify.com/settings/integrations and copy your **Personal API token** (starts with `apify_api_...`).
3. Set it as an env var:
   - **Windows (PowerShell):** `setx APIFY_TOKEN "apify_api_your_token_here"` — then close and reopen your terminal
   - **macOS/Linux:** add `export APIFY_TOKEN="apify_api_your_token_here"` to `~/.zshrc` or `~/.bashrc`, then `source` it

#### Python packages (auto-installed on first run, but if you want to do it yourself)
```bash
pip install requests faster-whisper
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
| `APIFY_TOKEN environment variable is not set` | Claude will offer to walk you through getting + setting a token. Or set it manually (see Manual setup above). |
| `FFmpeg is not installed or not on your PATH` | Claude will offer to run the install command for your OS. Needs admin on Windows. |
| `No video found. Post type: ...` | The Instagram link is a photo or carousel, not a video. Only videos/reels have transcripts. |
| Slow on first run | Normal — Whisper downloads its model (~150MB, one-time). Subsequent runs are fast. |
| Python packages fail to auto-install | Your Python might be in a restricted environment. Run `pip install requests faster-whisper` manually. |

---

## What it uses

- **[Apify](https://apify.com)** — scrapes the Instagram post and gets the video URL (only used for IG URLs)
- **[faster-whisper](https://github.com/SYSTRAN/faster-whisper)** — local CPU transcription (no API cost, runs on your machine)
- **[FFmpeg](https://ffmpeg.org)** — decodes video/audio formats so faster-whisper can read them
