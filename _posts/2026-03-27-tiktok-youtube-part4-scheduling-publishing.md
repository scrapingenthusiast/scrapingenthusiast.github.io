---
layout: post
title: "TikTok to YouTube Pipeline Part 4: Scheduling & Publishing to YouTube"
date: 2026-03-27 08:00:00 +0700
categories: [TikTok, Web Scraping, YouTube]
tags: [apify, tiktok-api, python, youtube-api, tutorial]
permalink: /tiktok/tiktok-youtube-part4-scheduling-publishing/
---

This is the **final part** of our series on building an automated YouTube news channel from TikTok data. We've searched TikTok ([Part 1](/tiktok/tiktok-youtube-part1-searching-api/)), downloaded videos ([Part 2](/tiktok/tiktok-youtube-part2-downloading-videos/)), and generated AI narration ([Part 3](/tiktok/tiktok-youtube-part3-ai-narration/)). Now we'll tie it all together: **uploading to YouTube** and **scheduling the pipeline to run automatically**.

> **Series Navigation**
> - [Part 1: Searching TikTok with the Apify Python Client](/tiktok/tiktok-youtube-part1-searching-api/)
> - [Part 2: Downloading & Organizing Video Files](/tiktok/tiktok-youtube-part2-downloading-videos/)
> - [Part 3: Auto-Generating News Scripts with AI](/tiktok/tiktok-youtube-part3-ai-narration/)
> - **Part 4: Scheduling & Publishing to YouTube** ← You are here

## Prerequisites

- Compiled video segments from Part 3
- A Google Cloud project with YouTube Data API v3 enabled
- OAuth2 credentials (`client_secret.json`)

## Step 1: Install Dependencies

```bash
pip install google-auth google-auth-oauthlib google-api-python-client
```

## Step 2: YouTube OAuth2 Authentication

Create `youtube_auth.py`:

```python
# youtube_auth.py

import os
import pickle
from pathlib import Path

from google.auth.transport.requests import Request
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build

SCOPES = ["https://www.googleapis.com/auth/youtube.upload"]
TOKEN_PATH = Path("credentials/youtube_token.pickle")
CLIENT_SECRET_PATH = Path("credentials/client_secret.json")


def get_authenticated_service():
    """Authenticate and return a YouTube API service object."""
    credentials = None

    if TOKEN_PATH.exists():
        with open(TOKEN_PATH, "rb") as token:
            credentials = pickle.load(token)

    if not credentials or not credentials.valid:
        if credentials and credentials.expired and credentials.refresh_token:
            credentials.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file(
                str(CLIENT_SECRET_PATH), SCOPES
            )
            credentials = flow.run_local_server(port=8090)

        TOKEN_PATH.parent.mkdir(parents=True, exist_ok=True)
        with open(TOKEN_PATH, "wb") as token:
            pickle.dump(credentials, token)

    return build("youtube", "v3", credentials=credentials)
```

The first time you run this, a browser window will open for Google OAuth consent. After that, the token is cached locally.

## Step 3: Upload Videos to YouTube

Create `upload_youtube.py`:

```python
# upload_youtube.py

import sys
from datetime import datetime
from pathlib import Path

from googleapiclient.http import MediaFileUpload

from youtube_auth import get_authenticated_service

OUTPUT_DIR = Path("data/output")


def generate_metadata(keyword: str) -> dict:
    """Generate title, description, and tags for a YouTube upload."""
    topic = keyword.replace("_", " ").title()
    date_str = datetime.now().strftime("%B %d, %Y")

    return {
        "title": f"🔴 {topic} — Latest Updates | {date_str}",
        "description": (
            f"Today's coverage of {topic}. "
            f"This segment compiles the latest on-the-ground footage "
            f"and verified reports.\n\n"
            f"⚠️ Some footage may be graphic. Viewer discretion advised.\n\n"
            f"📌 Subscribe for daily global news updates.\n"
            f"🔔 Turn on notifications to never miss breaking news.\n\n"
            f"#GlobalNews #{keyword.replace('_', '')} #BreakingNews"
        ),
        "tags": [
            topic, "breaking news", "global news",
            keyword.replace("_", " "), "world news",
            "current events", "news today",
        ],
    }


def upload_video(
    youtube,
    video_path: str,
    title: str,
    description: str,
    tags: list[str],
    category_id: str = "25",  # News & Politics
    privacy: str = "private",
):
    """Upload a single video to YouTube."""
    body = {
        "snippet": {
            "title": title,
            "description": description,
            "tags": tags,
            "categoryId": category_id,
        },
        "status": {
            "privacyStatus": privacy,
            "selfDeclaredMadeForKids": False,
        },
    }

    media = MediaFileUpload(
        video_path,
        mimetype="video/mp4",
        resumable=True,
        chunksize=1024 * 1024 * 10,  # 10 MB chunks
    )

    request = youtube.videos().insert(
        part="snippet,status",
        body=body,
        media_body=media,
    )

    response = None
    while response is None:
        status, response = request.next_chunk()
        if status:
            pct = int(status.progress() * 100)
            print(f"   📤 Uploading... {pct}%")

    video_id = response["id"]
    print(f"   ✅ Uploaded: https://youtu.be/{video_id}")

    return video_id


def main():
    youtube = get_authenticated_service()

    video_files = sorted(OUTPUT_DIR.glob("*_news_segment.mp4"))

    if not video_files:
        print("❌ No compiled segments found in data/output/")
        sys.exit(1)

    print(f"📋 Found {len(video_files)} segment(s) to upload.\n")

    for video_path in video_files:
        keyword = video_path.stem.replace("_news_segment", "")
        meta = generate_metadata(keyword)

        print(f"🎬 Uploading: {meta['title']}")

        upload_video(
            youtube,
            str(video_path),
            title=meta["title"],
            description=meta["description"],
            tags=meta["tags"],
            privacy="private",  # Change to "public" when ready
        )

        print()

    print("🎉 All uploads complete!")


if __name__ == "__main__":
    main()
```

