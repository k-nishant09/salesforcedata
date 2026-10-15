# Salesforce Playwright Scripts

This repository contains two helper scripts in [`scripts`](scripts) for automating a Salesforce report export with Playwright.

## Prerequisites

- Python 3
- Playwright for Python
- Chromium installed for Playwright

Install the dependencies with:

```bash
python3 -m pip install -r requirements.txt
python3 -m playwright install chromium
```

## Scripts

- [`scripts/salesforce_login_state.py`](scripts/salesforce_login_state.py): opens Salesforce in a browser so you can log in manually, then saves session state to [`scripts/playwright-state.json`](scripts/playwright-state.json)
- [`scripts/salesforce_export_report.py`](scripts/salesforce_export_report.py): reuses the saved session, updates the report filter to `Last 7 Days`, exports `Details Only` as CSV, and saves the file to [`scripts/downloads`](scripts/downloads)

## Usage

From the repository root, run:

```bash
python3 scripts/salesforce_login_state.py
python3 scripts/salesforce_export_report.py
```

Recommended flow:

1. Run [`scripts/salesforce_login_state.py`](scripts/salesforce_login_state.py) and log in through the opened browser window.
2. Once the Salesforce report page is fully visible, return to the terminal and press Enter.
3. Run [`scripts/salesforce_export_report.py`](scripts/salesforce_export_report.py) to export the CSV.

If the Salesforce session expires, rerun [`scripts/salesforce_login_state.py`](scripts/salesforce_login_state.py) to refresh [`scripts/playwright-state.json`](scripts/playwright-state.json).

## Analysis pipeline

The core pipeline takes a raw Salesforce case export CSV through four scripts and produces a dashboard JSON. All scripts are run from the repository root.

### Dependencies

```bash
python3 -m pip install -r requirements.txt
```

### Recommended: End-to-End Pipeline Runner

Use [`run_pipeline.py`](scripts/run_pipeline.py) to run the full workflow in a single command.

#### Use Case 1: First Run (Full)
Processes raw feed export from scratch, classifies with LLM, embeds all tickets, saves cache, and builds dashboard files:
```bash
python3 scripts/run_pipeline.py data/YYYY-MM-DD/MMDD-fusion-case-feed.csv --mode full
```

#### Use Case 2: Incremental Run (Weekly Delta)
Consolidates the new delta feed, auto-discovers base files in `--base`, merges them, categorizes new tickets, embeds new tickets, re-clusters, and builds dashboard files:
```bash
python3 scripts/run_pipeline.py data/YYYY-MM-DD/MMDD-fusion-case-feed.csv --mode incremental --base data/PREVIOUS-DIR
```

---

### Individual Scripts Overview

You can also run any individual step manually:

| Script | Input | Output | LLM calls |
|---|---|---|---|
| [`run_pipeline.py`](scripts/run_pipeline.py) | Raw export CSV + mode | All final JSONs & caches | Automated |
| [`consolidate_tickets.py`](scripts/consolidate_tickets.py) | Raw export CSV | `*-consolidated.csv` | None |
| [`merge_consolidated.py`](scripts/merge_consolidated.py) | Delta CSV + `--base` | `*-consolidated-merged.csv` | None |
| [`categorize_csp.py`](scripts/categorize_csp.py) | `*-consolidated.csv` | `*-Categories.csv` | 1–2 per ticket |
| [`cluster_csp.py`](scripts/cluster_csp.py) | `*-Categories.csv` | `*-Clusters.csv` | 1 per cluster + embedding per ticket |
| [`build_dashboard_data.py`](scripts/build_dashboard_data.py) | `*-Clusters.csv` | `*-dashboard*.json` | None |

> **VPN required** — `categorize_csp.py` and `cluster_csp.py` call the IBM model gateway. Make sure your VPN is connected before running them.

---

### First run (fresh CSV)

Use this flow the first time you process a new data file.

**Step 1 — Consolidate**

Merges multi-row case feed entries into one row per ticket, strips HTML, and drops noise/mojibake cases.

```bash
python3 scripts/consolidate_tickets.py data/YYYY-MM-DD/MMDD-fusion-case-feed.csv
# Output: data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated.csv
```

**Step 2 — Categorize**

Calls the LLM to classify each ticket into one or more categories (and an Upgrades subcategory where applicable). Tickets classified as Upgrades get a second LLM call to assign a subcategory.

```bash
python3 scripts/categorize_csp.py data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated.csv
# Output: data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated-Categories.csv
```

If the run is interrupted, resume without re-processing completed rows:

```bash
python3 scripts/categorize_csp.py data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated.csv --resume
```

**Step 3 — Cluster**

Generates embeddings for each ticket and runs HDBSCAN to group similar issues into clusters, then generates a plain-English description for each cluster. This is the slowest step — embeddings take several seconds per ticket due to gateway throttling.

On a **first run**, use `--savecache` to embed all rows and save the embedding cache for future use:

```bash
python3 scripts/cluster_csp.py data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated-Categories.csv --savecache
# Output: data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated-Clusters.csv
#         data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated.embeddings-cache.npz
```

If the run is interrupted, resume without re-embedding completed rows:

```bash
python3 scripts/cluster_csp.py data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated-Categories.csv --savecache --resume
```

**Step 4 — Build dashboard JSON**

Aggregates the Clusters CSV into the three JSON files consumed by the dashboard.

