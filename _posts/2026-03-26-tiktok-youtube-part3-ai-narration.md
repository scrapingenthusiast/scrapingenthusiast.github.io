---
layout: post
title: "TikTok to YouTube Pipeline Part 3: Auto-Generating News Scripts with AI"
date: 2026-03-26 08:00:00 +0700
categories: [TikTok, Web Scraping, YouTube]
tags: [apify, tiktok-api, python, openai, ffmpeg, tutorial]
permalink: /tiktok/tiktok-youtube-part3-ai-narration/
---

This is **Part 3** of our series on building an automated YouTube news channel from TikTok data. In [Part 2](/tiktok/tiktok-youtube-part2-downloading-videos/), we downloaded watermark-free videos and organized them by topic. Now we'll turn those raw clips into a polished news segment by **generating narration scripts with AI**, converting them to **text-to-speech audio**, and **compiling everything with FFmpeg**.

> **Series Navigation**
> - [Part 1: Searching TikTok with the Apify Python Client](/tiktok/tiktok-youtube-part1-searching-api/)
> - [Part 2: Downloading & Organizing Video Files](/tiktok/tiktok-youtube-part2-downloading-videos/)
> - **Part 3: Auto-Generating News Scripts with AI** ← You are here
> - [Part 4: Scheduling & Publishing to YouTube](/tiktok/tiktok-youtube-part4-scheduling-publishing/)

## Prerequisites

- Downloaded videos from Part 2
- Python 3.10+
- FFmpeg installed on your system (`brew install ffmpeg` on macOS)
- An OpenAI API key

## Step 1: Install Dependencies

```bash
pip install openai ffmpeg-python
```

Add your OpenAI key to `.env`:

```env
OPENAI_API_KEY=sk-your_key_here
```

## Step 2: Generate Narration Scripts from Video Descriptions

The TikTok video descriptions (`desc` field) captured in Part 1 contain valuable context — hashtags, locations, and brief captions. We'll feed a batch of these to an LLM to produce a professional news script.

Create `generate_script.py`:

```python
# generate_script.py

import json
import os
from pathlib import Path

from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()

METADATA_PATH = Path("data/videos/metadata_index.json")
SCRIPTS_DIR = Path("data/scripts")

SYSTEM_PROMPT = """You are a professional news anchor script writer.
Given a list of TikTok video descriptions about a specific topic,
write a concise, factual news narration script (60-90 seconds when
read aloud). The script should:
- Start with a strong headline opener
- Summarize the key events shown in the videos
- Maintain a neutral, journalistic tone
- End with a brief outlook or call to stay updated
Do NOT mention TikTok or social media in the script.
Output ONLY the script text, no titles or formatting."""


def load_metadata() -> dict[str, list[dict]]:
    """Load metadata and group by keyword."""
    with open(METADATA_PATH, "r", encoding="utf-8") as f:
        metadata = json.load(f)

    grouped = {}
    for item in metadata:
        keyword = item["keyword"]
        grouped.setdefault(keyword, []).append(item)

    return grouped


def generate_script_for_topic(
    client: OpenAI,
    keyword: str,
    videos: list[dict],
) -> str:
    """Generate a narration script for a topic using the LLM."""
    # Take the top 15 descriptions to stay within token limits
    descriptions = [v["description"] for v in videos[:15] if v["description"]]
    descriptions_text = "\n".join(
        f"- {desc}" for desc in descriptions
    )

    user_prompt = (
        f"Topic: {keyword.replace('_', ' ')}\n\n"
        f"Video descriptions from the field:\n{descriptions_text}\n\n"
        f"Write the news narration script."
    )

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": user_prompt},
        ],
        temperature=0.7,
        max_tokens=500,
    )

    return response.choices[0].message.content.strip()


def main():
    client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
    grouped = load_metadata()

    SCRIPTS_DIR.mkdir(parents=True, exist_ok=True)

    for keyword, videos in grouped.items():
        print(f"✍️  Generating script for: {keyword}...")
        script = generate_script_for_topic(client, keyword, videos)

        script_path = SCRIPTS_DIR / f"{keyword}_script.txt"
        script_path.write_text(script, encoding="utf-8")
        print(f"   📝 Saved to {script_path}")
        print(f"   Preview: {script[:120]}...\n")


if __name__ == "__main__":
    main()
```

## Step 3: Convert Scripts to Speech

Create `generate_tts.py`:

```python
# generate_tts.py

import os
from pathlib import Path

from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()

SCRIPTS_DIR = Path("data/scripts")
AUDIO_DIR = Path("data/audio")


def main():
    client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
    AUDIO_DIR.mkdir(parents=True, exist_ok=True)

    script_files = sorted(SCRIPTS_DIR.glob("*_script.txt"))

    for script_path in script_files:
        keyword = script_path.stem.replace("_script", "")
        audio_path = AUDIO_DIR / f"{keyword}_narration.mp3"

        if audio_path.exists():
            print(f"⏭️  Skipping {keyword} (audio exists)")
            continue

        print(f"🔊 Generating TTS for: {keyword}...")
        script_text = script_path.read_text(encoding="utf-8")

        response = client.audio.speech.create(
            model="tts-1",
            voice="onyx",  # Deep, authoritative news voice
            input=script_text,
        )

        response.stream_to_file(str(audio_path))
        print(f"   ✅ Saved to {audio_path}")


if __name__ == "__main__":
    main()
```

