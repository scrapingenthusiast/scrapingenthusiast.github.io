---
layout: post
title: "Check TikTok Brand Mentions On Demand: Advanced Search TikTok API in Server Mode"
date: 2026-11-24 07:00:00 +0700
permalink: /apify/tiktok-brand-mention-checks-server-mode/
categories: [Social Media, Marketing Analytics]
tags: [TikTok, Apify, Server Actors, Keyword Monitoring, Brand Monitoring, n8n, Zapier, Make]
author: Scraping Enthusiast
description: "A no-code guide for marketers: ask TikTok about your brand, competitors or campaign keywords whenever you need to, and get the videos back in a single web request from Advanced Search TikTok API's new server mode."
---

# Check TikTok Brand Mentions On Demand: Advanced Search TikTok API in Server Mode

Yesterday I wrote about [choosing between a real-time lookup and a scheduled run](/apify/real-time-vs-scheduled-social-data/) for TikTok and X data. In that post I noted that the TikTok and X scrapers I use were regular batch actors, not Server Actors. Three of them have since gained server mode, so I'm writing one post for each. This first one covers [Advanced Search TikTok API](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m), the actor I use to search TikTok by keyword.

Below I set up a brand mention check you can run from n8n, Make or Zapier, with a little code at the end for anyone who wants it.

## What "server mode" means, without the jargon

A normal Apify run goes like this. You fill in an input form, press Start, wait, then open a dataset (a table of results) or download it as a spreadsheet. Automation tools do the same thing behind the scenes: start the run, wait, then fetch the results.

Server mode (Apify calls it Standby, or a "Server Actor") skips all that. The actor sits behind a fixed web address. You visit that address with your question in the URL, and the videos come back right away in the response, as JSON. You don't start a run or wait for one, and there's no dataset to go and collect.

Two things follow from that:

- Nothing gets saved on Apify. The results exist only in the reply. If you want a record, your workflow has to write them somewhere, such as a sheet, Airtable or your CRM.
- The first call after a quiet spell can be slower. Apify has to start the actor up again (a "cold start"). Once it's running, it stays warm for about five minutes between calls.

## When it's worth it, and when the scheduled run is still better

Use server mode when a person or a tool is waiting for the answer:

- A PR lead types "did anyone post about our recall today?" into a Slack command.
- A dashboard has a "refresh TikTok mentions" button.
- A campaign manager wants to compare three hashtags right now, before a call.
- An AI assistant needs a "search TikTok for X" tool it can call mid-conversation.

Keep the scheduled batch run when nobody is waiting, or when the job is big. The daily sweep that drops yesterday's mentions into a sheet at 6 a.m. doesn't benefit from server mode. Very large pulls also belong in a batch run: server mode has caps per request (more on those below), and the normal input-form run still works exactly as before.

## The address and what you can ask for

The server lives at:

```
https://novi--advanced-search-tiktok-api.apify.actor
```

The search page is `/search`. Everything goes in the URL as query parameters:

