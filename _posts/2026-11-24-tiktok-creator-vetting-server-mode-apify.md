---
layout: post
title: "Vet TikTok Creators and Confirm Sponsored Posts Instantly: TikTok Profile Videos API in Server Mode"
date: 2026-11-24 07:20:00 +0700
permalink: /apify/tiktok-creator-vetting-server-mode/
categories: [Social Media, Influencer Marketing]
tags: [TikTok, Apify, Server Actors, Influencer Marketing, Creator Vetting, n8n, Zapier, Make]
author: Scraping Enthusiast
description: "A no-code guide for influencer and partnership teams: pull a creator's latest TikTok videos with one web request to vet them before signing, or to check that the sponsored video actually went live, using TikTok Profile Videos API's server mode."
---

# Vet TikTok Creators and Confirm Sponsored Posts Instantly: TikTok Profile Videos API in Server Mode

Influencer teams ask two questions over and over. Before the deal: *what has this creator been posting lately, and does anyone watch it?* After the deal: *did the sponsored video go live, and how is it doing?*

Both questions are about one creator and their recent videos, and someone usually needs the answer that day. [TikTok Profile Videos API](https://apify.com/novi/tiktok-user-api?fpr=7hce1m) now has a server mode that fits this well. You send a username and get back that creator's latest videos in the same request.

This is the third post in a short series on TikTok actors that gained server mode. The other two cover [brand mention checks by keyword](/apify/tiktok-brand-mention-checks-server-mode/) and [turning any TikTok link into data](/apify/paste-tiktok-link-get-data-server-mode/).

## What server mode changes for you

With a normal Apify run, you start a job, wait for it, then open or download a dataset. Automation tools wrap the same three steps, so a "quick look" at a creator still means waiting on a full run.

In server mode (Apify's name is Standby, or "Server Actor"), the actor waits at a web address. You call that address with a username and the videos come back in the response. You don't start a run, wait for it, or fetch a dataset afterwards.

Keep two things in mind:

- Nothing is saved on Apify. If you want a history of a creator's numbers, write them to your sheet, Airtable base or influencer tool in the next step.
- The first call after a quiet period is slower while Apify starts the actor. It then stays warm for about five minutes between calls.

## Instant lookup or scheduled run?

Use server mode when someone is waiting:

- A new creator lands in your influencer CRM or Airtable, and a vetting card should fill in automatically.
- A "latest posts" widget on an internal creator profile page.
- The morning a sponsored video is due: "has @creator posted it yet?"
- An AI assistant tool for questions like "what has @x posted lately?"

Use a scheduled batch run for the roster: tracking 150 creators every week, or pulling more than 100 videos per creator for a deeper audit. Server mode handles one creator per request, up to 100 videos, and the normal input run still covers everything else.

## The address and the three ways to name a creator

```
https://novi--tiktok-user-api.apify.actor/posts
```

`/posts` is the only endpoint. Give it exactly one of these:

| Parameter | Example |
|---|---|
| `username` | `nike` |
| `userId` | `6714563164218679553` |
| `url` | `https://www.tiktok.com/@nike` |

Then optionally `limit`, the number of latest videos you want, from 2 to 100 (default 20).

In practice, use `username` when your CRM stores handles and `url` when people paste profile links. Use `userId` if you already have TikTok's numeric ID, for example from an earlier lookup. Unlike a handle, the ID doesn't change if the creator renames their account.

The response is a list of the creator's latest videos, the same objects a normal run puts in a dataset.

## Store the token once, not in every doc

Each request needs your Apify API token as a header:

```
Authorization: Bearer <your token>
```

The token can also go in the URL as `?token=`, but influencer workflows get shared with agencies, freelancers and clients, and a token in a URL travels with every copy. Keep it in your automation tool's credential store, or in the `APIFY_TOKEN` environment variable if you use code.

## Building the vetting step without code

**n8n**, HTTP Request node:

1. Method `GET`, URL `https://novi--tiktok-user-api.apify.actor/posts`
2. Authentication: Header Auth credential, name `Authorization`, value `Bearer <token>`.
3. Query parameters: `username` (mapped from the CRM field) and `limit` = `30`.
4. Options: timeout 300000 ms. If you want to branch on errors, turn on the option that includes the response status.

**Make**: HTTP "Make a request", same method, URL and query string, `Authorization` header from a saved connection, longer timeout.

**Zapier**: Webhooks by Zapier, GET, same settings. Keep `limit` modest because Zapier steps have their own time limits.

Then do the vetting itself with the tool's built-in helpers (n8n's Code or Aggregate nodes, Make's array aggregator and math functions, Zapier Formatter):

- Activity: the newest and oldest `create_time` in the list tell you how often they post.
- Reach: average `statistics.play_count` across the videos.
- Engagement: total `statistics.digg_count` (likes), `comment_count` and `share_count`, compared with plays.
- Fit: skim the `desc` captions for topics, competitor brands, or other sponsorship tags.

Write those numbers back onto the creator's record. Your team decides what counts as a good number.

Checking the sponsored post uses the same request. Filter the returned videos for captions that contain your campaign hashtag or brand handle. If one matches, post its link and play count to the campaign channel. If none matches on the due date, ping the creator manager.

## Copy-paste code

curl:

```bash
curl -s -H "Authorization: Bearer $APIFY_TOKEN" \
  "https://novi--tiktok-user-api.apify.actor/posts?username=nike&limit=30"
```

A Python script that produces a quick vetting summary and checks for a campaign tag:

```python
import os
import requests

CAMPAIGN_TAG = "#yourcampaign"

resp = requests.get(
    "https://novi--tiktok-user-api.apify.actor/posts",
    params={"username": "nike", "limit": 30},
    headers={"Authorization": f"Bearer {os.environ['APIFY_TOKEN']}"},
    timeout=300,
)

if resp.status_code == 404:
    print("Creator not found. Check the handle.")
elif resp.status_code == 402:
    print("Apify spending limit reached. Raise it before retrying.")
elif resp.status_code != 200:
    print("Error", resp.status_code, resp.text[:200])
else:
    videos = resp.json()
    plays = [v.get("statistics", {}).get("play_count", 0) for v in videos]
    print("Videos returned:", len(videos))
    if plays:
        print("Average plays:", round(sum(plays) / len(plays)))
    sponsored = [v for v in videos if CAMPAIGN_TAG.lower() in (v.get("desc") or "").lower()]
    if sponsored:
        v = sponsored[0]
        print("Sponsored post found:", v.get("aweme_id"), v.get("statistics", {}).get("play_count"), "plays")
    else:
        print("No video with", CAMPAIGN_TAG, "in the latest", len(videos))
```

## What each error means

- `200`: success. You get a list of the creator's latest videos.
- `400`: the request is wrong. Usually you sent none of `username` / `userId` / `url`, or more than one. Read the error message and fix the input.
- `404` with "User not found or could not be resolved.": the creator couldn't be found. Check the handle for typos or a stray `@`, and check whether they changed their name or deleted the account. In a vetting flow, mark the record "check handle" so you don't keep retrying.
- `402`: your Apify account's spending limit has been reached. Increase it or wait for it to reset. Retrying won't fix it.
- `502`: the request to TikTok failed this time. Try again after a short wait.

## Limits and cost

- One creator per request, 2 to 100 of their latest videos.
- For many creators at once, or more than 100 videos, use the normal input run, which saves to a dataset.
- Apify gives a server-mode request about five minutes to finish, so give your HTTP step a timeout of that length.
- Results aren't stored. Save what you need.
- You pay per video returned (`result-item`), and only for what you actually receive. See the [Store page](https://apify.com/novi/tiktok-user-api?fpr=7hce1m) for current pricing.

## More reading

The actor's Server tab in Apify Console shows the endpoint reference. To decide which lookups should be instant and which can wait for a schedule, start with [real-time vs scheduled social data](/apify/real-time-vs-scheduled-social-data/) on this blog.