```bash
python3 scripts/build_dashboard_data.py data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated-Clusters.csv
# Output:
#   *-dashboard.json          — full raw rows (local only, gitignored — contains customer data)
#   *-dashboard-nofeed.json   — raw rows with feed columns stripped (safe to commit)
#   *-dashboard-public.json   — aggregated counts only, no raw rows (smallest, safe to commit)
```

---

### Incremental run (new data added to an existing dataset)

After the first full run, use `--incremental` on subsequent runs to skip tickets that have already been processed. Only new `Case Number` values are sent to the LLM or embedding API.

**Step 1 — Consolidate delta export**

```bash
python3 scripts/consolidate_tickets.py data/YYYY-MM-DD/MMDD-fusion-case-feed.csv
# Output: data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated.csv
```

**Step 1b — Merge delta with previous cumulative dataset (if delta export)**

If your weekly Salesforce report only exports new/updated tickets (e.g. "Last 7 Days"), merge it with the previous week's consolidated CSV before categorizing. It automatically handles overlapping date duplicates by taking the latest ticket version from delta:

```bash
python3 scripts/merge_consolidated.py \
  data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated.csv \
  --base data/PREVIOUS-DATE
# Output: data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated-merged.csv
```

**Step 2 — Categorize incrementally**

Loads the existing `-Categories.csv`, copies already-classified rows by `Case Number`, and only calls the LLM for new tickets. Overwrites the output in place.

```bash
python3 scripts/categorize_csp.py data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated.csv --incremental
```

If your new CSV is in a different dated folder from the previous run, point `--categories` at the previous run's directory (it will find the `*-Categories.csv` automatically) or directly at the file:

```bash
python3 scripts/categorize_csp.py data/2026-08-11/0811-fusion-case-feed-consolidated.csv \
  --incremental \
  --categories data/2026-08-10
```

**Step 3 — Cluster incrementally**

Loads the existing `.embeddings-cache.npz`, embeds only new tickets, and re-runs HDBSCAN over the full matrix to maintain clustering quality.

```bash
python3 scripts/cluster_csp.py data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated-Categories.csv --incremental
```

If your new CSV is in a different dated folder from the previous run, point `--cache` at the previous run's directory (it will find the `*.embeddings-cache.npz` automatically) or directly at the file:

```bash
python3 scripts/cluster_csp.py data/2026-08-11/0811-fusion-case-feed-consolidated-Categories.csv \
  --incremental \
  --cache data/2026-08-10
```

The updated cache is always saved beside the new input CSV after the run completes.

**Step 4 — Build dashboard JSON** (same as first run)

```bash
python3 scripts/build_dashboard_data.py data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated-Clusters.csv
```

---

### Backfill Upgrades subcategory on an existing Categories CSV

If you already have a `-Categories.csv` from a previous run and want to add the `Upgrades Subcategory` column without re-running the full categorization:

```bash
python3 scripts/add_subcategories.py data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated-Categories.csv
```

This only calls the LLM for rows where `Upgrades == Y`. The input file is overwritten in place; a `.bak.csv` backup is saved first.

---

## Static HTML dashboard

The dashboard is permanently hosted via GitHub Pages at:

**<https://pages.github.ibm.com/smithjp/fusion-salesforce/fusion_dashboard.html>**

It always reflects the latest version of [`fusion_dashboard.html`](fusion_dashboard.html) on the `main` branch and auto-loads the most recent run's data on page open.

### What the dashboard shows

| Tab | What it shows |
|---|---|
| **Overview** | Case counts per issue category with a date-range filter |
| **By Release** | Horizontal bar charts breaking down cases per category by Fusion release version; subcategory picker for Upgrades |
| **By Month** | Monthly case volume as a bar + line chart; filterable by category and date range |
| **Top Accounts** | Accounts with the most open cases, broken down by category |
| **By Issue** | Clickable table of clustered issue descriptions; selecting a row shows a per-release breakdown for that cluster |
| **Export Data** | Date-filtered CSV export of raw case rows (feed columns excluded on the hosted version) |

### Data files committed to git

Each run produces three JSON files under `data/YYYY-MM-DD/`. Two are committed and served by the hosted dashboard:

| File | Contents | Committed |
|---|---|---|
| `*-dashboard.json` | Full raw rows including `Comments` and `Body` feed columns | ❌ No — gitignored (contains customer data) |
| `*-dashboard-nofeed.json` | Raw rows with feed columns stripped; includes scrubbed `Description` | ✅ Yes — **default** loaded by hosted dashboard |
| `*-dashboard-public.json` | Pre-aggregated counts only, no raw rows | ✅ Yes — smallest, fully safe fallback |

The hosted dashboard loads `*-dashboard-nofeed.json` by default. This enables all tabs including date filtering and CSV export. To use a local full JSON (with feed data), enter its path in the **Dashboard JSON path** panel on the Export Data tab.

### Publishing a new run

After building the dashboard JSON, update the default path in [`fusion_dashboard.html`](fusion_dashboard.html) (the line with `value="data/..."`):

```html
value="data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated-Clusters-dashboard-nofeed.json"
```

Then commit and push:

```bash
git add data/YYYY-MM-DD/*-dashboard-nofeed.json data/YYYY-MM-DD/*-dashboard-public.json fusion_dashboard.html
git commit -m "Data run YYYY-MM-DD"
git push origin main
```

## Local dashboard

To serve the dashboard locally against a full JSON (with raw feed data):

```bash
python3 -m http.server 8000 --bind 127.0.0.1
```

Open <http://127.0.0.1:8000/fusion_dashboard.html> and load:

```text
data/YYYY-MM-DD/MMDD-fusion-case-feed-consolidated-Clusters-dashboard.json
```
