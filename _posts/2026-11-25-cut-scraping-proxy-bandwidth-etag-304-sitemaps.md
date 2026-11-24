---
layout: post
title: "Cut Scraping Proxy Bills with Conditional HTTP (ETag 304), Sitemaps, and Asset Blocking"
date: 2026-11-25 00:10:00 +0700
permalink: /web-scraping/cut-scraping-proxy-bandwidth-etag-304-sitemaps/
categories: [Web Scraping, Cost Optimization, Automation]
tags: [Web Scraping, ETag, HTTP 304, Sitemaps, Playwright, Proxies, Python, Apify]
author: Scraping Enthusiast
description: "Stop paying residential proxy bandwidth to re-download unchanged pages. Combine sitemap lastmod checks, RFC 9110 ETag/304 conditional requests, gzip, and Playwright route blocking, with real byte math and a Python SQLite tracker."
---

Residential proxies are usually billed by the gigabyte, somewhere between $2/GB on volume plans and $8/GB pay-as-you-go at major providers. If your job polls 5,000 competitor pages every hour in a headless browser, downloading 2 MB of HTML, images, fonts, and scripts per page adds up to 10 GB an hour, or 7.2 TB over a 30-day month. Even at $2/GB, re-fetching pages that did not change since the last run can cost thousands of dollars a month.

Most recurring monitors do not need that traffic. By stacking three standard HTTP and browser controls (a `sitemap.xml` date check, `ETag` / `Last-Modified` conditional requests with `gzip`, and asset blocking when a browser is unavoidable), you only pay bandwidth for bytes that actually changed.

## How much traffic each layer removes

Take the same 5,000 pages polled hourly (3.6 million page checks per month), and assume only 10% of URLs change on any given run while the rest stay static:

- Full Playwright page load without blocking (2 MB average): 7,200 GB/month ($14,400 at $2/GB).
- Raw HTTP `GET` without compression on HTML only (200 KB average): 720 GB/month ($1,440 at $2/GB).
- HTTP `GET` with `Accept-Encoding: gzip` (around 30 KB average, 85% smaller): 108 GB/month ($216 at $2/GB).
- Conditional `GET` returning `304 Not Modified` on the 90% unchanged pages (0 body bytes) plus gzipped `200` on the 10% changed pages: roughly 10.8 GB/month of body traffic ($21.60 at $2/GB), before headers.

Those are back-of-the-envelope numbers, because real sites differ in compression ratios and header sizes. The pattern holds whenever most pages sit still between polls.

## Layer 1: pre-filter URLs with sitemap.xml

Before opening connections to 5,000 product URLs, fetch the site's `sitemap.xml`. According to the [Sitemaps XML protocol](https://www.sitemaps.org/protocol.html), each `<url>` block can include a `<lastmod>` timestamp showing when that page was last updated. When `<lastmod>` matches the timestamp stored in your database from the last run, skip the page request completely.

Two caveats keep this realistic. First, many CMSs and e-commerce plugins stamp every URL in `sitemap.xml` with today's build timestamp, or forget to update `<lastmod>` when only a price changes. Second, a sitemap fetch itself should use conditional headers (`If-None-Match`), because a `304` on the sitemap tells you nothing inside it moved. Treat `<lastmod>` as a first-pass filter for sites you verified, and verify one or two changed pages before trusting it blindly.

## Layer 2: conditional requests (ETag 304 and gzip)

