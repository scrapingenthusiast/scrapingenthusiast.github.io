---
layout: post
title: "How to Combine Multiple Apify TikTok Scraper Runs into One Excel File"
permalink: /combine-multiple-apify-tiktok-runs-excel/
---

# How to Combine Multiple Apify TikTok Scraper Runs into One Excel File

If you've been using Apify to scrape TikTok data at scale, you've probably already discovered that breaking your job into multiple **Actor runs** is the smart way to go. Smaller runs don't time out, they're easier to retry if something goes wrong, and they get around the pesky **pagination limits** that come with large searches. But once those runs are done, you're left with a collection of separate **Datasets** — and the real question becomes: how do you pull them all together into a single spreadsheet you can actually work with?

That's exactly what this tutorial covers. We'll use Python's **Apify client** and **Pandas** to fetch the results from multiple `novi/fast-tiktok-api` runs, handle the nested JSON response format, and export everything neatly into a single `.xlsx` file.

---

## 1. Getting Your Environment Ready

Before anything else, you'll need to install three packages. They work together to handle the API communication, data manipulation, and Excel export.

```bash
pip install pandas apify openpyxl
```

Here's what each one does:

* **`apify`**: Connects you to the Apify API so you can trigger actors and fetch dataset results.
* **`pandas`**: The powerhouse for wrangling your data into shape.
* **`openpyxl`**: The underlying engine Pandas uses when writing `.xlsx` files.

---

## 2. Writing the Script Step by Step

### Step 1: Connect to Apify

First things first — set up your client. For a quick local test, you can just paste your API token directly into the script. If you're building something more permanent, it's worth pulling the token from an environment variable instead.

```python
import pandas as pd
from apify_client import ApifyClient

APIFY_API_TOKEN = "your_token_here"  # or os.getenv("APIFY_API_TOKEN")
ACTOR_ID = "novi/fast-tiktok-api"

client = ApifyClient(APIFY_API_TOKEN)
```

### Step 2: Run the Actor for Each Keyword

Now define the list of search terms you want to scrape, then loop through them and trigger a separate Actor run for each one. The `try/except` block is important here — if one keyword fails (say, the API throws an error), you want the rest of the runs to continue rather than the whole script crashing.

```python
queries = ["dance", "coding"]
dataset_ids = []

for query in queries:
    print(f"Triggering Run: query='{query}'...")
    try:
        run = client.actor(ACTOR_ID).call(run_input={
            "type": "SEARCH",
            "keyword": query,
            "limit": 10
        })
        dataset_ids.append((query, run['defaultDatasetId']))
        print(f"  → Dataset ID: {run['defaultDatasetId']}")
    except Exception as e:
        print(f"  ✗ Actor error ({query}): {e}")
```

### Step 3: Pull the Data and Flatten It

Here's where a lot of people run into trouble. The data that comes back from the TikTok scraper is **nested JSON** — things like video stats, author info, and music details are stored as objects within objects. If you dump this directly into a regular Pandas DataFrame, those nested fields turn into unhelpful raw dictionary columns.

The fix is `pd.json_normalize()`, which automatically unwraps those nested structures into individual flat columns.

```python
dataframes = []

for query, dataset_id in dataset_ids:
    print(f"Fetching data from dataset: {dataset_id}...")
    items = list(client.dataset(dataset_id).iterate_items())

    if items:
        df = pd.json_normalize(items)   # This flattens the nested JSON
        df["source_query"] = query      # Track which keyword produced this data
        dataframes.append(df)
        print(f"  → {len(df)} records fetched")
```

### Step 4: Merge Everything and Save

Once you have all the individual DataFrames, merging them is straightforward. Then export to Excel — just make sure to set `index=False` so Pandas doesn't add an unwanted row-number column to your spreadsheet.

```python
if dataframes:
    merged_df = pd.concat(dataframes, ignore_index=True)
    output_file = "./Merged_TikTok_Data.xlsx"
    merged_df.to_excel(output_file, index=False, engine="openpyxl")
    print(f"\n✅ Done! {len(merged_df)} records saved to: {output_file}")
else:
    print("⚠️  No data extracted.")
```

---

## 3. The Complete Script

Here's everything put together in one place, ready to run:

```python
# Prerequisites: pip install pandas apify openpyxl

import pandas as pd
from apify_client import ApifyClient

# ── Config ────────────────────────────────────────────────────────────────────
APIFY_API_TOKEN = "your_token_here"
ACTOR_ID = "novi/fast-tiktok-api"

client = ApifyClient(APIFY_API_TOKEN)

# ── Input ─────────────────────────────────────────────────────────────────────
queries = ["dance", "coding"]
dataset_ids = []

# ── Dispatch Actor Runs ───────────────────────────────────────────────────────
for query in queries:
    print(f"Triggering Run: query='{query}'...")
    try:
        run = client.actor(ACTOR_ID).call(run_input={"type": "SEARCH", "keyword": query, "limit": 10})
        dataset_ids.append((query, run['defaultDatasetId']))
        print(f"  → Dataset ID: {run['defaultDatasetId']}")
    except Exception as e:
        print(f"  ✗ Actor error ({query}): {e}")

# ── Data Collection and Normalization ─────────────────────────────────────────
dataframes = []
for query, dataset_id in dataset_ids:
    print(f"Fetching data from dataset: {dataset_id}...")
    items = list(client.dataset(dataset_id).iterate_items())

    if items:
        df = pd.json_normalize(items)  # Flatten nested JSON
        df['source_query'] = query
        dataframes.append(df)
        print(f"  → {len(df)} records fetched")

# ── Merge and Export ──────────────────────────────────────────────────────────
if dataframes:
    merged_df = pd.concat(dataframes, ignore_index=True)
    output_file = "./Merged_TikTok_Data.xlsx"
    merged_df.to_excel(output_file, index=False, engine="openpyxl")
    print(f"\n✅ Done! {len(merged_df)} records saved to: {output_file}")
else:
    print("⚠️  No data extracted.")
```

---

## 4. Where to Take This Next

This script is great for one-off analysis. If you start needing to run this regularly or at higher volume, here are some improvements worth looking into:

* **Run multiple actors in parallel:** Right now each Actor call blocks until it's done before starting the next one. Switching to `asyncio` and the async Apify client lets you fire them all off at the same time, which dramatically cuts your total wait time.
* **Move away from Excel for large datasets:** Excel maxes out at about one million rows. If your data is growing, consider writing directly to a database like **PostgreSQL**, or storing results as **Parquet** files on something like AWS S3.
* **Handle rate limits gracefully:** If you're running many keywords back-to-back, Apify may return a `429 Too Many Requests` error. The `tenacity` library makes it easy to add automatic retry logic with exponential backoff so your script can recover from those situations without you having to babysit it.
