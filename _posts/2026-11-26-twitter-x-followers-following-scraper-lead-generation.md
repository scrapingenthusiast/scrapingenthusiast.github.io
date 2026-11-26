---
layout: post
title: "Turn Any X (Twitter) Audience into a Lead List: Followers & Following Scraper with Email Filters"
date: 2026-11-26 07:30:00 +0700
permalink: /apify/twitter-x-followers-following-scraper-lead-generation/
categories: [Social Media, Lead Generation]
tags: [Twitter, X.com, Apify, Followers, Lead Generation, Audience Research, Influencer Marketing]
author: Scraping Enthusiast
description: "A no-code guide to exporting the followers and following lists of any public X (Twitter) account, filtering them by follower count, verified status, email or website, and paying only for the profiles you keep."
---

# Turn Any X (Twitter) Audience into a Lead List: Followers & Following Scraper with Email Filters

The followers of an account tell you a lot. A competitor's followers are people who already care about your market. A conference account's followers are the attendees and speakers. The accounts a well-known investor follows are a short list of companies and people they rate.

X shows these lists on every public profile, but scrolling through them by hand stops being useful after the first fifty names. [Twitter X.com Followers & Following Scraper](https://apify.com/xtdata/twitter-x-followers-following-scraper?fpr=7hce1m) exports them as a table instead: full profiles, follower counts, websites, and any email addresses written in the bio.

This post covers what you get, how to set up a run for lead generation or audience research, and how the pricing works when you use filters.

## What you give it and what comes back

You list the accounts you care about. Any mix of these works:

| Input | Example |
|---|---|
| Username | `nasa` or `@nasa` |
| Numeric user ID | `44196397` |
| Profile URL | `https://x.com/apify` or `https://x.com/apify/followers` |

Then you pick the lists: **followers** (on by default), **following**, or both. `maxItems` sets how many users to deliver per account and per list. Ask for 500 followers of three accounts and you get up to 1,500 rows.

Each row is a full profile:

- `username`, `name`, `description` (the bio), `location`, `createdAt`
- `followersCount`, `followingCount`, `tweetCount`, `likeCount`, `listedCount`, `mediaCount`
- `verified` and `isBlueVerified`
- `website`, the expanded link from the profile instead of a `t.co` short link
- `emails`, a list of addresses found in the bio
- `type` (`follower` or `following`) and `sourceUser`, the account whose list the row came from

The last two matter when you scrape several accounts at once. You can filter the spreadsheet by `sourceUser` later and still know which competitor each lead came from.

The scraper doesn't log in to X. It reads what any visitor can see, so protected accounts return nothing (and cost nothing).

## Filters run before you pay

Most of a big account's followers are of no use to you. Plenty are inactive, have no bio, or have nine followers of their own. You can tell the actor to skip those before they reach your dataset:

| Filter | Keeps only users who... |
|---|---|
| `minFollowers` / `maxFollowers` | have a follower count in that range |
| `onlyVerified` | are verified or have a blue check |
| `onlyWithEmail` | have an email address in their bio |
| `onlyWithWebsite` | have a website in their profile |

Users that don't match are neither delivered nor charged. That changes the math for lead generation. You aren't paying for 10,000 followers so you can keep 300 of them. You pay for the 300.

The actor checks a limited number of profiles per list when filters are on, so a run can't keep scanning forever looking for matches. The default is `maxItems` × 5, and `maxScanned` changes it. If you ask for 100 users with an email and only get 40, the first 500 profiles didn't contain more. Raise `maxScanned` or loosen a filter.

## Three setups worth copying

**1. Competitor audience with contact details.** Followers of two competitors, keeping people who have at least 1,000 followers and an email in their bio:

```json
{
    "usernames": ["competitor_one", "competitor_two"],
    "minFollowers": 1000,
    "onlyWithEmail": true,
    "maxItems": 200,
    "maxScanned": 5000
}
```

This makes a short outreach list of people with some reach who put their contact address in public themselves.

**2. Micro-influencers inside a niche.** Take the followers of a community account in your niche and keep the 5,000 to 50,000 follower range, where creators are big enough to matter and small enough to answer DMs:

```json
{
    "startUrls": ["https://x.com/some_niche_community"],
    "minFollowers": 5000,
    "maxFollowers": 50000,
    "onlyWithWebsite": true,
    "maxItems": 100
}
```

**3. Who does this person follow?** The following list of a founder, investor or journalist is often more revealing than their followers, because they picked every account on it:

```json
{
    "usernames": ["some_investor"],
    "getFollowers": false,
    "getFollowing": true,
    "maxItems": 500
}
```

Turn on both lists with `getFollowers` and `getFollowing` if you want to map a small network. Every row says which list it came from.

## What it costs

Pricing is per delivered profile, with no start fee:

| Apify plan | Per follower | Per following |
|---|---|---|
| Free | $0.002 | $0.004 |
| Bronze / Silver | $0.0005 | $0.001 |
| Gold and up | $0.0004 | $0.0009 |

On a paid plan, 1,000 followers cost $0.50. Filtered-out users are free, and so are protected accounts and failed requests, so a strict filter only costs you the rows you keep. You can also set a maximum cost on the run and the actor stops when it reaches it.

## After the export

Results download as CSV, Excel or JSON, or can be read through the Apify API. A few things people usually do next:

- Sort by `followersCount` and work down the list.
- Drop the `emails` column into your outreach tool, after checking each address was meant to be public.
- Remove duplicates across `sourceUser` values: people who follow *both* of your competitors are usually the warmest leads.
- Enrich the strongest profiles with their recent posts using [X.com Twitter User Tweets Scraper](https://apify.com/xtdata/twitter-x-user-tweets-scraper?fpr=7hce1m), so your first message refers to something they actually wrote.

If you start from a topic instead of an account, find the accounts first with [X.com Twitter User Search Scraper](https://apify.com/xtdata/twitter-x-user-search-scraper?fpr=7hce1m), or pull conversations by keyword with [Twitter - X.com Scraper](https://apify.com/xtdata/twitter-x-scraper?fpr=7hce1m), then feed the handles you like into the followers scraper.

## A note on using this data responsibly

Everything here is public profile data, but public doesn't mean anyone agreed to bulk marketing. Keep outreach personal and relevant, honor opt-outs right away, and check the rules that apply to you (GDPR in the EU, CAN-SPAM in the US) before you email anyone from a scraped list.

## Try it

Open [Twitter X.com Followers & Following Scraper](https://apify.com/xtdata/twitter-x-followers-following-scraper?fpr=7hce1m), add one account you already know well, and run it with `maxItems` set to 50. Seeing who shows up in a familiar audience is the quickest way to decide which filters you need for the real run.