| Parameter | What it does | Default |
|---|---|---|
| `keyword` | One keyword. Repeat it for several: `keyword=nike&keyword=adidas` | none |
| `keywords` | Several keywords, comma-separated | none |
| `limit` | Videos per keyword, from 30 to 500 | 30 |
| `region` | Two-letter country code (IR, AQ and KP aren't supported) | GB |
| `sortType` | `0` relevance, `1` most liked, `2` most recent | 0 |
| `publishTime` | `ALL_TIME`, `YESTERDAY`, `WEEK`, `MONTH`, `THREE_MONTH`, `SIX_MONTH` | ALL_TIME |
| `includeSearchTerms` | `true` adds a `__search_terms` object to each video: the `keyword` that found it, plus the `sort_type`, `publish_time` and `region` used | off |
| `fetchUserBio` | `true` adds each creator's bio (slower, and billed extra per unique creator) | off |

For a brand check, the version I'd use is "most recent, last week, in my market":

```
/search?keyword=yourbrand&keyword=competitor&limit=30&sortType=2&publishTime=WEEK&region=US&includeSearchTerms=true
```

Turn `includeSearchTerms` on whenever you search for more than one keyword. Without it, you can't tell which keyword found which video once everything lands in one sheet. With it, you can group or filter rows by `__search_terms.keyword`.

There's also a `POST /search` version that takes the same fields as a JSON body, plus `customMapFunction`, which reshapes each video into a smaller object. Most no-code users won't need it. And if you open the bare address (`GET /`), you get a short description of the endpoints and limits.

## Keep your token out of shared docs

Every call needs your Apify API token, sent as a header:

```
Authorization: Bearer <your token>
```

You *can* put it in the URL as `?token=...`, but please don't. URLs end up in browser history, screenshots, shared Google Docs and Slack threads. In no-code tools, save the token once as a credential. In code, read it from an environment variable called `APIFY_TOKEN`.

## The no-code setup

All three tools have a generic HTTP request step, and that's all you need.

**n8n** (HTTP Request node):

1. Method: `GET`.
2. URL: `https://novi--advanced-search-tiktok-api.apify.actor/search`
3. Authentication: Generic Credential Type > Header Auth. Name `Authorization`, value `Bearer ` followed by your token. Save it as a credential so it isn't visible in the workflow.
4. Turn on "Send Query Parameters" and add `keyword` (twice if you want two), `limit`, `sortType`, `publishTime`, `region`, `includeSearchTerms`.
5. Under Options, raise the timeout to 300000 ms (five minutes).

**Make** (HTTP > "Make a request"): same method and URL, a header named `Authorization` with `Bearer <token>` stored in a connection or variable, query string items for the parameters, and a longer timeout in the module settings.

**Zapier** (Webhooks by Zapier, GET, or an API request step): same URL and parameters, header `Authorization`. Zapier steps have their own time limits, so keep `limit` low there and search for fewer keywords per call.

What comes back has two parts. `items` is the list of videos. `meta` is a short summary. Point your next step (Google Sheets "add row", Slack message, Airtable record) at `items`, and map fields like these:

- `desc`: the caption
- `author.unique_id`: the creator's handle
- `create_time`: when it was posted (a Unix timestamp)
- `statistics.play_count`, `statistics.digg_count` (likes), `statistics.comment_count`, `statistics.share_count`
- `aweme_id`: the video ID, useful for removing duplicates
- `__search_terms.keyword`: which of your keywords found the video (only when `includeSearchTerms` is on)

## Copy-paste code, if you prefer

The same check in curl:

```bash
curl -s -H "Authorization: Bearer $APIFY_TOKEN" \
  "https://novi--advanced-search-tiktok-api.apify.actor/search?keyword=yourbrand&keyword=competitor&limit=30&sortType=2&publishTime=WEEK&region=US&includeSearchTerms=true"
```

And a short Python script that prints a one-line summary per video and flags partial answers:

```python
import os
import requests

resp = requests.get(
    "https://novi--advanced-search-tiktok-api.apify.actor/search",
    params={
        "keyword": ["yourbrand", "competitor"],
        "limit": 30,
        "sortType": 2,
        "publishTime": "WEEK",
        "region": "US",
        "includeSearchTerms": "true",
    },
    headers={"Authorization": f"Bearer {os.environ['APIFY_TOKEN']}"},
    timeout=300,
)

if resp.status_code != 200:
    print("Error", resp.status_code, resp.json().get("error"))
else:
    data = resp.json()
    meta = data["meta"]
    if meta["partial"] or meta["chargeLimitReached"]:
        print("Heads up: this is only part of the results", meta)
    for v in data["items"]:
        stats = v.get("statistics", {})
        print(v.get("__search_terms", {}).get("keyword"), v.get("author", {}).get("unique_id"),
              stats.get("play_count"), (v.get("desc") or "")[:80])
```

`requests` turns the `keyword` list into `keyword=yourbrand&keyword=competitor` for you.

## When something goes wrong

The answers this actor can give you, in plain words:

- `200` with `meta.partial` set to `false`: all good. `meta.keywords` tells you how many videos each keyword returned.
- `200` with `meta.partial` and `meta.timedOut` both `true`: the search ran long, so the actor stopped at about three and a half minutes and sent back what it had. Use the videos, but ask for fewer keywords or a lower `limit` next time.
- `200` with `meta.chargeLimitReached` set to `true`: you hit the spending cap set on your Apify account, so you got part of the results. Raise the cap or wait for it to reset. This one doesn't come back as an error, so add a check for it in your workflow (an n8n IF node or a Make filter on `meta.chargeLimitReached`).
- `400` with an `error` message: something in the request is wrong, such as more than 10 keywords or broken JSON in a POST body. Read the message, fix the input and try again.
- `500` with "Failed to retrieve search list.": the search failed on the server side. Wait a bit and retry; don't loop instantly.

## Limits worth knowing

- Up to 10 keywords per request, and up to 500 videos per keyword. The actor searches up to four keywords at a time.
- The "unlimited" option from the normal run (`isUnlimited`) isn't available in server mode.
- Apify gives a server-mode request about five minutes to finish. The actor's own cut-off of about three and a half minutes is there to keep you under that.
- Results aren't stored. Save what you need in the next step.
- You pay per video returned (pay-per-event `result-item`), and only for videos you actually receive. `fetchUserBio` adds a separate charge per unique creator. Current prices are on the [Store page](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m).

If you need thousands of videos for a report, a long keyword list, or a dataset you can come back to later, use a normal scheduled run.

## Where to find the reference

The actor's Server tab in Apify Console lists the endpoints and parameters. The [Store page](https://apify.com/novi/advanced-search-tiktok-api?fpr=7hce1m) has pricing and the full input list for batch runs. For the bigger "do I need real-time at all?" question, see [yesterday's post](/apify/real-time-vs-scheduled-social-data/).

The next post in this series covers the [Ultimate TikTok Scraper](/apify/paste-tiktok-link-get-data-server-mode/), where you paste any TikTok link and get the data back.
