# Jira “Time in Status” Metrics Collection & Post‑Processing Guide

This document explains how our metrics are collected from Jira, how we post‑process the raw CSVs, and what every output column means. It’s written so you can walk a teammate through the whole flow, end‑to‑end.

---

## What we’re measuring (in one paragraph)

For a given board filter (JQL), we pull all matching issues from Jira Cloud, fetch each issue’s full changelog, reconstruct the **exact time windows an issue spent in each status**, and then:

- export **per‑issue totals** (sum of time across all visits to a status),
- export **per‑status aggregates** (board‑wide and per‑project), and
- optionally export **raw intervals** (one row per contiguous time-in-status segment).

The main export writes **hours**. A post‑processing script then:

- **maps** raw Jira status labels to our **display labels**,
- **merges collisions** (e.g., _In Progress_ + _In Code Review_ → **Started**),
- **converts hours → days** (24h per day), and
- **orders columns** exactly the way we want.

> We intentionally count **all time spent** in a status across repeats (back‑and‑forth moves are common). If an issue enters a status multiple times, those durations **all** contribute to the per‑issue total for that status.

---

## Components

### 1) Main exporter — `jira_time_in_status_total.py`

**Purpose**

- Pull issues by JQL.
- Fetch the **full changelog** for each issue.
- Rebuild status intervals and compute per‑status **total hours** per issue.
- Write CSVs.

**Key inputs**

- **Config file** (`jira_metrics.toml` or `.json`) next to the script.
- Items you can set in config (or CLI/env):
    - `base_url`, `email`, `api_token`
    - `jql` (your board filter)
    - `statuses` (ordered list of **raw Jira status names** you care about in the output)
    - `pagesize`, `sleep`, `cutoff_now`
    - file names for outputs (`out`, `intervals_out`, `agg_out`), `aggregate`, `percentiles`

**APIs used**

- Search (enhanced): `POST /rest/api/3/search/jql`
- Changelog (paged): `GET /rest/api/3/issue/{issueKey}/changelog`

**How time is calculated**

1. Start at `issue.fields.created` with the issue’s current status at that time.
1. For each changelog entry where `field == "status"`, close the current interval at the change timestamp and open a new one with the new status.
1. The last interval ends at:
    - **now (UTC)** if `cutoff_now = true`, or
    - `resolutiondate` (if present) when `cutoff_now = false`.
1. Negative/invalid spans are skipped.
1. Per‑issue totals are **sum of all intervals** for each status (i.e., repeats add up).

**Outputs (main exporter)**

> The exporter writes **hours**. The post‑processor converts to **days** and remaps labels.

1. **Per‑issue totals CSV** — e.g. `metrics_total.csv`
1. Columns:
    - `issue_key`: Jira key (e.g., `APIM-123`)
    - `summary`: current issue summary
    - `project`: project key (e.g., `APIM`)
    - One column per **raw** status in your `statuses` list: `"<Status>__total_hours"` (e.g., `In Progress__total_hours`)
    - Additional statuses not in your `statuses` list are emitted as `OTHER[<Status>]__total_hours`
1. **Intervals CSV** (optional) — e.g. `intervals.csv`
1. Columns (one row per contiguous time-in-status segment):
    - `issue_key`, `summary`, `project`
    - `status`: raw Jira status name at the time
    - `start`, `end`: ISO8601 UTC timestamps
    - `duration_hours`: continuous duration of that segment in hours
1. **Aggregates CSV** (when `aggregate = true`) — e.g. `aggregates.csv`
1. Rows represent either:
    - **board** scope (all issues), or
    - **project** scope (one row per project).
1. Columns:
    - `scope`: `"board"` or `"project"`
    - `project`: blank for board-level rows, project key otherwise
    - `status`: raw status name (as exported)
    - `count_issues`: number of issues with **non‑zero** time in this status
    - `sum_hours`: sum of totals across those issues (hours)
    - `mean_hours`: average time (hours)
    - `p50`, `p75`, `p85`, `p90`, `p95`: percentiles of per‑issue totals (hours)
    - _A percentile is the value below which x% of observations fall. E.g., `p90 = 12.0h` means 90% of issues spent ≤12h in that status._

> Note: Percentiles are computed on per‑issue totals (not on intervals).

**Assumptions & caveats**

