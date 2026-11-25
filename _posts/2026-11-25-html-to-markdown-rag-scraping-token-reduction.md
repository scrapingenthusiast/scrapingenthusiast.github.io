---
layout: post
title: "HTML to Clean Markdown for LLM and RAG Scrapers: Cutting Token Costs and Preserving Structure"
date: 2026-11-25 08:20:00 +0700
permalink: /web-scraping/html-to-markdown-rag-scraping-token-reduction/
categories: [Web Scraping, Cost Optimization, AI Pipelines]
tags: [Web Scraping, RAG, Markdown, LLM, HTMLParser, Python, Chunking, Apify]
author: Scraping Enthusiast
description: "Stop feeding raw HTML into LLM prompts and vector databases. Benchmark token savings across four live sites, strip boilerplate with Python's html.parser, and chunk Markdown on heading boundaries with breadcrumb metadata."
---

Sending raw HTML straight into an embedding model or LLM context window burns tokens on markup your prompt never uses. A single marketing or documentation page often carries hundreds of kilobytes of inline CSS, SVG icons, navigation drawers, cookie banners, and hydration JSON around a few paragraphs of real prose.

When you embed that noise alongside the article, two things break at once. First, token bills jump by 4x to 60x depending on how heavy the site's frontend framework is. Second, repeated sidebar menus and footer disclaimers pollute vector similarity search so unrelated pages cluster together.

Converting scraped HTML into stripped, main-content Markdown before chunking fixes both issues, as long as you decide up front whether to keep outbound link URLs.

## Live token benchmark across four public pages

We fetched four real pages with plain HTTP (`Accept-Encoding: gzip`) and compared three formats: raw HTML, naive tag stripping (removing `<script>`, `<style>`, and HTML tags while leaving navbars and sidebars in), and main-content Markdown built with Python's standard library [`html.parser`](https://docs.python.org/3/library/html.parser.html). Token estimates below use the standard 4-characters-per-token rule of thumb for English and code.

| Target URL | Raw HTML bytes (est. tokens) | Naive text strip (est. tokens) | Clean Markdown (est. tokens) | Savings vs HTML | Parse time |
|---|---|---|---|---|---|
| `https://docs.python.org/3/library/html.parser.html` | 55,264 (~13,800) | 12,084 (~3,006) | 10,913 (~2,720) | 80.3% | 24.5 ms |
| `https://www.rfc-editor.org/rfc/rfc9309.html` | 82,367 (~20,562) | 21,465 (~5,350) | 23,400 (~5,834) | 71.6% | 28.0 ms |
| `https://en.wikipedia.org/wiki/Web_scraping` | 244,743 (~61,060) | 31,486 (~7,831) | 43,594 (~10,874) | 82.2% | 43.0 ms |
| `https://nextjs.org/blog` | 1,071,304 (~267,804) | 16,292 (~4,068) | 15,801 (~3,947) | 98.5% | 50.6 ms |

The raw-HTML-to-Markdown drop runs from 71.6% on a sparse RFC specification up to 98.5% on a Next.js App Router blog index (`267,804` estimated tokens down to `3,947`).