When you do fetch a page, ask for compression and save its cache validators. Under [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html) and [MDN's conditional request guide](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Conditional_requests), servers return `ETag` and `Last-Modified` headers on a `200 OK`. On your next poll, send those saved values back in `If-None-Match` and `If-Modified-Since`. If the page has not changed, the server replies `304 Not Modified` with an empty body.

We tested four live public endpoints with Python `urllib` to see what happens in practice across real servers and CDNs:

| Target URL | Raw bytes | Gzip wire bytes | Savings | `ETag` (gzip response) | Conditional `304` bytes |
|---|---|---|---|---|---|
| `https://docs.python.org/3/library/urllib.request.html` | 226,580 | 30,706 | 86.4% | `"6b05c40c-37514"` | 0 |
| `https://www.sitemaps.org/protocol.html` | 45,292 | 8,923 | 80.3% | `"041cb8a7310d91:0"` | 0 |
| `https://novidevelop.github.io/sitemap.xml` | 8,783 | 1,562 | 82.2% | `W/"6b059192-224f"` | 0 |
| `https://en.wikipedia.org/api/rest_v1/page/summary/Web_scraping` | 2,308 | 779 | 66.2% | `W/"1385963349/3158d8f0-e6c8-11f1-b7e7-f2d05fa71244"` | 0 |

Enabling `Accept-Encoding: gzip` cut HTML and XML wire sizes by 80.3% to 86.4% before conditional caching even kicked in, and every target returned `304` with `0` body bytes on `If-None-Match`.

That probe also caught two CDN behaviors that break naive `ETag` scrapers:

- `novidevelop.github.io/sitemap.xml` returned a strong ETag (`"6b059192-224f"`) when fetched uncompressed, yet returned a weak ETag (`W/"6b059192-224f"`) when fetched with `Accept-Encoding: gzip`.
- `sitemaps.org/protocol.html` returned two completely different ETag strings across encodings (`"6b6deb8a7310d91:0"` uncompressed vs `"041cb8a7310d91:0"` gzipped).

As documented in [Cloudflare's ETag headers reference](https://developers.cloudflare.com/cache/reference/etag-headers/), CDNs routinely convert strong origin ETags to `W/` weak ETags when applying compression at the edge, and features such as Email Obfuscation or Automatic HTTPS Rewrites can alter response bytes or strip ETags altogether. Two simple rules prevent cache misses on your scraper: always use the same `Accept-Encoding` header across runs, and store the exact `ETag` string verbatim (never strip the `W/` prefix). In our local test server simulating weak CDN ETags, sending `W/"weak-v1"` produced `304` (`0` bytes), while stripping `W/` forced a full `200 OK` re-download.

## Layer 3: blocking Playwright assets when a browser is required

Some targets render prices only after client-side JavaScript runs. If you have to launch Playwright over a residential proxy, use [`page.route()`](https://playwright.dev/python/docs/network) to abort images, fonts, media, stylesheets, and third-party analytics trackers before they cross the proxy:

```python
BLOCKED_TYPES = {"image", "media", "font", "stylesheet"}
BLOCKED_HOSTS = ("googletagmanager.com", "google-analytics.com", "doubleclick.net")


def route_filter(route):
    req = route.request
    if req.resource_type in BLOCKED_TYPES or any(h in req.url for h in BLOCKED_HOSTS):
        route.abort()
    else:
        route.continue_()


# Attach before page.goto():
# page.route("**/*", route_filter)
```

One caution with `stylesheet` blocking: a few anti-bot scripts check computed element bounding boxes or layout visibility, and some scrapers rely on CSS layout heuristics. Test each site with `stylesheet` blocked first; if a target breaks or triggers extra challenges, drop `"stylesheet"` from `BLOCKED_TYPES` while keeping `image`, `media`, and `font` blocked.

## Real-world project: a low-cost catalog and policy tracker

Here is a self-contained Python and SQLite tracker that chains `sitemap.xml` checks, strong and weak `ETag` / `Last-Modified` validation, `gzip` decompression, and a normalized-text SHA-256 fallback for dynamic pages that omit ETags and mutate HTML comments on every render.

```python
from datetime import datetime, timezone
import gzip
import hashlib
from html.parser import HTMLParser
import re
import sqlite3
import urllib.error
import urllib.request
import xml.etree.ElementTree as ET


class VisibleTextParser(HTMLParser):
    def __init__(self):
        super().__init__()
        self._skip = False
        self._parts = []

    def handle_starttag(self, tag, attrs):
        if tag.lower() in ("script", "style", "noscript"):
            self._skip = True

    def handle_endtag(self, tag):
        if tag.lower() in ("script", "style", "noscript"):
            self._skip = False

    def handle_data(self, data):
        if not self._skip:
            txt = data.strip()
            if txt:
                self._parts.append(txt)

    def digest(self) -> tuple[str, str]:
        norm = re.sub(r"\s+", " ", " ".join(self._parts)).strip()
        return hashlib.sha256(norm.encode("utf-8")).hexdigest(), norm


def http_get_conditional(url: str, etag=None, last_modified=None) -> dict:
    headers = {
        "User-Agent": "CatalogDeltaTracker/1.0",
        "Accept-Encoding": "gzip",
    }
    if etag:
        headers["If-None-Match"] = etag
    if last_modified:
        headers["If-Modified-Since"] = last_modified

    req = urllib.request.Request(url, headers=headers)
    try:
        with urllib.request.urlopen(req, timeout=20) as resp:
            wire = resp.read()
            enc = (resp.headers.get("Content-Encoding") or "").lower()
            raw = gzip.decompress(wire) if enc == "gzip" else wire
            return {
                "status": resp.status,
                "wire_bytes": len(wire),
                "etag": resp.headers.get("ETag"),
                "last_modified": resp.headers.get("Last-Modified"),
                "body": raw,
            }
    except urllib.error.HTTPError as err:
        if err.code == 304:
            return {
                "status": 304,
                "wire_bytes": 0,
                "etag": err.headers.get("ETag") or etag,
                "last_modified": err.headers.get("Last-Modified") or last_modified,
                "body": b"",
            }
        raise


def poll_sitemap_tracker(db_path: str, sitemap_url: str) -> dict:
    conn = sqlite3.connect(db_path)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS crawl_state (
            url TEXT PRIMARY KEY,
            sitemap_lastmod TEXT,
            etag TEXT,
            last_modified TEXT,
            content_sha256 TEXT,
            last_status INTEGER,
            updated_at TEXT
        )
    """)

    sm = http_get_conditional(sitemap_url)
    root = ET.fromstring(sm["body"])
    ns = {"sm": "http://www.sitemaps.org/schemas/sitemap/0.9"}

    report = {
        "wire_bytes": sm["wire_bytes"],
        "skipped_by_sitemap": 0,
        "http_304": 0,
        "http_200_unchanged_text": 0,
        "changed_urls": [],
    }

    for url_el in root.findall("sm:url", ns):
        loc = url_el.findtext("sm:loc", default="", namespaces=ns)
        lastmod_el = url_el.find("sm:lastmod", ns)
        sm_lastmod = lastmod_el.text if lastmod_el is not None else None

        prev = conn.execute(
            "SELECT sitemap_lastmod, etag, last_modified, content_sha256 FROM crawl_state WHERE url=?",
            (loc,),
        ).fetchone()

        if prev and sm_lastmod and prev[0] == sm_lastmod:
            report["skipped_by_sitemap"] += 1
            continue

        res = http_get_conditional(
            loc,
            etag=prev[1] if prev else None,
            last_modified=prev[2] if prev else None,
        )
        report["wire_bytes"] += res["wire_bytes"]

        if res["status"] == 304:
            report["http_304"] += 1
            conn.execute(
                "UPDATE crawl_state SET sitemap_lastmod=?, last_status=304, updated_at=? WHERE url=?",
                (sm_lastmod, datetime.now(timezone.utc).isoformat(), loc),
            )
            continue

        parser = VisibleTextParser()
        parser.feed(res["body"].decode("utf-8", errors="replace"))
        digest, norm_text = parser.digest()

        if prev and prev[3] == digest:
            report["http_200_unchanged_text"] += 1
        else:
            report["changed_urls"].append({"url": loc, "summary": norm_text[:160]})

        conn.execute(
            "INSERT OR REPLACE INTO crawl_state VALUES (?, ?, ?, ?, ?, ?, ?)",
            (
                loc,
                sm_lastmod,
                res["etag"],
                res["last_modified"],
                digest,
                res["status"],
                datetime.now(timezone.utc).isoformat(),
            ),
        )

    conn.commit()
    conn.close()
    return report
```

We ran this tracker across three cycles against a local four-page test server covering a strong-ETag page, a weak `W/` ETag page, a dynamic no-ETag page with changing HTML comments, and a static archive page:

- Cycle 1 (cold start): downloaded all four pages (`861` wire bytes total) and recorded four initial snapshots.
- Cycle 2 (CMS falsely bumped `<lastmod>` on three unchanged pages): skipped one URL via sitemap, received two `304` replies (`0` body bytes each), caught the dynamic no-ETag render comment change with the SHA-256 text check, transferred `383` bytes total, and fired `0` false alerts.
- Cycle 3 (two prices dropped, one strong-ETag page and one dynamic page): transferred `543` bytes, skipped one URL via sitemap, got one `304`, and reported only the two genuine price drops.

Drop the `changed_urls` list into an n8n HTTP webhook or Slack alert so downstream automations only run when visible text moves.

## Knowing when conditional HTTP stops applying

Conditional HTTP works well for competitor blogs, documentation sites, SaaS pricing pages, and standard storefronts. Social platforms such as TikTok and X do not expose public `sitemap.xml` change logs or stable `ETag` headers across feeds, and running headless browsers over metered residential proxies to watch social hashtags wastes money quickly.

For teams pairing site change tracking with social monitoring, keep the conditional HTTP loop above on web pages and send the social side to pay-per-event APIs where billing tracks returned items instead of rendered page megabytes: [Fast TikTok API](https://apify.com/novi/fast-tiktok-api?fpr=7hce1m) for user profiles and videos, [TikTok Trend API](https://apify.com/novi/tiktok-trend-api?fpr=7hce1m) for regional trending feeds, and [Twitter - X.com Scraper](https://apify.com/xtdata/twitter-x-scraper?fpr=7hce1m) for keyword and handle lookups. That keeps metered proxy usage focused on pages that actually support HTTP caching.