- Times are in **UTC**; we don’t do business-hour calendars.
- “Time in status” is built from **changelogs**; if admins bulk-move or edit history, that affects results.
- If your board renames statuses later, intervals will reflect whatever names existed at each point in time.

---

### 2) Post‑processor — `postprocess_jira_csvs.py`

**Purpose**

- Keep the exporter simple and generic; do all presentation/cleanup here.
- Make CSVs analysis‑ready for the team.

**What it does**

***Map raw Jira → display labels*** (and do it idempotently):
- Mapping table we use:

| Raw Jira | Display |
| --- | --- |
| To Do | **Not Started** |
| In Progress | **Started** |
| In Code Review | **Started** |
| Ready for Test | **To Do** |
| In Testing | **In Progress** |
| Doc Review | **Peer Review** |
| Done | **Done** |

- Display column order we enforce everywhere: `Not Started, Started, To Do, In Progress, Peer Review, Done`

***Merge collisions safely***
- If multiple raw statuses map to the same display label (e.g., _In Progress_ and _In Code Review_ → **Started**), the script **sums** their values into the same display column.

***Convert hours → days (24h)***
- Converts only when the source column is `__total_hours` (or `duration_hours`, `sum_hours`, `mean_hours`), so you can re‑run the post‑processor and never double‑divide.
- Outputs new files with a `_days.csv` suffix.
  
***Guarantee all six display columns*** in the per‑issue CSV
- Even if the raw export happened to omit a column (e.g., no issue had time in “To Do”), the post‑processor **creates** it and fills with `0.00`, so your sheets/charts can rely on a stable schema.
  
***Filter `Done` rows by `Resolution` = “Done” (best‑effort)***
- If a CSV has a `Resolution` column, the post‑processor **keeps all non‑Done** rows and only keeps **Done** rows whose `Resolution` is exactly “Done” (case‑insensitive).
- If a CSV **doesn’t** include a `Resolution` column, we can’t enforce this automatically for that file type (limitation of the source). See “Strong filtering options” below.

**Outputs (post‑processor)**

- For each `*.csv` in the folder, it writes a `*_days.csv` neighbor with:
    - **labels mapped and ordered** as above,
    - **collisions merged**,
    - **hours converted to days**,
    - **(if present)** Done filtered by Resolution.

**Strong filtering options (optional)**

- If you require strict filtering everywhere, you have two choices:
    1. **Add `Resolution` to all source CSVs** in the exporter and re‑run; or
    1. **Constrain JQL** to emulate “keep all non‑Done, and only Done with Resolution=Done” up front, e.g.:

```
(status != Done) OR (status = Done AND resolution = Done)
```

- Then the exported CSVs already respect the rule.

---

## File glossary

### Per‑issue totals — `metrics_total.csv` (main) → `metrics_total_days.csv` (post‑processed)

- `issue_key` — Jira key
- `summary` — current issue summary
- `project` — project key
- **Status total columns** (main): `"<Raw Status>__total_hours"`
- **Status total columns** (post‑processed): `"<Display Status>__total_days"`
- Examples after post‑processing:
- `Not Started__total_days, Started__total_days, To Do__total_days, In Progress__total_days, Peer Review__total_days, Done__total_days`
- There may also be `OTHER[<Status>]__total_days` for statuses outside our mapping.

> Semantics: each cell is the **sum of all time (days)** that issue spent in that status across the entire lifecycle (repeats included).

### Intervals — `intervals.csv` (main) → `intervals_days.csv` (post‑processed)

- `issue_key`, `summary`, `project`
- `status` — (post‑processed) display label per mapping
- `start`, `end` — ISO timestamps (UTC)
- `duration_days` — continuous span for this interval, in days
- _(Rows are chronological segments; totals are not pre‑summed here.)_

### Aggregates — `aggregates.csv` (main) → `aggregates_days.csv` (post‑processed)

- `scope` — `"board"` or `"project"`
- `project` — blank for board rows
- `status` — (post‑processed) display label
- `count_issues` — number of issues with **non‑zero** time in this status
- `sum_days` — sum of per‑issue totals (days)
- `mean_days` — average per‑issue total (days)
- `p50`, `p75`, `p85`, `p90`, `p95` — percentiles of per‑issue totals (days)

> Example: A `p90` of `3.50` for **Peer Review** at project scope means **90%** of that project’s tickets spent **≤ 3.5 days** in Peer Review (total across repeats).

---

## Usage