On the Wikipedia row, clean Markdown (`43,594` bytes) came out larger than naive plain-text stripping (`31,486` bytes). That happened because our Markdown converter preserved `[label](https://...)` URLs on every outbound link, and [Wikipedia's Web scraping article](https://en.wikipedia.org/wiki/Web_scraping) carries hundreds of external reference URLs. If your RAG pipeline needs clickable citations, keep the URLs. If you only need semantic retrieval over prose, dropping link destinations (`label` instead of `[label](url)`) saves roughly 12 KB of URL tokens on citation-heavy pages.

## Two HTML parsing bugs that silently empty your output

- Void elements such as `<meta>`, `<link>`, `<input>`, `<img>`, and `<br>` never fire `handle_endtag()`. If your parser increments a nesting depth counter on every start tag and decrements on end tags, a single `<img>` inside a skipped `<nav>` block leaves `_skip_depth` stuck above zero and silently drops the rest of the page. Always check tags against the HTML5 void element set before touching depth counters.
- Substring matching on class names causes false positives. Checking `if "header" in class_attr` skips legitimate containers on documentation sites whose wrapper classes include names like `section-header` or `mw-page-header`. Match exact class tokens (`set(class_attr.split())`) against a known boilerplate list instead of substring checks.

## Standard-library HTML-to-Markdown extractor and heading chunker

Production teams with diverse news and blog sources often use dedicated heuristic libraries like [Trafilatura](https://trafilatura.readthedocs.io/en/latest/) or [Mozilla Readability](https://github.com/mozilla/readability), which score DOM subtrees by link density and paragraph length. When you scrape a known set of documentation portals or storefronts and want zero external dependencies, Python's `html.parser` handles extraction and heading-aware chunking in under 55 ms per page:

```python
from html.parser import HTMLParser
import re

FENCE = "`" * 3

VOID_TAGS = {
    "area", "base", "br", "col", "embed", "hr", "img",
    "input", "link", "meta", "param", "source", "track", "wbr",
}

SKIP_TAGS = {
    "script", "style", "noscript", "svg", "nav", "header",
    "footer", "aside", "form", "button", "iframe", "template",
}

BOILERPLATE_CLASSES = {
    "sphinxsidebar", "related", "navbox", "catlinks", "printfooter",
    "mw-jump-link", "mw-editsection", "vector-header-container",
    "vector-column-start", "vector-column-end", "vector-page-toolbar",
    "vector-body-before-content", "noprint", "toc", "breadcrumb",
    "cookie-banner", "site-footer", "site-header",
}


class MainContentMarkdownParser(HTMLParser):
    def __init__(self, prefer_main: bool = True, keep_links: bool = True):
        super().__init__()
        self.prefer_main = prefer_main
        self.keep_links = keep_links
        self._in_main_depth = 0
        self._skip_depth = 0
        self._in_pre = False
        self._in_code = False
        self._href_stack = []
        self._link_buf = None
        self._chunks = []

    def _is_boilerplate(self, tag: str, attrs_map: dict) -> bool:
        if tag in SKIP_TAGS:
            return True
        role = attrs_map.get("role", "").lower()
        if role in ("navigation", "banner", "contentinfo", "complementary", "search"):
            return True
        classes = set(attrs_map.get("class", "").lower().split())
        elem_id = attrs_map.get("id", "").lower()
        return bool(classes & BOILERPLATE_CLASSES) or elem_id in BOILERPLATE_CLASSES

    def _active(self) -> bool:
        if self._skip_depth > 0:
            return False
        if self.prefer_main and self._in_main_depth == 0:
            return False
        return True

    def handle_starttag(self, tag, attrs):
        tag = tag.lower()
        is_void = tag in VOID_TAGS
        attrs_map = {k.lower(): (v or "") for k, v in attrs}

        if self._skip_depth > 0:
            if not is_void:
                self._skip_depth += 1
            return

        is_main_root = (
            tag in ("main", "article")
            or attrs_map.get("role", "").lower() == "main"
            or attrs_map.get("id", "").lower() == "mw-content-text"
        )

        if self._in_main_depth == 0:
            if is_main_root:
                if not is_void:
                    self._in_main_depth = 1
            elif self._is_boilerplate(tag, attrs_map):
                if not is_void:
                    self._skip_depth = 1
                return
        else:
            if self._is_boilerplate(tag, attrs_map):
                if not is_void:
                    self._skip_depth = 1
                return
            if not is_void:
                self._in_main_depth += 1

        if not self._active():
            return

        if tag in ("h1", "h2", "h3", "h4", "h5", "h6"):
            self._chunks.append(f"\n\n{'#' * int(tag[1])} ")
        elif tag == "p":
            self._chunks.append("\n\n")
        elif tag == "br":
            self._chunks.append("\n")
        elif tag == "li":
            self._chunks.append("\n- ")
        elif tag == "blockquote":
            self._chunks.append("\n\n> ")
        elif tag == "pre":
            self._in_pre = True
            self._chunks.append(f"\n\n{FENCE}\n")
        elif tag == "code" and not self._in_pre:
            self._in_code = True
            self._chunks.append("`")
        elif tag == "tr":
            self._chunks.append("\n| ")
        elif tag == "a":
            self._href_stack.append(attrs_map.get("href", "").strip())
            self._link_buf = []

    def handle_endtag(self, tag):
        tag = tag.lower()
        if tag in VOID_TAGS:
            return
        if self._skip_depth > 0:
            self._skip_depth -= 1
            return

        active = self._active()
        if self._in_main_depth > 0:
            self._in_main_depth -= 1
        if not active:
            return

        if tag == "pre":
            self._in_pre = False
            self._chunks.append(f"\n{FENCE}\n\n")
        elif tag == "code" and not self._in_pre and self._in_code:
            self._in_code = False
            self._chunks.append("`")
        elif tag in ("td", "th"):
            self._chunks.append(" | ")
        elif tag == "a" and self._href_stack:
            href = self._href_stack.pop()
            label = re.sub(r"\s+", " ", "".join(self._link_buf or [])).strip()
            self._link_buf = None
            if label:
                if self.keep_links and href.startswith(("http://", "https://")):
                    self._chunks.append(f"[{label}]({href})")
                else:
                    self._chunks.append(label)

    def handle_data(self, data):
        if not self._active():
            return
        if self._in_pre:
            self._chunks.append(data)
            return
        text = re.sub(r"\s+", " ", data)
        if not text.strip():
            if text and self._chunks and not self._chunks[-1].endswith((" ", "\n")):
                self._chunks.append(" ")
            return
        if self._link_buf is not None:
            self._link_buf.append(text)
        else:
            self._chunks.append(text)

    def markdown(self) -> str:
        lines = [line.rstrip() for line in "".join(self._chunks).splitlines()]
        return re.sub(r"\n{3,}", "\n\n", "\n".join(lines)).strip()


def html_to_clean_markdown(html_text: str, keep_links: bool = True) -> str:
    parser = MainContentMarkdownParser(prefer_main=True, keep_links=keep_links)
    parser.feed(html_text)
    md = parser.markdown()
    if len(md) < 200:
        fallback = MainContentMarkdownParser(prefer_main=False, keep_links=keep_links)
        fallback.feed(html_text)
        md = fallback.markdown()
    return md


def chunk_markdown_by_headings(md: str, max_chars: int = 1600) -> list:
    heading_stack = []
    sections = []
    cur_lines = []
    in_fence = False

    for line in md.splitlines():
        if line.strip().startswith(FENCE):
            in_fence = not in_fence
            cur_lines.append(line)
            continue

        match = re.match(r"^(#{1,6})\s+(.+)$", line) if not in_fence else None
        if match:
            if cur_lines:
                sections.append((" > ".join(t for _, t in heading_stack), "\n".join(cur_lines).strip()))
                cur_lines = []
            level = len(match.group(1))
            title = match.group(2).strip()
            heading_stack = [(lvl, t) for lvl, t in heading_stack if lvl < level]
            heading_stack.append((level, title))
            cur_lines.append(line)
        else:
            cur_lines.append(line)

    if cur_lines:
        sections.append((" > ".join(t for _, t in heading_stack), "\n".join(cur_lines).strip()))

    chunks = []
    for breadcrumb, body in sections:
        if not body:
            continue
        if len(body) <= max_chars:
            chunks.append({"breadcrumb": breadcrumb, "text": body, "chars": len(body)})
            continue

        blocks = re.split(r"\n\n+", body)
        buf = []
        buf_len = 0
        fence_open = False
        for blk in blocks:
            blk_fences = blk.count(FENCE)
            if buf and not fence_open and buf_len + len(blk) + 2 > max_chars:
                text = "\n\n".join(buf)
                chunks.append({"breadcrumb": breadcrumb, "text": text, "chars": len(text)})
                buf = [blk]
                buf_len = len(blk)
            else:
                buf.append(blk)
                buf_len += len(blk) + (2 if len(buf) > 1 else 0)
            if blk_fences % 2 == 1:
                fence_open = not fence_open
        if buf:
            text = "\n\n".join(buf)
            chunks.append({"breadcrumb": breadcrumb, "text": text, "chars": len(text)})

    return chunks
```

## Why heading breadcrumbs matter for RAG retrieval

Fixed-character splitters slice right through code blocks and strip away which section a paragraph came from. On our four test pages, `chunk_markdown_by_headings(..., max_chars=1600)` produced `9` chunks on Python's `html.parser` docs (average `1,207` chars), `38` chunks on RFC 9309 (average `612` chars), `32` chunks on Wikipedia's Web scraping page (average `1,357` chars), and `80` chunks on [Next.js Blog](https://nextjs.org/blog) (average `195` chars), with `0` split or unclosed code fences across all `159` chunks.

Prepending `breadcrumb` (such as `Robots Exclusion Protocol > Abstract` or `Techniques > Human copy-and-paste`) to each chunk before calling your embedding API gives the retriever section context even when a paragraph refers to "this method" or "the header above."

## Skipping HTML cleanup altogether for social sources

HTML-to-Markdown cleanup pays off on blogs, product documentation, support centers, and standards specs. On social platforms, rendering TikTok or X web feeds into HTML just to strip 95% of the DOM back out wastes headless browser compute and still misses nested comments or engagement counts.

When your RAG index mixes web documentation with live market or customer feedback from social channels, pull the social side directly as structured JSON using [TikTok Search API](https://apify.com/novi/tiktok-search-api?fpr=7hce1m), [TikTok Comment API](https://apify.com/novi/tiktok-comment-api?fpr=7hce1m), or [Twitter - X.com Scraper](https://apify.com/xtdata/twitter-x-scraper?fpr=7hce1m). You can format the returned captions, replies, and timestamps into compact Markdown records with a five-line template and feed them straight into the same heading-aware chunker.
