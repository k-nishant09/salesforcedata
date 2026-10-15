# Project Notes

Personal scratch pad — todos, ideas, and reminders for the fusion-salesforce project.

---

## Running this thing

1. generate case report to csv file
2. run pipeline against file
3. upload data (nofeed) dashboard to git 
4. copy feed data to Box 

## dashboard

1. python3 scripts/build_dashboard_data.py ~/Downloads/Fusion-cases-jul22     
2. python3 -m http.server 8000 --bind 127.0.0.1
3. http://127.0.0.1:8000/fusion_dashboard.html
4. load from scripts/downloads/Fusion-cases-jul22-dashboard.json
5. macOS - shift-cmd-R reloads browser data

## To Do

- [ ] update the automatic script to pull correct reports 
- [ ] update readme with information about copying data into fusion/salesforce (Box)
- [ ] update readme with information about --min_cluster-size
- [ ] readme add descritpion about what filters and columns are used from Lightning
- [ ] chatbot but this could only be with local as you need access to LLM somewhere

---

## Ideas / Future Work

---

## Workflow - First 

1. download CSV from Lightning tool - use the base report to get case feed for tickets opened between a range of dates
2. Run consolidation: python3 scripts/consolidate_tickets.py data/2026-08-14/fusion-case-feed.csv
3. Run categorization: 
- python3 scripts/categorize_csp.py data/2026-08-14/fusion-case-feed-consolidated-merged.csv 
- python3 -u scripts/categorize_csp.py data/2026-08-14/fusion-case-feed-consolidated-merged.csv > 0814-category.log 2>&1 &
4. Run clustering: 
- python3 /Users/spike/repos/fusion-salesforce/scripts/cluster_csp.py data/2026-08-10/0810-fusion-case-feed-consolidated-Categories.csv --savecache
- python3 -u /home/smithjp/fusion-salesforce/scripts/cluster_csp.py data/2026-08-10/0810-fusion-case-feed-consolidated-Categories.csv  --savecache  > 0810-cluster.log 2>&1 &
5. Build dashboard: 
- python3 scripts/build_dashboard_data.py data/2026-08-10/0810-fusion-case-feed-consolidated-Clusters.csv 2>&1 &

OR

- python3 scripts/run_pipeline.py data/2026-08-17-test/fusion-case-feed.csv --mode full
- python3 -u scripts/run_pipeline.py data/2026-08-30/fusion-case-feed.csv --mode full > 08-30.log 2>&1 &

### test min-cluster-size
- python3 -u scripts/run_pipeline.py data/2026-09-17/fusion-case-feed.csv --mode full > salesforce.log 2>&1 &
- python3 -u scripts/run_pipeline.py data/test-3/fusion-case-feed.csv --mode full --min-cluster-size 3 > salesforce.log 2>&1 &


6. load dashboard : python3 -m http.server 8000 --bind 127.0.0.1
7. open local dashboard: http://127.0.0.1:8000/fusion_dashboard.html
8. load data into dashboard: data/2026-08-10/0810-fusion-case-feed-consolidated-Clusters-dashboard.json


---

## Workflow - Incremetnal

1. download CSV - use the "incremental" report to get modified data (Last Modified Data)
2. Run consolidation: 
- python3 scripts/consolidate_tickets.py data/2026-08-14/fusion-case-feed.csv
3. Run merge: 
- python3 scripts/merge_consolidated.py data/2026-08-14/fusion-case-feed-consolidated.csv data/2026-08-10/0810-fusion-case-feed-consolidated.csv
4. Run categorization: 
- python3 scripts/categorize_csp.py data/2026-08-14/fusion-case-feed-consolidated-merged.csv --mode incremental --categories data/2026-08-10
- python3 -u scripts/categorize_csp.py data/2026-08-14/fusion-case-feed-consolidated-merged.csv --mode incremental --categories data/2026-08-10 > 0814-category.log 2>&1 &
5. Run Clustering: pythonw3 scripts/cluster_csp.py data/2026-08-14/fusion-case-feed-consolidated-Categories.csv --incremental --cache data/2026-08-10
- python3 -u scripts/cluster_csp.py data/2026-09-23/fusion-case-feed-consolidated-merged-Categories.csv --mode incremental --cache data/2026-09-17 > salesforce.log 2>&1 &
6. Build dashboard: 
- python3 scripts/build_dashboard_data.py data/2026-10-06/fusion-case-feed-consolidated-merged-Clusters.csv

OR

   python3 -u scripts/run_pipeline.py data/2026-10-06/fusion-case-feed.csv --mode incremental --base data/2026-10-01 > salesforce.log 2>&1 &

7. load dashboard : python3 -m http.server 8000 --bind 127.0.0.1
8. open local dashboard: http://127.0.0.1:8000/fusion_dashboard.html
9. load data into dashboard: data/2026-10-06/0810-fusion-case-feed-consolidated-Clusters-dashboard.json

---

## fix categories
python3 scripts/add_subcategories.py data/2026-08-10/0810-fusion-case-feed-consolidated-Categories.csv

---

## Commit

git add fusion_dashboard.html scripts/build_dashboard_data.py scripts/categorize_csp.py scripts/cluster_csp.py scripts/csp_utils.py
git commit -m "Dashboard date filters, By Month tab, cluster script fixes, progress ETA, test mode"
git push origin main

---

## Renewing the GitHub Token (IBM)
tokens
1. Go to https://github.ibm.com/settings/ → **Generate new token (classic)**
   - Note: `fusion-salesforce local`
   - Scope: check **repo**
   - Copy the token immediately after generating

2. Update the remote URL with the new token:
   ```
   git remote set-url origin https://<NEW_TOKEN>@github.ibm.com/smithjp/fusion-salesforce.git
   ```

