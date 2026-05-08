---
name: quality-report
description: Generate a one-page data quality triage report with traffic-light scores for completeness, consistency, and usability. Outputs actionable blockers and cleaning recommendations before full EDA.
argument-hint: "[dataset-path]"
allowed-tools: Read Write Bash Glob
---

# Skill: Quality Report

Produce a concise data quality triage report for the dataset at `$ARGUMENTS`. The report helps a data scientist decide whether the dataset is ready for EDA or needs cleaning first.

## Principle

This skill runs **real Python inspection**, not guesses from `.head()`. Every number in the report must come from an actual computation on the full dataset.

## Step 1 — Inspect the dataset

Run a Python one-shot script via `Bash` that loads the file and prints structured JSON with the metrics below. Use `python3 -c "..."` or write a temp script at `/tmp/qr_inspect.py` and execute it.

The script must compute and print (as JSON to stdout):

```json
{
  "shape": {"rows": N, "cols": M},
  "memory_mb": float,
  "dtypes": {"col": "dtype", ...},
  "missing": {
    "global_pct": float,
    "per_col_pct": {"col": pct, ...},
    "complete_rows_pct": float
  },
  "consistency": {
    "exact_duplicates_pct": float,
    "candidate_keys": [cols_with_unique_ratio_above_99pct],
    "mixed_type_cols": [cols_where_dtype_is_object_but_values_are_heterogeneous]
  },
  "usability": {
    "constant_cols": [cols_with_nunique_eq_1],
    "near_unique_cols": [cols_with_nunique_ratio_above_95pct_that_arent_keys],
    "high_cardinality_cat_cols": [object_cols_with_nunique_above_50]
  }
}
```

Support these formats by extension: `.csv`, `.parquet`, `.json`, `.jsonl`, `.xlsx`, `.xls`. If the format is unknown, fail with a clear error message.

## Step 2 — Classify each dimension

Apply these thresholds to set traffic-light scores:

### Completeness
- 🟢 Green — global missing < 5% AND no column above 50% missing
- 🟡 Yellow — global missing 5-20% OR one column above 50% missing
- 🔴 Red — global missing > 20% OR multiple columns above 50% missing

### Consistency
- 🟢 Green — exact duplicates < 1% AND no mixed-type columns
- 🟡 Yellow — exact duplicates 1-5% OR one mixed-type column
- 🔴 Red — exact duplicates > 5% OR multiple mixed-type columns

### Usability
- 🟢 Green — at least 80% of columns are usable (not constant, not near-unique non-key, reasonable cardinality)
- 🟡 Yellow — 60-80% usable
- 🔴 Red — below 60% usable

### Overall
The overall light is the **worst** of the three dimensions. No averaging — a single red is a red overall, because it signals a blocker.

## Step 3 — Write the report

Create a markdown file named `quality_report_<dataset_stem>.md` in the current working directory. Follow this template exactly:

```markdown
# Quality Report: <dataset filename>

**Generated:** <ISO date>
**Rows × Cols:** <N> × <M>
**Memory:** <X> MB

## 🚦 Overall: <emoji> <one-line verdict>

## Dimensiones

### Completeness: <emoji>
- Global missing: <X>%
- Rows fully populated: <X>%
- Worst columns: <col (X%), col (X%)...>

### Consistency: <emoji>
- Exact duplicates: <X>%
- Candidate keys: <list or "none">
- Mixed-type columns: <list or "none">

### Usability: <emoji>
- Usable columns: <X> / <M>
- Constant columns: <list>
- Near-unique non-keys: <list>
- High-cardinality categoricals: <list>

## Bloqueantes (fix antes de EDA)

<Only list items that are truly blocking — a red light must produce at least one blocker. If no blockers, write "Ninguno.">

- **<issue>**: <concrete description with numbers>. Recomendación: <one-line fix>.

## Recomendaciones de limpieza

<Ranked by impact. 3-5 items max.>

1. <action> — <why it matters>
2. ...

## Stats rápidos

| Columna | dtype | Nulos | Únicos |
|---------|-------|-------|--------|
| ...     | ...   | ...%  | ...    |
```

Keep the report under 60 lines total. If the dataset has many columns, show only the top 10 most relevant in the stats table and mention `(+ X more)`.

## Writing rules

- Write the narrative sections in **Spanish**.
- Keep fixed labels (like `Rows × Cols`, `Memory`) in English — they're technical metadata.
- Every quantitative claim must trace back to the inspection JSON — no invented numbers.
- Be specific in blockers: "La columna `assist` tiene 82% de nulos" beats "hay muchos nulos".
- If the overall light is 🟢, the report should still flag any minor cleaning suggestions — green is not "perfect", it's "ready for EDA".

## Output

After writing the file, report:
- Path of the generated markdown file
- Overall traffic light
- Number of blockers identified

Do not print the full report contents to the conversation — the user will open the file.
