# Redirects Workflow Guide

## TL;DR: Quickstart

**Goal:** Automatically generate **cross-version redirects** between Gravitee documentation versions (e.g., APIM 4.0 → 4.1 → 4.2), ensuring every page knows its forward and backward equivalents.

### 1️⃣ Set up your environment

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 2️⃣ Prepare your inputs

- Place versioned Markdown docs in the appropriate source directory (e.g., `docs/apim/4.0`, `docs/apim/4.1`, ...)
- Ensure folder structure matches GitBook expectations.

### 3️⃣ Run the entire workflow

```bash
bash run_all.sh apim
```

This will:

1. Extract all text from Markdown files.
2. Create embeddings.
3. Build an approximate search index.
4. Match similar pages across versions.
5. Generate forward/backward redirect YAMLs.
6. Reconcile them into clean, de-duplicated redirect sets.

### 4️⃣ Output summary

After a successful run, you’ll have:

- **`out/apim/yaml/apim.<ver>.gitbook.redirects.yaml`** — raw redirect files per version.
- **`out/apim/reconciled/`** — reconciled YAMLs with duplicates and no-ops removed.
- **`out/apim/strip_collisions.csv`** — logs showing how duplicate paths were resolved.

---

## Purpose and Overview

### Why we need redirects

As documentation evolves, pages are renamed, merged, or split. We need **redirects** so that users following links to older versions still end up on the right new page — or at least a relevant fallback.

This workflow builds **bidirectional redirects** (forward and backward) automatically, by comparing the _content_ of pages across all versions.

In short: it ensures `4.0/foo/bar.md` → `4.2/guides/bar.md` and vice versa.

### Conceptual diagram

```
 ┌──────────────────────────────────────────────┐
 │                Documentation                │
 │          (4.0, 4.1, 4.2, 4.3...)            │
 └──────────────────────────────────────────────┘
              │
              ▼
     [1] Extract + Embed Content
              │
              ▼
     [2] LSH + Similarity Matching
              │
              ▼
     [3] YAML Redirect Generation
              │
              ▼
     [4] Reconcile + Strip Versions
              │
              ▼
     [5] Clean, Stable Redirects
```

---

## Detailed Step-by-Step Explanation

### 1. `extract_docs.py` — Text extraction

**Purpose:** Read every Markdown file, clean it, and produce structured content suitable for embedding.

**In plain terms:** This script walks through all documentation versions (e.g., `4.0/`, `4.1/`, etc.), opens each `.md` file, removes front matter and formatting, and outputs plain text. Each record keeps track of:

- Its version (e.g., `4.0`)
- Its relative path (`guides/install.md`)
- The cleaned text body

**Technical note:** It outputs a single JSON or CSV dataset used by later stages.

---

### 2. `content_index.py` — Embeddings and vectorization

**Purpose:** Convert textual page content into numerical embeddings using a transformer model.

**In plain terms:** Each page becomes a long list of numbers (a “vector”) that represents its meaning. Pages with similar text produce similar vectors.

**Technical detail:** Uses a sentence-transformer model (e.g., `all-MiniLM-L6-v2`). The vectors are stored alongside the page metadata.

---

### 3. `build_lsh_index.py` — Approximate search

**Purpose:** Build a **Locality-Sensitive Hashing (LSH)** index so we can find similar pages quickly without comparing every pair.

**In plain terms:** Imagine putting all pages into buckets where similar ones end up together. This makes finding related pages much faster.

**Technical detail:**

- Hashes embedding vectors into bit signatures.
- Stores them in a lightweight search index.
- Enables efficient nearest-neighbor lookups between versions.

---

### 4. `build_content_clusters.py` — Cross-version page matching

**Purpose:** Match equivalent or similar pages between all versions.

**In plain terms:** This is where the magic happens. The script compares every page in one version to pages in another version and records which ones are the same or nearly identical.

**Technical detail:**

- Uses cosine similarity between embedding vectors.
- Produces clusters of pages that share high semantic overlap (e.g., ≥ 0.8 similarity).
- Output: a JSON or CSV linking `(version_A/page_X)` ↔ `(version_B/page_Y)`.

---

### 5. `make_yaml_redirects.py` — Generate forward/backward redirects

**Purpose:** Turn those cross-version matches into **GitBook-compatible YAML redirect files.**

**In plain terms:** For every matched pair, it writes a redirect like this:

```yaml
redirects:
  4.0/getting-started/foo: getting-started/foo.md
  4.1/getting-started/foo: getting-started/foo.md
```

Each target version gets its own YAML file (`apim.4.1.gitbook.redirects.yaml`, etc.) listing where older pages should point inside it.

**Technical detail:**

- Each YAML corresponds to one target version.
- The source side always includes the version prefix (`4.0/foo` → `bar.md`).
- These files serve as the raw redirect data.

---

### 6. `reconcile_bidirectional_redirects.py` — Cleanup and normalization

**Purpose:** Reconcile and simplify the raw YAMLs to remove redundancy and inconsistencies.

**In plain terms:** Imagine we have these:

```
4.0/foo → README.md
4.1/foo → foo.md
```

The script checks both directions (4.0 → 4.1 and 4.1 → 4.0) and concludes that the concrete `foo.md` is better than `README.md`. So it upgrades the redirect and drops the weaker fallback.

**Technical detail:**

- Loads all redirect YAMLs.
- Cross-compares all version pairs.
- Replaces “README.md” fallbacks with concrete matches.
- Removes no-op redirects (where the source equals the destination).
- Then strips version prefixes from keys (`4.0/foo` → `foo`) while merging duplicates.

### No-op redirects explained

A _no-op_ means the redirect points to itself. For example:

```
4.0/foo → foo.md
```

Since this is effectively already the same page, it’s removed.

---

## Post-Processing and Collision Handling

After version-stripping, multiple redirects might now share the same `from` key. For instance:

```
4.0/guides/foo → guides/foo.md
4.1/guides/foo → guides/bar.md
4.2/guides/foo → guides/foobar.md
```

becomes:

```
guides/foo: guides/foo.md
```

But we need to decide _which_ destination to keep.

### Keep Policy (Current)

**We keep the most concrete, non-README destination.** If both are README or both are concrete, we keep the first one (deterministic order).

**Alternative options:**

1. **Latest-wins:** Always prefer the destination from the newest version.
2. **Oldest-wins:** Keep the earliest mapping for historical continuity.
3. **Weighted:** Combine metadata (e.g., page length, similarity score) to choose the strongest match.

**CSV logs:** Every time a duplicate is removed, it logs the decision in `strip_collisions.csv`:

```
yaml_version,from_key_collapsed,kept_dst,dropped_dst,dropped_from_key
4.0,guides/foo,guides/foo.md,guides/bar.md,4.1/guides/foo
```

---

## Example: Reconciliation Workflow

```
      ┌──────────────────────┐
      │ Raw redirect YAMLs  │
      └──────────────────────┘
                │
                ▼
    [1] Bidirectional upgrade pass
        - README → concrete pages
                │
                ▼
    [2] No-op removal
        - Drop self-redirects
                │
                ▼
    [3] Strip version prefixes
        - 4.0/foo → foo
                │
                ▼
    [4] Collision resolution
        - Merge duplicates (keep best)
                │
                ▼
      ┌──────────────────────┐
      │ Clean, stable YAMLs │
      └──────────────────────┘
```

---

## Determinism and Reproducibility

The workflow is fully deterministic:

- Versions are processed in sorted numeric order.
- Conflicts are resolved consistently (shortest → lexicographic).
- Output is identical across runs given the same inputs.

This ensures stable diffs in version control and predictable results when re-running the workflow.

---

## Debugging and Validation

- `strip_collisions.csv`: shows which mappings collapsed.
- Console output: reports how many entries were upgraded or removed per version.
- Re-run with `--yamls_glob` targeting intermediate YAMLs to inspect.

---

## Appendix: Design Notes

### Why embedding-based matching?

Static path comparison doesn’t work because pages get renamed or moved. Embeddings capture **semantic meaning**, so a renamed page (“Setting up Gateway”) will still match “Configuring the Gateway.”

### Why bidirectional reconciliation?

Sometimes a match exists only one way — `A → B` but not `B → A`. The reconciliation process ensures both directions are represented and aligned.

### Why version stripping?

GitBook expects redirects to be version-agnostic once merged. Removing prefixes produces portable, single-layer mappings that work across all docs.

---

## Final Outcome

The result is a **clean, deterministic redirect map** where every doc version gracefully points to its equivalent in other versions — no broken links, no duplication, and full cross-version navigation.
