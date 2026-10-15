# Incremental Processing Notes

Notes on making `categorize_csp.py` and `cluster_csp.py` work incrementally on large, regularly-updated datasets without reprocessing the entire dataset each run.

---

## The core problem

Both scripts are full-recompute operations by design:

| Script | Why it's slow | What makes incremental hard |
|---|---|---|
| `categorize_csp.py` | One LLM call per row (~1–2s each) | Categories are per-row and independent — easy to make incremental |
| `cluster_csp.py` | Embedding per row + HDBSCAN over the entire matrix | HDBSCAN is a global algorithm — adding new points changes cluster assignments for all existing points |

---

## `categorize_csp.py` — fully incremental is straightforward

Each row is classified independently, so a proper incremental mode is clean to implement:

1. Keep a persistent output CSV alongside your source data (the "seen" set, keyed on a stable row ID such as a ticket/case number).
2. On each run, diff the input against the output — only process rows not already present in the output.
3. Append new results to the output rather than overwriting it.

The existing `--resume` flag already does something similar, but it only recovers an interrupted session. A true `--incremental` mode would persist the output permanently across separate runs.

**Implementation sketch (~20 lines):**
```
# --incremental: load existing output CSV, extract already-seen ticket IDs,
#                only process rows whose ID is not in the output,
#                then append new rows to the output (rather than overwriting)
```

---

## `cluster_csp.py` — true incrementality is fundamentally limited

HDBSCAN is a **global density algorithm** — when new tickets are added, the density landscape shifts and cluster boundaries change. There is no mathematically correct way to add rows without re-clustering everything. Options:

### Option 1: Persistent embedding cache (recommended)
Save embeddings permanently (keyed by ticket ID or a hash of `Subject+Description`). On each run:
1. Load the existing embedding cache.
2. Identify new rows (by ticket ID or row hash).
3. Embed only the new rows, add them to the cache, save the cache.
4. Run HDBSCAN over the full combined matrix.

This eliminates the dominant cost — the embedding API calls. With 10k tickets at ~2s each, embeddings take hours; HDBSCAN itself runs in seconds over the resulting matrix. The `.npy` checkpoint pattern already exists within a single run — this extends it across runs.

**Implementation sketch:**
```
# Persistent cache: .npz file storing {ids: [...], embeddings: [...]}
# On each run, load cache, find rows not in cache, embed only those,
# merge into cache, save, then run HDBSCAN over the full matrix.
```

### Option 2: Assign new rows to existing clusters (fast, approximate)
Embed only new rows, then for each one find the nearest existing cluster centroid and assign that cluster label. Skip re-clustering entirely.

- Pro: very fast, cluster IDs never drift between runs.
- Con: new emergent clusters can never be discovered; assignments are approximate.

### Option 3: Periodic full recluster (pragmatic)
Run the full pipeline on a schedule (weekly/monthly) and accept that cluster IDs may shift over time. Use the category columns from `categorize_csp.py` — which are stable per-row — as the primary analytical artifact.

---

## Stable row key

Any incremental implementation needs a stable identifier per ticket. Candidates:
- `Case Number` column (if present and unique)
- A hash of `Subject` + `Description` (deterministic, no extra column needed)

---

## Summary of recommended changes

| Script | Change | Effort |
|---|---|---|
| `categorize_csp.py` | Add `--incremental` flag: load existing output, skip seen IDs, append new rows | Low (~20 lines) |
| `cluster_csp.py` | Persistent `.npz` embedding cache keyed by ticket ID/hash; embed only new rows per run | Medium (~40 lines) |