> **Tip:** Start with `privacy="private"` to review your uploads before making them public.

## Step 4: The Master Pipeline Script

Create `pipeline.py` to orchestrate the entire workflow in a single command:

```python
# pipeline.py

import subprocess
import sys
from pathlib import Path


def run_step(script: str, args: list[str] = None):
    """Run a pipeline step and exit on failure."""
    cmd = [sys.executable, script] + (args or [])
    print(f"\n{'='*60}")
    print(f"▶️  Running: {' '.join(cmd)}")
    print(f"{'='*60}\n")

    result = subprocess.run(cmd)
    if result.returncode != 0:
        print(f"\n❌ Step failed: {script}")
        sys.exit(1)


def find_latest_results() -> str:
    """Find the most recent search results JSON file."""
    results = sorted(Path("data/raw").glob("tiktok_results_*.json"))
    if not results:
        raise FileNotFoundError("No results found in data/raw/")
    return str(results[-1])


def main():
    print("🚀 Starting TikTok → YouTube Pipeline\n")

    # Step 1: Search TikTok
    run_step("search_tiktok.py")

    # Step 2: Download videos
    latest_json = find_latest_results()
    run_step("download_videos.py", [latest_json])

    # Step 3: Generate scripts
    run_step("generate_script.py")

    # Step 4: Generate TTS audio
    run_step("generate_tts.py")

    # Step 5: Compile videos
    run_step("compile_video.py")

    # Step 6: Upload to YouTube
    run_step("upload_youtube.py")

    print("\n🎉 Pipeline complete! Check your YouTube Studio.")


if __name__ == "__main__":
    main()
```

Run the entire pipeline:

```bash
python pipeline.py
```

## Step 5: Schedule with Apify

To run this pipeline automatically, you have two options:

### Option A: Apify Scheduler + Webhooks

1. Go to the [Advanced TikTok Search API Actor](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m) page on [Apify](https://apify.com?fpr=7hce1m).
2. Click **Schedule** and set it to run every 12 hours.
3. Add a **Webhook** that triggers on run completion, pointing to your server endpoint.
4. Your server receives the webhook, fetches the dataset, and runs steps 2–6.

### Option B: Cron Job (Self-Hosted)

For a simpler setup, add a cron job on your server:

```bash
# Run every 12 hours at 6:00 AM and 6:00 PM
0 6,18 * * * cd /path/to/project && /usr/bin/python3 pipeline.py >> logs/pipeline.log 2>&1
```

## Final Project Structure

```
tiktok-youtube-pipeline/
├── .env
├── config.py
├── search_tiktok.py          # Part 1
├── download_videos.py        # Part 2
├── generate_script.py        # Part 3
├── generate_tts.py           # Part 3
├── compile_video.py          # Part 3
├── youtube_auth.py           # Part 4
├── upload_youtube.py         # Part 4
├── pipeline.py               # Orchestrator
├── credentials/
│   ├── client_secret.json
│   └── youtube_token.pickle
├── data/
│   ├── raw/                  # JSON search results
│   ├── videos/               # Downloaded .mp4 files
│   ├── scripts/              # AI-generated narration
│   ├── audio/                # TTS audio files
│   └── output/               # Final compiled segments
└── logs/
```

## Conclusion

Over this 4-part series, we built a complete automated pipeline that:

1. **Searches** TikTok for breaking news footage using the [Advanced TikTok Search API](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m)
2. **Downloads** watermark-free video clips asynchronously
3. **Generates** professional narration scripts and TTS audio using AI
4. **Compiles** everything into polished news segments with FFmpeg
5. **Uploads** to YouTube with proper metadata
6. **Runs automatically** on a schedule

This pipeline transforms raw social media footage into a professional news channel — all with Python and a handful of APIs. The total cost per run is approximately $0.05 (Apify compute + OpenAI tokens), making it incredibly cost-effective for daily news production.

> **Series Navigation**
> - [Part 1: Searching TikTok with the Apify Python Client](/tiktok/tiktok-youtube-part1-searching-api/)
> - [Part 2: Downloading & Organizing Video Files](/tiktok/tiktok-youtube-part2-downloading-videos/)
> - [Part 3: Auto-Generating News Scripts with AI](/tiktok/tiktok-youtube-part3-ai-narration/)
> - **Part 4: Scheduling & Publishing to YouTube** ← You are here
