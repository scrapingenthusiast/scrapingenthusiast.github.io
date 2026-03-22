---
layout: post
title: "Building an End-to-End Web Scraping Pipeline: A Complete Guide"
permalink: /web-scraping/end-to-end-web-scraping-pipeline/
categories: [Web Scraping, Data Pipeline]
tags: [httpx, selectolax, python]
---

Building a web scraping application requires moving beyond isolated, single-run scripts to designing a resilient, automated data pipeline. This guide covers the technical lifecycle of building a functional scraping project—using a price tracker as our baseline architecture—from network requests to data persistence.

## Defining Scope and Data Architecture

Before writing extraction logic, you must define your data model and assess the target environment. 

### 1. Data Modeling
Define the exact schema for your extraction. For a price tracker, the minimum required fields are:
* **`product_id`**: A unique identifier (often the SKU or a hash of the URL).
* **`product_name`**: The raw string of the item.
* **`price_current`**: The parsed numeric value (e.g., float or integer representing cents).
* **`timestamp_utc`**: The exact time of the scrape in ISO 8601 format.

### 2. Target Assessment (Static vs. Dynamic)
Inspect the network traffic of your target URL using browser developer tools. 
* **Server-Side Rendered (SSR) / Static:** If the target data exists in the initial HTML document payload, you can rely on standard HTTP clients.
* **Client-Side Rendered (CSR) / Dynamic:** If the data populates via subsequent XHR/Fetch requests or requires JavaScript execution, you will need browser automation or API reverse-engineering.

### 3. Legal and Compliance Boundaries
Scraping carries operational and legal risks. Always implement the following checks:
* **`robots.txt`**: Parse the target's `robots.txt` file to verify permitted paths and explicit crawl delays.
* **Rate Limiting**: Implement sleep mechanisms or token bucket algorithms to avoid overwhelming the host server.
* **Terms of Service (ToS)**: Ensure your scraping does not violate explicit terms, particularly regarding authenticated endpoints or personally identifiable information (PII).

## Selecting the Modern Scraping Stack

Legacy tools like `requests` and `BeautifulSoup` are fine for tutorials, but production pipelines require speed and evasion capabilities. 

### For Static Payloads
* **HTTP Client:** Use **`httpx`** instead of `requests`. It supports async operations and HTTP/2, which is crucial for modern scraping and mimicking real browser behavior.
  
  ```python
  import httpx
  import asyncio

  async def fetch_page(url):
      headers = {
          "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
          "Accept-Language": "en-US,en;q=0.9"
      }
      async with httpx.AsyncClient(http2=True) as client:
          response = await client.get(url, headers=headers)
          return response.text
  ```

* **HTML Parser:** Use **`selectolax`** or **`lxml`**. They are written in C and are orders of magnitude faster than `BeautifulSoup` for parsing large DOM trees.

  ```python
  from selectolax.parser import HTMLParser

  html = "<div class='price'>$1,299.99</div>"
  tree = HTMLParser(html)
  price_node = tree.css_first('.price')
  print(price_node.text()) # $1,299.99
  ```

### For Dynamic Payloads
* **Browser Automation:** Use **Playwright** (with Python or Node.js). It is significantly faster and more reliable than Selenium. 
* **Anti-Bot Evasion:** Headless browsers are easily detected. You will need to patch your browser instances using tools like **`playwright-stealth`** to mask your `navigator.webdriver` flags and manage **TLS fingerprinting**.

## Core Extraction and Normalization Logic

The extraction phase involves retrieving the document, locating the nodes, and sanitizing the output.

### 1. Network Execution and Evasion
Basic HTTP requests will get blocked by WAFs (Web Application Firewalls) like Cloudflare or Akamai. Ensure you:
* Pass a complete set of **Headers**, including a modern `User-Agent`, `Accept-Language`, and `Sec-Fetch-*` metadata.
* Implement **Proxy Rotation** (Residential or Datacenter IP pools) if scraping at volume to avoid IP bans.

### 2. DOM Parsing
Use highly specific **CSS Selectors** or **XPath** queries to target your data. 
* *Example:* Target the price via `<span data-test-id="product-price">` rather than a generic CSS class that might change during the site's next deployment.

### 3. Data Normalization
Raw scraped data is inherently dirty. You must clean it before storage:
* Strip currency symbols and commas using **Regex (Regular Expressions)**.
* Cast the cleaned string to a standard float format.

  ```python
  import re

  def clean_price(raw_price_str):
      # Extract numbers and decimal point
      clean_str = re.sub(r'[^\d.]', '', raw_price_str)
      return float(clean_str)

  print(clean_price("$1,299.99\n")) # 1299.99
  ```

## Data Storage and Persistence

Data must be structured and queryable. 

* **Development/Local:** Use **CSV** files or **SQLite**. SQLite allows you to write actual SQL queries and prevents the file corruption risks inherent to appending to flat CSVs.
* **Production:** Migrate to a relational database like **PostgreSQL**. If the schema varies heavily between target sites, a NoSQL document store like **MongoDB** provides flexibility.

## Automation and Alerting

A price tracker is useless if it doesn't run continuously.

### 1. Scheduling
Deploy your script to run at fixed intervals.
* **Lightweight:** Use a standard Linux **`cron`** job or **GitHub Actions** (using scheduled workflow dispatches).
* **Enterprise:** Use workflow orchestration tools like **Apache Airflow** to manage retries, dependencies, and complex schedules.

### 2. Event Notifications
Set up logic to act on the data. Compare the newly inserted `price_current` against the previous database entry. If the price drops below a threshold, trigger an HTTP POST request to a **Slack or Discord Webhook** to notify you instantly.
