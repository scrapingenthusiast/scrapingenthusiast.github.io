---
layout: post
title: "TikTok to YouTube Pipeline Part 2: Downloading & Organizing Video Files"
date: 2026-03-25 08:00:00 +0700
categories: [TikTok, Web Scraping, YouTube]
tags: [apify, tiktok-api, python, tutorial]
permalink: /tiktok/tiktok-youtube-part2-downloading-videos/
---

This is **Part 2** of our series on building an automated YouTube news channel with TikTok data. In [Part 1](/tiktok/tiktok-youtube-part1-searching-api/), we used the [Advanced TikTok Search API Actor](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m) to search TikTok and save structured JSON results. Now, we'll write an **async video downloader** that fetches watermark-free clips and organizes them by topic.

> **Series Navigation**
> - [Part 1: Searching TikTok with the Apify Python Client](/tiktok/tiktok-youtube-part1-searching-api/)
> - **Part 2: Downloading & Organizing Video Files** ← You are here
> - [Part 3: Auto-Generating News Scripts with AI](/tiktok/tiktok-youtube-part3-ai-narration/)
> - [Part 4: Scheduling & Publishing to YouTube](/tiktok/tiktok-youtube-part4-scheduling-publishing/)

## Prerequisites

- The JSON output file from Part 1
- Python 3.10+

## Step 1: Install Dependencies

```bash
pip install httpx tqdm
```

We'll use `httpx` for its first-class async support and `tqdm` for progress bars.

## Step 2: The Async Video Downloader

Create `download_videos.py`:

```python
# download_videos.py

import asyncio
import json
import re
import sys
from pathlib import Path

import httpx
from tqdm.asyncio import tqdm_asyncio

MAX_CONCURRENT = 5
DOWNLOAD_DIR = Path("data/videos")
TIMEOUT = 60.0


def sanitize_filename(name: str) -> str:
    """Remove unsafe characters from a filename."""
    return re.sub(r'[^\w\-.]', '_', name)[:80]


def load_results(json_path: str) -> list[dict]:
    """Load the search results JSON file."""
    with open(json_path, "r", encoding="utf-8") as f:
        return json.load(f)


def build_download_manifest(results: list[dict]) -> list[dict]:
    """Build a list of downloads, skipping entries without a valid URL."""
    manifest = []
    seen_ids = set()

    for item in results:
        aweme_id = item.get("aweme_id")
        url = item.get("video_url_no_watermark")

        if not aweme_id or not url:
            continue
        if aweme_id in seen_ids:
            continue

        seen_ids.add(aweme_id)
        keyword = sanitize_filename(item.get("_search_keyword", "unknown"))
        filename = f"{aweme_id}.mp4"

        manifest.append({
            "aweme_id": aweme_id,
            "url": url,
            "keyword": keyword,
            "output_dir": DOWNLOAD_DIR / keyword,
            "output_path": DOWNLOAD_DIR / keyword / filename,
            "description": item.get("description", ""),
        })

    return manifest


async def download_one(
    client: httpx.AsyncClient,
    task: dict,
    semaphore: asyncio.Semaphore,
) -> dict:
    """Download a single video file with retry logic."""
    task["output_dir"].mkdir(parents=True, exist_ok=True)

    if task["output_path"].exists():
        return {"aweme_id": task["aweme_id"], "status": "skipped"}

    async with semaphore:
        for attempt in range(3):
            try:
                async with client.stream("GET", task["url"],
                                         timeout=TIMEOUT) as resp:
                    resp.raise_for_status()
                    with open(task["output_path"], "wb") as f:
                        async for chunk in resp.aiter_bytes(
                            chunk_size=1024 * 64
                        ):
                            f.write(chunk)

                return {"aweme_id": task["aweme_id"], "status": "ok"}

            except (httpx.HTTPStatusError, httpx.ReadTimeout) as e:
                if attempt == 2:
                    return {
                        "aweme_id": task["aweme_id"],
                        "status": "failed",
                        "error": str(e),
                    }
                await asyncio.sleep(2 ** attempt)

    return {"aweme_id": task["aweme_id"], "status": "failed"}


async def download_all(manifest: list[dict]) -> list[dict]:
    """Download all videos concurrently with a semaphore limit."""
    semaphore = asyncio.Semaphore(MAX_CONCURRENT)
    headers = {
        "User-Agent": (
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
            "AppleWebKit/537.36"
        ),
        "Referer": "https://www.tiktok.com/",
    }

    async with httpx.AsyncClient(
        headers=headers, follow_redirects=True
    ) as client:
        tasks = [
            download_one(client, task, semaphore)
            for task in manifest
        ]
        results = await tqdm_asyncio.gather(
            *tasks, desc="Downloading videos"
        )

    return results


def save_metadata(manifest: list[dict], output_path: Path):
    """Save a metadata index alongside the downloaded videos."""
    metadata = []
    for task in manifest:
        metadata.append({
            "aweme_id": task["aweme_id"],
            "keyword": task["keyword"],
            "description": task["description"],
            "local_path": str(task["output_path"]),
        })

    with open(output_path, "w", encoding="utf-8") as f:
        json.dump(metadata, f, ensure_ascii=False, indent=2)


def main():
    if len(sys.argv) < 2:
        print("Usage: python download_videos.py <path_to_results.json>")
        sys.exit(1)

    json_path = sys.argv[1]
    results = load_results(json_path)
    manifest = build_download_manifest(results)

    print(f"📋 {len(manifest)} unique videos to download "
          f"(from {len(results)} search results)")

    download_results = asyncio.run(download_all(manifest))

    ok = sum(1 for r in download_results if r["status"] == "ok")
    skipped = sum(1 for r in download_results if r["status"] == "skipped")
    failed = sum(1 for r in download_results if r["status"] == "failed")

    print(f"\n✅ Downloaded: {ok}  ⏭️ Skipped: {skipped}  ❌ Failed: {failed}")

    # Save metadata index
    meta_path = DOWNLOAD_DIR / "metadata_index.json"
    save_metadata(manifest, meta_path)
    print(f"📁 Metadata saved to {meta_path}")


if __name__ == "__main__":
    main()
```

