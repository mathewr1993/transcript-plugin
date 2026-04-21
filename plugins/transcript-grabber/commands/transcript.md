# Video Transcript Grabber

Grab the transcript from an Instagram video link OR a local video/audio file on your computer. One command, clean output.

## Arguments
- `$ARGUMENTS` — Either an Instagram post URL **or** a local file path (video or audio).
  - IG URL example: `/transcript https://www.instagram.com/p/DWfipnmjYjo/`
  - IG reel example: `/transcript https://www.instagram.com/reel/ABC123/`
  - Local file example: `/transcript C:/Users/you/Downloads/interview.mp4`
  - Local audio example: `/transcript /Users/you/recordings/podcast.mp3`
  - Supported local formats: any format ffmpeg can decode — mp4, mov, mkv, webm, mp3, wav, m4a, etc.

## Instructions

You are a fast transcript grabber. One script, one run, zero delays.

### INPUT DETECTION

The argument is either:
- **A URL** (starts with `http://` or `https://`) → use the Instagram scrape flow. Requires `APIFY_TOKEN`.
- **A local file path** → skip scraping, transcribe the file directly. Does NOT require `APIFY_TOKEN`.

The script below auto-detects which mode to use based on whether the argument starts with `http`.

### ERROR HANDLING — BE PROACTIVE

Your job is to make this skill feel magical for non-technical users. The Python script auto-installs its pip dependencies. But two things can still fail on a fresh machine — and when they do, **don't just print the error and stop**. Detect the specific failure mode below and handle it for the user.

**Case 1: `APIFY_TOKEN` missing (URL mode only)**
The script exits with `ERROR: APIFY_TOKEN environment variable is not set.`

Respond with:
> "You don't have an Apify API token set up yet — we need one to scrape the Instagram post. It takes 2 minutes:
>
> **1.** Go to https://console.apify.com/sign-up and create a free account (free tier includes ~$5/month credit — plenty for transcripts).
> **2.** Once signed in, go to https://console.apify.com/settings/integrations and copy your **Personal API token** (starts with `apify_api_...`).
> **3.** Paste the token in this chat and I'll set the environment variable for you."

When they paste the token, run the appropriate command yourself via Bash:
- **Windows:** `setx APIFY_TOKEN "their_token_here"`
- **macOS/Linux:** append `export APIFY_TOKEN="their_token_here"` to `~/.zshrc` (Mac) or `~/.bashrc` (Linux)

Then tell them: **"Done. Close and reopen your terminal (Windows) or run `source ~/.zshrc` (Mac), then re-run `/transcript <url>`."**

**Case 2: FFmpeg missing**
The script exits with `ERROR: FFmpeg is not installed...` and prints the correct install command for their OS.

Offer to run that command for them:
> "FFmpeg is missing — that's the video/audio decoder faster-whisper uses. I can install it for you. Want me to run: `<exact command from script output>`?"

If yes, run it via Bash. On Windows this needs admin rights — if winget fails, fall back to guiding them to download the installer from https://ffmpeg.org/download.html. After install succeeds, tell them to close and reopen their terminal, then re-run the skill.

**Do not silently retry the script in a loop.** One clean failure → one clear fix → one re-run.

### CRITICAL RULES