## Step 4: Compile Videos with FFmpeg

Now we bring it all together — combine downloaded video clips with the AI narration audio into a single news segment.

Create `compile_video.py`:

```python
# compile_video.py

import json
import subprocess
from pathlib import Path

VIDEOS_DIR = Path("data/videos")
AUDIO_DIR = Path("data/audio")
OUTPUT_DIR = Path("data/output")
METADATA_PATH = VIDEOS_DIR / "metadata_index.json"


def get_video_duration(video_path: str) -> float:
    """Get video duration in seconds using ffprobe."""
    result = subprocess.run(
        [
            "ffprobe", "-v", "error",
            "-show_entries", "format=duration",
            "-of", "default=noprint_wrappers=1:nokey=1",
            video_path,
        ],
        capture_output=True, text=True,
    )
    return float(result.stdout.strip())


def compile_topic(keyword: str, video_paths: list[str]):
    """Compile videos for a single topic into one news segment."""
    audio_path = AUDIO_DIR / f"{keyword}_narration.mp3"

    if not audio_path.exists():
        print(f"⚠️  No narration audio for {keyword}, skipping.")
        return

    # Filter to videos that actually exist and take the top 5
    existing = [p for p in video_paths if Path(p).exists()][:5]
    if not existing:
        print(f"⚠️  No videos found for {keyword}, skipping.")
        return

    OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

    # Create a concat file for FFmpeg
    concat_file = OUTPUT_DIR / f"{keyword}_concat.txt"
    with open(concat_file, "w") as f:
        for vp in existing:
            f.write(f"file '{Path(vp).absolute()}'\n")

    output_path = OUTPUT_DIR / f"{keyword}_news_segment.mp4"

    # Step 1: Concatenate video clips
    temp_video = OUTPUT_DIR / f"{keyword}_temp_concat.mp4"
    subprocess.run([
        "ffmpeg", "-y",
        "-f", "concat", "-safe", "0",
        "-i", str(concat_file),
        "-c:v", "libx264",
        "-crf", "23",
        "-preset", "fast",
        "-vf", "scale=1920:1080:force_original_aspect_ratio=decrease,"
               "pad=1920:1080:(ow-iw)/2:(oh-ih)/2",
        "-r", "30",
        "-an",  # Remove original audio
        str(temp_video),
    ], check=True)

    # Step 2: Overlay narration audio
    subprocess.run([
        "ffmpeg", "-y",
        "-i", str(temp_video),
        "-i", str(audio_path),
        "-c:v", "copy",
        "-c:a", "aac",
        "-b:a", "192k",
        "-shortest",
        str(output_path),
    ], check=True)

    # Cleanup
    temp_video.unlink(missing_ok=True)
    concat_file.unlink(missing_ok=True)

    duration = get_video_duration(str(output_path))
    print(f"🎬 {keyword}: {output_path} ({duration:.1f}s)")


def main():
    with open(METADATA_PATH, "r", encoding="utf-8") as f:
        metadata = json.load(f)

    # Group by keyword
    grouped = {}
    for item in metadata:
        keyword = item["keyword"]
        grouped.setdefault(keyword, []).append(item["local_path"])

    for keyword, paths in grouped.items():
        print(f"\n🎞️  Compiling: {keyword}")
        compile_topic(keyword, paths)

    print(f"\n✅ All segments compiled in {OUTPUT_DIR}/")


if __name__ == "__main__":
    main()
```

## Step 5: Run the Full Post-Processing Pipeline

```bash
# Generate narration scripts
python generate_script.py

# Convert to speech
python generate_tts.py

# Compile final videos
python compile_video.py
```

Expected output:

```
✍️  Generating script for: Ukraine_war...
   📝 Saved to data/scripts/Ukraine_war_script.txt
🔊 Generating TTS for: Ukraine_war...
   ✅ Saved to data/audio/Ukraine_war_narration.mp3

🎞️  Compiling: Ukraine_war
🎬 Ukraine_war: data/output/Ukraine_war_news_segment.mp4 (87.3s)

✅ All segments compiled in data/output/
```

## What's Next?

In [Part 4](/tiktok/tiktok-youtube-part4-scheduling-publishing/), we'll upload these compiled news segments to YouTube using the YouTube Data API, and set up Apify scheduling to run this entire pipeline automatically every 12 hours.

> **Looking for more TikTok data sources?** Beyond keyword search, you can also pull trending videos, user profiles, hashtags, and comments. Browse our full collection of [TikTok and Twitter scraping tools](/tools/) to expand your data pipeline.

> **Series Navigation**
> - [Part 1: Searching TikTok with the Apify Python Client](/tiktok/tiktok-youtube-part1-searching-api/)
> - [Part 2: Downloading & Organizing Video Files](/tiktok/tiktok-youtube-part2-downloading-videos/)
> - **Part 3: Auto-Generating News Scripts with AI** ← You are here
> - [Part 4: Scheduling & Publishing to YouTube](/tiktok/tiktok-youtube-part4-scheduling-publishing/)
