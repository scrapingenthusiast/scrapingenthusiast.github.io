---
layout: post
title: "TikTok to YouTube Pipeline Part 1: Searching TikTok with the Apify Python Client"
date: 2026-03-24 08:00:00 +0700
categories: [TikTok, Web Scraping, YouTube]
tags: [apify, tiktok-api, python, tutorial]
permalink: /tiktok/tiktok-youtube-part1-searching-api/
---

This is **Part 1** of a 4-part series on building an automated YouTube news channel powered by TikTok data. In this installment, we'll write the Python code that searches TikTok using the [Advanced TikTok Search API Actor](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m) on Apify and saves structured results to a local JSON file.

> **Series Navigation**
> - **Part 1: Searching TikTok with the Apify Python Client** ← You are here
> - [Part 2: Downloading & Organizing Video Files](/tiktok/tiktok-youtube-part2-downloading-videos/)
> - [Part 3: Auto-Generating News Scripts with AI](/tiktok/tiktok-youtube-part3-ai-narration/)
> - [Part 4: Scheduling & Publishing to YouTube](/tiktok/tiktok-youtube-part4-scheduling-publishing/)

## Prerequisites

- Python 3.10+
- An [Apify account](https://apify.com?fpr=7hce1m) (free tier works for testing)
- Your Apify API token (found in **Settings → Integrations**)

## Step 1: Install Dependencies

```bash
pip install apify-client python-dotenv
```

Create a `.env` file in your project root:

```env
APIFY_TOKEN=your_apify_api_token_here
```

## Step 2: Define Your Search Configuration

Create a file called `config.py` to centralize your search parameters. This makes it easy to manage multiple hotspot topics:

```python
# config.py

HOTSPOT_SEARCHES = [
    {
        "keyword": "Ukraine war",
        "region": "UA",
        "sortType": 2,       # Most recent
        "publishTime": "WEEK",
        "limit": 50,
    },
    {
        "keyword": "earthquake Turkey",
        "region": "TR",
        "sortType": 2,
        "publishTime": "YESTERDAY",
        "limit": 50,
    },
    {
        "keyword": "protest France",
        "region": "FR",
        "sortType": 1,       # Most liked
        "publishTime": "WEEK",
        "limit": 30,
    },
]
```

Each dictionary maps directly to the input schema of the [Advanced TikTok Search API Actor](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m). The key parameters are:

| Parameter     | Purpose                                           |
|---------------|---------------------------------------------------|
| `keyword`     | The search term (supports spaces)                 |
| `region`      | Two-letter country code to localize results       |
| `sortType`    | `0` = Relevance, `1` = Most Liked, `2` = Most Recent |
| `publishTime` | Time filter: `YESTERDAY`, `WEEK`, `MONTH`, etc.  |
| `limit`       | Soft cap on the number of videos returned         |

## Step 3: Write the Search Script

Create `search_tiktok.py`:

```python
# search_tiktok.py

import json
import os
from datetime import datetime
from pathlib import Path

from apify_client import ApifyClient
from dotenv import load_dotenv

from config import HOTSPOT_SEARCHES

load_dotenv()

ACTOR_ID = "novi/advanced-search-tiktok-api"
OUTPUT_DIR = Path("data/raw")


def run_search(client: ApifyClient, search_params: dict) -> list[dict]:
    """Run a single TikTok search and return the results."""
    print(f"🔍 Searching: '{search_params['keyword']}' "
          f"(region={search_params.get('region', 'ANY')})...")

    run = client.actor(ACTOR_ID).call(run_input=search_params)
    items = list(client.dataset(run["defaultDatasetId"]).iterate_items())

    print(f"   ✅ Found {len(items)} videos.")
    return items


def extract_essential_fields(video: dict) -> dict:
    """Extract only the fields we need for the pipeline."""
    stats = video.get("statistics", {})
    author = video.get("author", {})
    play_addr = video.get("video", {}).get("play_addr", {})
    download_addr = video.get("video", {}).get("download_addr", {})

    return {
        "aweme_id": video.get("aweme_id"),
        "description": video.get("desc", ""),
        "share_url": video.get("share_url", ""),
        "create_time": video.get("create_time"),
        "region": video.get("region"),
        # Author info
        "author_username": author.get("unique_id"),
        "author_nickname": author.get("nickname"),
        # Statistics
        "play_count": stats.get("play_count", 0),
        "like_count": stats.get("digg_count", 0),
        "comment_count": stats.get("comment_count", 0),
        "share_count": stats.get("share_count", 0),
        # Video URLs
        "video_url_no_watermark": (
            play_addr.get("url_list", [None])[0]
        ),
        "video_url_watermark": (
            download_addr.get("url_list", [None])[0]
        ),
        "video_duration_ms": video.get("video", {}).get("duration"),
        # Hashtags
        "hashtags": [
            t.get("hashtag_name", "")
            for t in video.get("text_extra", [])
            if t.get("type") == 1
        ],
    }


def main():
    token = os.environ.get("APIFY_TOKEN")
    if not token:
        raise SystemExit("❌ APIFY_TOKEN not set. Check your .env file.")

    client = ApifyClient(token)
    OUTPUT_DIR.mkdir(parents=True, exist_ok=True)
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")

    all_results = []

    for search_params in HOTSPOT_SEARCHES:
        raw_videos = run_search(client, search_params)
        extracted = [extract_essential_fields(v) for v in raw_videos]

        # Tag each result with the original search keyword
        for item in extracted:
            item["_search_keyword"] = search_params["keyword"]

        all_results.extend(extracted)

    # Save to a single JSON file
    output_path = OUTPUT_DIR / f"tiktok_results_{timestamp}.json"
    with open(output_path, "w", encoding="utf-8") as f:
        json.dump(all_results, f, ensure_ascii=False, indent=2)

    print(f"\n📁 Saved {len(all_results)} videos to {output_path}")


if __name__ == "__main__":
    main()
```

## Step 4: Run It

```bash
python search_tiktok.py
```

Expected output:

```
🔍 Searching: 'Ukraine war' (region=UA)...
   ✅ Found 50 videos.
🔍 Searching: 'earthquake Turkey' (region=TR)...
   ✅ Found 42 videos.
🔍 Searching: 'protest France' (region=FR)...
   ✅ Found 30 videos.

📁 Saved 122 videos to data/raw/tiktok_results_20260324_080000.json
```

## Understanding the Output

Each entry in the JSON file looks like this:

```json
{
  "aweme_id": "7229167805625847041",
  "description": "Breaking: massive protest in central Paris #protest #france",
  "share_url": "https://www.tiktok.com/@user/video/7229167805625847041",
  "create_time": 1683171893,
  "region": "FR",
  "author_username": "news_reporter_01",
  "author_nickname": "Breaking News FR",
  "play_count": 585709,
  "like_count": 25006,
  "comment_count": 183,
  "share_count": 492,
  "video_url_no_watermark": "https://v19.tiktokcdn-us.com/...",
  "video_url_watermark": "https://v19.tiktokcdn-us.com/...",
  "video_duration_ms": 54635,
  "hashtags": ["protest", "france"],
  "_search_keyword": "protest France"
}
```

The `video_url_no_watermark` field is the key asset — this is the clean video URL we'll download in Part 2.

## What's Next?

In [Part 2](/tiktok/tiktok-youtube-part2-downloading-videos/), we'll write an async downloader that fetches all these watermark-free videos, organizes them by topic and date, and handles retries for failed downloads.

> **Series Navigation**
> - **Part 1: Searching TikTok with the Apify Python Client** ← You are here
> - [Part 2: Downloading & Organizing Video Files](/tiktok/tiktok-youtube-part2-downloading-videos/)
> - [Part 3: Auto-Generating News Scripts with AI](/tiktok/tiktok-youtube-part3-ai-narration/)
> - [Part 4: Scheduling & Publishing to YouTube](/tiktok/tiktok-youtube-part4-scheduling-publishing/)