1. **One script does everything.** Auto-detect URL vs local file, download if needed, transcribe — all in a single Python execution.
2. **For URLs, use Apify** with `apify~instagram-scraper` actor + `directUrls`. For local files, skip Apify entirely.
3. **Use faster-whisper for transcription** (base model, cpu, int8). Works on both video and audio files.
4. **Save results to `~/temp_post.json`** (user's home dir) to avoid Windows cp1252 Unicode errors.
5. **Output both timestamped and clean transcript.** Timestamped first, then clean flowing text.
6. **No commentary, no analysis.** Just deliver the transcript unless the user asks for more.

### Step 1: Run the All-in-One Script

Run this single Python script via Bash (with a 5-minute timeout):

```python
import os, sys, subprocess, shutil, platform, json, time
from pathlib import Path

# --- PREFLIGHT 1: Auto-install missing Python packages ---
def ensure(pkg, import_name=None):
    try:
        __import__(import_name or pkg.replace("-", "_"))
    except ImportError:
        print(f"Installing {pkg} (one-time setup)...")
        subprocess.check_call([sys.executable, "-m", "pip", "install", "-q", pkg])

ensure("requests")
ensure("faster-whisper", "faster_whisper")

import requests  # safe to import now

# --- PREFLIGHT 2: Check FFmpeg ---
if not shutil.which("ffmpeg"):
    system = platform.system()
    print("ERROR: FFmpeg is not installed or not on your PATH.")
    print("faster-whisper needs FFmpeg to decode video/audio files.")
    print()
    if system == "Windows":
        print("Install command (Windows, needs admin): winget install --id=Gyan.FFmpeg -e --source winget")
        print("Or download manually from: https://ffmpeg.org/download.html")
    elif system == "Darwin":
        print("Install command (macOS):   brew install ffmpeg")
        print("(If you don't have Homebrew: https://brew.sh)")
    else:
        print("Install command (Debian/Ubuntu):   sudo apt update && sudo apt install -y ffmpeg")
        print("(For other Linux distros, use your package manager's ffmpeg package)")
    print()
    print("After installing, close and reopen your terminal, then re-run /transcript.")
    exit(1)

# --- Config ---
INPUT = "$ARGUMENTS"  # Replace with the actual URL or file path from arguments
HOME = Path.home()
VIDEO_PATH = str(HOME / "temp_video.mp4")
POST_PATH = str(HOME / "temp_post.json")

# --- Detect input type ---
is_url = INPUT.startswith(("http://", "https://"))

if is_url:
    # --- URL mode: scrape Instagram + download video ---
    TOKEN = os.environ.get("APIFY_TOKEN")
    if not TOKEN:
        print("ERROR: APIFY_TOKEN environment variable is not set.")
        print("Get a token at https://console.apify.com/settings/integrations")
        print("Then set it — Windows (PowerShell): setx APIFY_TOKEN \"your_token_here\"")
        print("              macOS/Linux: add 'export APIFY_TOKEN=your_token_here' to ~/.zshrc or ~/.bashrc")
        print("Restart your terminal after setting it.")
        exit(1)

    print("Scraping post...")
    resp = requests.post(
        f'https://api.apify.com/v2/acts/apify~instagram-scraper/runs?token={TOKEN}',
        json={'directUrls': [INPUT], 'resultsType': 'posts', 'resultsLimit': 1}
    )
    run_id = resp.json()['data']['id']

    for i in range(60):
        time.sleep(2)
        sr = requests.get(f'https://api.apify.com/v2/actor-runs/{run_id}?token={TOKEN}')
        status = sr.json()['data']['status']
        if status == 'SUCCEEDED':
            break
        if status in ('FAILED', 'ABORTED', 'TIMED-OUT'):
            print(f"Apify run failed: {status}")
            exit(1)

    dataset_id = sr.json()['data']['defaultDatasetId']
    results = requests.get(f'https://api.apify.com/v2/datasets/{dataset_id}/items?token={TOKEN}').json()

    with open(POST_PATH, 'w', encoding='utf-8') as f:
        json.dump(results, f, ensure_ascii=False, indent=2)

    post = results[0]
    video_url = post.get('videoUrl')
    owner = post.get('ownerUsername', 'unknown')
    caption_preview = (post.get('caption') or '')[:100]

    if not video_url:
        print(f"No video found. Post type: {post.get('type')}")
        print(f"Caption: {caption_preview}")
        exit(1)

    print(f"@{owner} | {post.get('type')} | Downloading...")
    r = requests.get(video_url, stream=True)
    with open(VIDEO_PATH, 'wb') as f:
        for chunk in r.iter_content(8192):
            f.write(chunk)

    transcribe_path = VIDEO_PATH
    label = f"@{owner}"

else:
    # --- Local file mode: skip scraping, transcribe directly ---
    transcribe_path = INPUT
    if not os.path.isfile(transcribe_path):
        print(f"ERROR: File not found: {transcribe_path}")
        print("Pass either an Instagram URL (https://...) or a valid local file path.")
        exit(1)
    label = os.path.basename(transcribe_path)
    print(f"Local file: {label} | Transcribing...")

# --- Transcribe ---
from faster_whisper import WhisperModel
model = WhisperModel("base", device="cpu", compute_type="int8")
segments, info = model.transcribe(transcribe_path, language="en")

print(f"\n--- {label} | {info.duration:.0f}s ---\n")

full = []
for seg in segments:
    print(f"[{seg.start:.1f}s - {seg.end:.1f}s] {seg.text.strip()}")
    full.append(seg.text.strip())

print(f"\n--- CLEAN TRANSCRIPT ---\n")
print(" ".join(full))
```

Replace `$ARGUMENTS` with the actual URL or file path from the user's input. Write the script to a temp file and run it with `py` and a 300000ms timeout.

### Step 2: Display the Transcript

Show the user:
1. **Source and duration** — `@username | Xs` for IG posts, or `filename.ext | Xs` for local files.
2. **Timestamped transcript** — each segment with `[start - end]`
3. **Clean transcript** — flowing text, no timestamps

That's it. No analysis, no review, no formatting unless asked.