3. Push:
   ```
   git push origin main
   ```

---

## Prompt changes

- added sub-categories for upgrade
- added RDR for DF category

---

## Notes / Reminders

Good. Now update both scripts — import progress_log, remove datetime import (no longer needed directly), and replace the inline print+guard:

Now verify both modules import cleanly:

Output looks exactly right. Here's a summary of what changed:

scripts/csp_utils.py — added progress_log(idx, total, start_time) which:
Prints for rows 0–9 and every 25th row after that
Row 0 shows no ETA (no data yet to average)
All subsequent rows compute avg_per_row × remaining_rows and show the absolute estimated completion timestamp
scripts/categorize_csp.py — removed from datetime import datetime, replaced the inline guard+print with progress_log(idx, rows, start_time)
scripts/cluster_csp.py — same as above
great ; now justtify the output for readability, for example the 2nd line has one more character then the first:
2026-07-29 11:40:15 Processing 9 of 17403 | Estimated time of completion 2026-07-29 15:51:30

2026-07-29 11:40:28 Processing 25 of 17403 | Estimated time of completion 2026-07-29 15:38:34

The variable-width part is the row index (9 vs 25). I need to know the max width to pad against — that's len(str(total)):

Quick verification:

idx is now right-padded to the same width as total, so the of 17403 and ETA columns line up perfectly across all rows.

similar error in cluster_csp.py (current directory):


Same fix as categorize_csp.py — move the combined_text build to csp_utils.py so it's shared, then use it in both scripts.




I'll move the combined_text logic into csp_utils.py as a build_combined_text() helper, then use it in both scripts:

Now update both scripts to use it:

Verify clean imports:

build_combined_text() is now in scripts/csp_utils.py alongside the other shared helpers. Both cluster_csp.py and categorize_csp.py import and call it — handles any combination of Subject, Description, and Body columns gracefully.

seeing this error:
(base) spike:11:54AM ~/repos/fusion-salesforce % python3 scripts/cluster_csp.py scripts/downloads/4-fusion-case-feed-report-Categories.csv
Loading data...
Checking connectivity to model gateway...
Generating embeddings for clustering...
2026-07-29 11:55:03 Processing   0 of 200
Error - embedding endpoint failed for granite-embedding-125m at https://model-gateway-model-gateway.apps.f73l056.fusion.tadn.ibm.com. The gateway is reachable, but /v1/embeddings returned HTTP 500.

The embedding model name is wrong. Let me check what's configured:


The gateway is reachable but it's rejecting the model name. Let me query what models are actually available on that server:

The model name is correct. The HTTP 500 is coming from the gateway itself. Let me check what the embedding endpoint actually returns:

The endpoint works fine from curl. The issue is the model name in the response — it says "model": "ibm-granitegranite-embedding-1" but the script is sending granite-embedding-125m. The gateway accepts that model ID (it's in /v1/models), so the HTTP 500 must be happening for a different reason — likely the text being sent is too long and truncating at the gateway. Let me check the actual error body:



The HTTP 500 error handler swallows the actual response body, so we never see what the gateway is complaining about. Let me fix the error handler to print the response body, which will tell us the real cause:

Run the script again and paste the new error output — the gateway's response body will now be printed and we'll know exactly what it's rejecting.



The gateway itself is crashing — it's an EngineCore error inside vLLM, not something wrong with our code. This typically happens when the input text is too long and exceeds the model's token limit. The fix is to truncate combined_text before sending it to the embedding endpoint.

Let me check what the model's context limit is and how long our texts are:

Some texts are very long (up to 10k chars). granite-embedding-125m has a 512-token limit — roughly 2000 characters. The fix is to truncate combined_text in get_vllm_embedding(). 512 tokens ≈ ~2000 chars is a safe ceiling:

Done. The root cause was that granite-embedding-125m has a 512-token (~2000 character) context window — the 10th row happened to have 10,000+ characters which caused the vLLM engine to crash with HTTP 500.



---

## Adding / Changing Subcategories After a Full Run

If you run the complete pipeline and later add or change subcategories in `csp_categories.py`, you do **not** need to re-run the full pipeline. Only re-run these three steps:

**Step 1 — Re-classify subcategories** (LLM call only for affected category rows):
```bash
python3 scripts/add_subcategories.py <your>-Categories.csv [--resume]
```
This reads the existing `*-Categories.csv`, calls the LLM only for rows matching categories that have a fixed subcategory list in `SUBCATEGORIES`, and overwrites the file in-place (saves a `.bak` backup first).

**Step 2 — Re-cluster:**
```bash
# Full run:
python3 scripts/cluster_csp.py <your>-Categories.csv --mode savecache

# Incremental run:
python3 scripts/cluster_csp.py <your>-Categories.csv --mode incremental --cache <prior.embeddings-cache.npz>
```

**Step 3 — Rebuild dashboard:**
```bash
python3 scripts/build_dashboard_data.py <your>-Clusters.csv
```

**What you do NOT need to re-run:**
- `consolidate_tickets.py` — source data unchanged
- `merge_consolidated.py` — merge unchanged
- `categorize_csp.py` — top-level categories unchanged

** test new cluster counts
python3 -u scripts/cluster_csp.py \
  data/2026-10-01/fusion-case-feed-consolidated-merged-Categories.csv \
  --mode clusterfromcache \
  --cache data/2026-10-01 \
  --min-cluster-size 3  > salesforce-3.log 2>&1 &


python3 -u scripts/cluster_csp.py data/2026-10-06/fusion-case-feed-consolidated-merged-Categories.csv \
  --mode clusterfromcache --cache data/2026-10-06 --min-cluster-size 3 > salesforce-3b.log 2>&1 &

