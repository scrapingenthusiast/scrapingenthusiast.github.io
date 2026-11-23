---
layout: post
title: "Paste a TikTok Link, Get the Numbers: Ultimate TikTok Scraper in Server Mode"
date: 2026-11-24 07:10:00 +0700
permalink: /apify/paste-tiktok-link-get-data-server-mode/
categories: [Social Media, Marketing Analytics]
tags: [TikTok, Apify, Server Actors, Google Sheets, Apps Script, n8n, Zapier, Make]
author: Scraping Enthusiast
description: "Turn any TikTok link (a video, profile, hashtag, sound or search) into views, likes and captions with one web request. A no-code walkthrough of Ultimate TikTok Scraper's server mode, plus a Google Sheets script."
---

# Paste a TikTok Link, Get the Numbers: Ultimate TikTok Scraper in Server Mode

Most marketing teams have a channel or a sheet where people drop TikTok links: "look at this creator", "this video is blowing up", "can we track this hashtag?" Someone then has to open each link and copy the views and likes by hand.

[Ultimate TikTok Scraper](https://apify.com/novi/tiktok-scraper-ultimate?fpr=7hce1m) now has a server mode that takes care of that copying. You send it one TikTok link of any kind and get the data straight back. This is the second post in a series on TikTok actors that gained server mode. The first covered [keyword and brand mention checks](/apify/tiktok-brand-mention-checks-server-mode/).

## What server mode is

Normally an Apify actor runs as a job: you start it, it collects data into a dataset, and you download the dataset when it's done. In server mode (Apify calls it Standby, or a "Server Actor") the actor waits at a fixed web address. You send it a request and the data comes back in the reply. You don't start a run or wait for one to finish, and there's no dataset to fetch. The trade-off is that nothing is stored for you, so whatever you want to keep has to be saved by the next step in your workflow.

One more thing to expect: if nobody has called the actor for a while, the first request takes longer because Apify has to start it up. After that it stays warm for about five minutes between calls.

## When a link lookup beats a scheduled run

Server mode suits one link at a time, when someone wants the answer now:

- A Slack shortcut: paste a link, get back views, likes, comments and shares.
- A Google Sheet where the team pastes links in column A and a button fills in the stats.
- A browser extension that shows a video's numbers while you're looking at it.
- An AI assistant that gets a "look up this TikTok link" tool.

A normal scheduled run is still the better choice for a list of 200 links, for more than 100 videos, or for a weekly report you want to keep as a dataset. Server mode handles one link per request and returns at most 100 videos. The batch run works the same way it always did.

## One address for every kind of link

The server lives here:

```
https://novi--tiktok-scraper-ultimate.apify.actor
```

There's only one endpoint, `/videos`, and you give it either a `url` or a `keyword`, but not both.

`url` takes any TikTok link the actor supports, and short links work too:

- a single video: `https://www.tiktok.com/@billieeilish/video/7050551461734042926`
- a profile: `https://www.tiktok.com/@gordonramsayofficial`
- a hashtag: `https://www.tiktok.com/tag/duet`
- a music (sound) page, a search page, or the home/trending feed

`keyword` searches TikTok instead, for example `podcast`.

Optional extras:

| Parameter | What it does | Default |
|---|---|---|
| `limit` | How many videos to return, 1 to 100 | 20 |
| `location` | Two-letter country code (IR, AQ, KP and TD aren't supported) | US |
| `sortType` | For searches and hashtags: `DEFAULT`, `RELEVANCE`, `MOST_LIKED`, `MOST_RECENT` | none |
| `dateRange` | For searches and hashtags: `DEFAULT`, `ALL_TIME`, `YESTERDAY`, `WEEK`, `MONTH`, `THREE_MONTH`, `SIX_MONTH` | none |

Some ready-made examples:

```
/videos?url=https://www.tiktok.com/@gordonramsayofficial&limit=20
/videos?url=https://www.tiktok.com/tag/duet&sortType=MOST_LIKED
/videos?url=https://www.tiktok.com/@billieeilish/video/7050551461734042926
/videos?keyword=podcast&location=US&dateRange=WEEK
```

The reply is a list of video objects, the same ones you'd see in a dataset from a normal run. For a single-video link the list holds that one video.

## Your token stays in the credential store

Requests need your Apify API token in a header:

```
Authorization: Bearer <your token>
```

Apify also accepts `?token=` in the URL. Skip that option: these links tend to get pasted into team docs, and the token would go along with them. Save the token as a credential in n8n, Make or Zapier, as a Script Property in Google Apps Script, or as the `APIFY_TOKEN` environment variable in code.

## The no-code version

**n8n**, HTTP Request node:

1. Method `GET`, URL `https://novi--tiktok-scraper-ultimate.apify.actor/videos`
2. Authentication: Header Auth credential, name `Authorization`, value `Bearer <token>`.
3. Query parameters: `url` mapped from your trigger (the Slack message text, the sheet cell), and `limit` if it's a profile or hashtag.
4. Options: timeout of 300000 ms.

Add query parameters as separate fields rather than gluing the link onto the URL yourself. The tool then encodes the TikTok link correctly, and links containing `?` or `&` won't break.

**Make**: HTTP "Make a request" module with the same method and URL, an `Authorization` header from a stored connection, `url` in the query string, and a longer timeout.

**Zapier**: Webhooks by Zapier, GET, with the same URL, query parameter and header. Use small `limit` values, since Zapier steps have their own time limits.

Fields worth mapping into your sheet or message: `desc` (caption), `author.unique_id` (handle), `statistics.play_count`, `statistics.digg_count` (likes), `statistics.comment_count`, `statistics.share_count`, and `create_time`.

## A Google Sheets button that fills in the stats

This is the setup I'd give a social team. People paste links into column A, someone clicks a menu item, and columns B to F fill in. In the sheet, open Extensions > Apps Script, paste the code below, then go to Project Settings > Script Properties and add `APIFY_TOKEN`. The token stays out of the sheet itself.

```javascript
function onOpen() {
  SpreadsheetApp.getUi().createMenu('TikTok')
    .addItem('Fill stats for links', 'fillStats').addToUi();
}

function fillStats() {
  const token = PropertiesService.getScriptProperties().getProperty('APIFY_TOKEN');
  const sheet = SpreadsheetApp.getActiveSheet();
  if (sheet.getLastRow() < 2) return;
  const rows = sheet.getRange(2, 1, sheet.getLastRow() - 1, 1).getValues();

  rows.forEach(([link], i) => {
    if (!link) return;
    const res = UrlFetchApp.fetch(
      'https://novi--tiktok-scraper-ultimate.apify.actor/videos?limit=1&url=' + encodeURIComponent(link),
      { headers: { Authorization: 'Bearer ' + token }, muteHttpExceptions: true }
    );
    const code = res.getResponseCode();
    const out = sheet.getRange(i + 2, 2, 1, 5);
    if (code !== 200) {
      out.setValues([['Error ' + code, '', '', '', '']]);
      return;
    }
    const v = JSON.parse(res.getContentText())[0] || {};
    const s = v.statistics || {};
    out.setValues([[(v.author || {}).unique_id || '', s.play_count || '', s.digg_count || '',
                     s.comment_count || '', (v.desc || '').slice(0, 100)]]);
  });
}
```

It asks for `limit=1`, so a profile or hashtag link gives you its first video and a video link gives you that video.

For terminal users, the same call in curl (`--data-urlencode` takes care of encoding the link):

```bash
curl -s -G -H "Authorization: Bearer $APIFY_TOKEN" \
  "https://novi--tiktok-scraper-ultimate.apify.actor/videos" \
  --data-urlencode "url=https://www.tiktok.com/@gordonramsayofficial" \
  --data-urlencode "limit=20"
```

And in Node.js 18 or later:

```javascript
const params = new URLSearchParams({ url: 'https://www.tiktok.com/tag/duet', sortType: 'MOST_LIKED', limit: '20' });
const res = await fetch(`https://novi--tiktok-scraper-ultimate.apify.actor/videos?${params}`, {
  headers: { Authorization: `Bearer ${process.env.APIFY_TOKEN}` },
  signal: AbortSignal.timeout(300_000),
});
if (res.status === 402) console.log('Spending limit reached on your Apify account');
else if (!res.ok) console.log(res.status, await res.text());
else console.log((await res.json()).map(v => [v.author?.unique_id, v.statistics?.play_count]));
```

## What the error codes mean

- `200`: it worked. You get a list of videos.
- `400`: the request isn't right. Common causes: you sent both `url` and `keyword`, sent neither, or pasted a link the actor can't handle (the message starts with "Unsupported TikTok URL"). Check what was actually pasted. Links from other sites or with stray text around them are the usual problem.
- `402`: your Apify account hit its spending limit. Nothing more will come back until you raise it or it resets. Have your workflow send someone a message about it; retrying won't help.
- `404`: the path is wrong. It's `/videos`, so check for a typo.
- `502` with "Upstream request failed": fetching from TikTok failed this time. Wait a little and try again.

## Limits and cost

- One link or one keyword per request, and 1 to 100 videos.
- `customMapFunction`, several links at once, and more than 100 videos are only available in the normal run.
- Apify gives a server-mode request about five minutes to finish, so set your tool's timeout to match.
- Results aren't stored on Apify.
- Billing is per video returned (`result-item`), plus the same per-task events a normal run uses. A single-video lookup, for example, also counts as a `single-video-query`. Current rates are on the [Store page](https://apify.com/novi/tiktok-scraper-ultimate?fpr=7hce1m).

## Further reading

The endpoint reference is on the actor's Server tab in Apify Console. If you're still deciding whether you need instant lookups at all, start with [real-time vs scheduled social data](/apify/real-time-vs-scheduled-social-data/). The last post in this series looks at [vetting creators with TikTok Profile Videos API](/apify/tiktok-creator-vetting-server-mode/).
