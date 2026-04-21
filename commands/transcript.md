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

### FIRST-TIME SETUP CHECK (URL mode only)

Before running the script on an Instagram URL, check whether the user has `APIFY_TOKEN` set in their environment. You can test with `echo $APIFY_TOKEN` (bash) or `echo %APIFY_TOKEN%` (cmd) — if it's empty or the script below exits with the "APIFY_TOKEN not set" error, **do not keep retrying**. Instead, walk the user through first-time setup:

> "Looks like you haven't set up your Apify API token yet — this skill uses Apify to scrape the Instagram post. Here's a 2-minute setup:
>
> **1. Create a free Apify account**
> Go to https://console.apify.com/sign-up and sign up (free tier includes ~$5/month of credit, plenty for transcripts).
>
> **2. Grab your API token**
> After signing in, go to https://console.apify.com/settings/integrations and copy your **Personal API token** (starts with `apify_api_...`).
>
> **3. Set it as an environment variable**
> - **Windows (PowerShell):** Run `setx APIFY_TOKEN "apify_api_your_token_here"`, then close and reopen your terminal.
> - **macOS/Linux:** Add `export APIFY_TOKEN="apify_api_your_token_here"` to `~/.zshrc` (Mac) or `~/.bashrc` (Linux), then run `source ~/.zshrc` or open a new terminal.
>
> **4. Install Python dependencies (one-time)**
> `pip install requests faster-whisper`
>
> Once that's done, re-run `/transcript <url>` and you're set."

Then stop and wait for them to complete setup. Do not re-run the script until they confirm.

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
import requests, json, time, os
from pathlib import Path

# Config
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