1. **Configure** `jira_metrics.toml` next to the exporter (example keys):

```
base_url = "https://your-org.atlassian.net"
email    = "you@your-org.com"
api_token = "<Jira API token>"
jql = 'project in (...) AND issuetype = ... AND ... ORDER BY created DESC'

statuses = [
  "To Do", "In Progress", "In Code Review", "Ready for Test",
  "In Testing", "Doc Review", "Done"
]

cutoff_now = true
pagesize   = 100
sleep      = 0.0

out           = "metrics_total.csv"
intervals_out = "intervals.csv"
aggregate     = true
agg_out       = "aggregates.csv"
percentiles   = [50, 75, 85, 90, 95]
```

2. **Run exporter**

```bash
python jira_time_in_status_total.py
```

3. **Run post‑processor** in the same folder

```bash
python postprocess_jira_csvs.py
```

You’ll see `*_days.csv` files appear for each source CSV.

4. **Analyze / share**
   
Import the `*_days.csv` into Sheets/Excel/BI. Build pivot charts by **project** and **status** using `sum_days`, `mean_days`, and the percentile columns.

---

## Design choices & “gotchas” we handled

- **Back‑and‑forth moves are common** → we **sum every visit** to a status (rather than first‑pass only).
- **Label changes & collisions** → post‑processor maps raw Jira → display, **merges** colliding columns, and enforces **stable column ordering**.
- **Idempotent post‑processing** → you can re‑run safely; no double conversion and no double‑mapping.
- **Filtering Done by Resolution** → applied when a `Resolution` column exists; otherwise use stricter JQL or enhance the exporter to include Resolution everywhere.
- **Time unit** → “days” means **24 hours** (not business days).
- **Rate limiting** → the exporter respects HTTP 429 and retries; tune `sleep` if needed.
- **Timezone** → all timestamps are treated as UTC.

---

## Troubleshooting

- **HTTP 410 Gone** on `/rest/api/3/search`: use the **enhanced** endpoint `/rest/api/3/search/jql` (the script already does).
- **HTTP 400 Bad Request**: usually malformed JQL; test the JQL in the Jira UI.
- **Missing “Not Started” column in per‑issue**: post‑processor now **forces** all six display columns; if you still don’t see it, ensure you ran the post‑processor on the correct CSV.
- **Numbers look too small by 24×**: likely the source was already in days; the post‑processor only converts when the original header was `__total_hours`. Re‑run on the original `hours` CSVs.
- **“Done” items that aren’t really done**: include a `Resolution` column in your source CSV or narrow JQL as shown above.

---

## Security

- Store the **API token** outside source control (e.g., in `jira_metrics.toml` not committed, or use env vars).
- Don’t paste tokens in chat or screenshots. Rotate if exposed.

---

## Appendix — Percentiles (quick explainer)

A percentile answers: “**What time covers X% of issues?**”

If the **p90** for **Started** is **2.1 days**, then 90% of issues spent **≤ 2.1 days** in **Started** (total across repeats). This is great for setting expectations: “Most tickets clear Started in ~2 days; the slowest 10% take longer.”

### Detailed Explanation

The data (and therefore these plots) are based on **time spent in each Jira status**, not _when_ issues entered or exited those statuses.

- Each issue has a duration value = total number of days it spent _in a status_ (not when it entered or left).
- When we compute percentiles (p50, p75, p95, etc.), we’re summarizing **how long tickets typically stayed** in that status before moving on.

**Example:** Let’s say the p95 for Peer Review = **58 days**. That means 95% of all tickets that ever entered _Peer Review_ stayed there for **58 days or fewer**, and 5% stayed even longer. It **does not** mean that 95% of tickets entered Peer Review by day 58. It means they **spent up to 58 days inside that column** before moving to the next one (like “Done”).

### Why The Distinction Matters

- These durations measure **status aging** — how long tickets sit in each workflow step.
- It tells you where bottlenecks are (e.g., if Peer Review durations are long, reviews are slow).
- It does **not** tell you how long it takes before something _reaches_ a status for the first time.

### TL;DR Summary

| **Term** | **What It Means** | **In Your Graph** |
| --- | --- | --- |
| **p50** | 50% of tickets spent _≤ this many days_ in the status | Typical / median cycle time for that step |
| **p95** | 95% of tickets spent _≤ this many days_ in the status | Outliers beyond this are very slow tickets |