## Step 3: Run the Downloader

```bash
python download_videos.py data/raw/tiktok_results_20260324_080000.json
```

Expected output:

```
📋 118 unique videos to download (from 122 search results)
Downloading videos: 100%|████████████████| 118/118 [01:42<00:00]

✅ Downloaded: 112  ⏭️ Skipped: 0  ❌ Failed: 6
📁 Metadata saved to data/videos/metadata_index.json
```

## Directory Structure After Download

```
data/
└── videos/
    ├── metadata_index.json
    ├── Ukraine_war/
    │   ├── 7229167805625847041.mp4
    │   ├── 7229167805625847042.mp4
    │   └── ...
    ├── earthquake_Turkey/
    │   ├── 7229167805625847050.mp4
    │   └── ...
    └── protest_France/
        ├── 7229167805625847060.mp4
        └── ...
```

Videos are automatically grouped by keyword, making it easy to find footage for a specific news topic.

## Key Design Decisions

### Why Async?

TikTok video CDN URLs are short-lived and can be slow from certain regions. Downloading 100+ videos sequentially could take 30+ minutes. With `asyncio` and a concurrency limit of 5, we typically finish in under 3 minutes.

### Duplicate Detection

The `build_download_manifest` function tracks `aweme_id` values in a set, ensuring we never download the same video twice — even if it appears in multiple search results.

### Retry with Exponential Backoff

CDN URLs can occasionally return 403 or timeout. Our downloader retries up to 3 times with exponential backoff (1s, 2s, 4s) before marking a video as failed.

### Metadata Index

The `metadata_index.json` file serves as a manifest for the next steps in the pipeline. It maps each video to its local path, original keyword, and description — exactly the data we'll need to generate narration scripts in Part 3.

## What's Next?

In [Part 3](/tiktok/tiktok-youtube-part3-ai-narration/), we'll use the downloaded videos and their descriptions to auto-generate news narration scripts with AI (OpenAI / Gemini), convert them to speech, and compile everything into a single news segment using FFmpeg.

> **Series Navigation**
> - [Part 1: Searching TikTok with the Apify Python Client](/tiktok/tiktok-youtube-part1-searching-api/)
> - **Part 2: Downloading & Organizing Video Files** ← You are here
> - [Part 3: Auto-Generating News Scripts with AI](/tiktok/tiktok-youtube-part3-ai-narration/)
> - [Part 4: Scheduling & Publishing to YouTube](/tiktok/tiktok-youtube-part4-scheduling-publishing/)
