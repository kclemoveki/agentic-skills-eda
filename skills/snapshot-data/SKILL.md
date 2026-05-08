---
name: snapshot-data
description: Generate a versionable YAML manifest next to a dataset capturing SHA-256 hash, schema fingerprint, shape, and column metadata. Used to pin a dataset version so future re-runs can verify they are working on the same data.
argument-hint: "[dataset-path]"
allowed-tools: Read Write Bash Glob
---

# Skill: Snapshot Data

Produce a manifest YAML for the dataset at `$ARGUMENTS` so any future re-run of an analysis can verify that the input data is unchanged. The manifest captures both byte-level identity (file hash) and semantic shape (schema fingerprint).

## Principle

Two hashes are not redundant — they answer different questions:
- **`sha256`** of the file detects any byte-level change, including a benign re-export with different encoding.
- **`schema fingerprint`** detects *semantic* changes. If the file is re-saved with a different encoding the SHA changes but the fingerprint stays — that tells you the *data* is the same even if the bytes are not.

## Step 1 — Locate and validate the dataset

Resolve `$ARGUMENTS` to an absolute path. If the file does not exist, fail with a clear error message. Support these formats by extension: `.csv`, `.parquet`, `.json`, `.jsonl`, `.xlsx`, `.xls`. If the format is unknown, fail with a clear error message.

## Step 2 — Compute the manifest

Run a Python one-shot via `Bash` (use `python3 -c "..."` or write a temp script at `/tmp/snapshot.py`) that computes and prints YAML to stdout. The script must:

1. Compute `sha256` of the file by reading it in chunks (do not load the whole file in memory for large datasets).
2. Capture `os.stat`: `size_bytes`, `modified_at` (ISO-8601, UTC).
3. Load the dataset with the appropriate reader (`pd.read_csv` etc.) and capture:
   - `rows`, `cols`
   - `columns` (list, in original order)
   - `dtypes` per column (as strings, e.g. `"int64"`, `"object"`)
   - `null_counts` per column (only columns with at least 1 null)
4. Compute `dtypes_fingerprint`: SHA-256 (first 16 hex chars) of the canonical string `<rows>|<sorted_columns>|<sorted_dtypes_tuples>` where each pair is `name:dtype`. Sorting is deliberate so column order does not affect the fingerprint — only schema does.
5. Capture generation metadata: `by: snapshot-data`, `at: <ISO-8601 UTC now>`, `format: <inferred from extension>`.

The manifest structure must follow this exact YAML layout:

```yaml
file:
  path: <relative path from cwd>
  size_bytes: <int>
  sha256: <64-hex>
  modified_at: <ISO-8601>
schema:
  rows: <int>
  cols: <int>
  columns:
    - <name>
    - ...
  dtypes:
    <name>: <dtype string>
    ...
  dtypes_fingerprint: <16-hex>
  null_counts:
    <name>: <count>
    ...
generated:
  by: snapshot-data
  at: <ISO-8601 UTC>
  format: <csv|parquet|json|jsonl|xlsx|xls>
```

Use `yaml.safe_dump(..., sort_keys=False)` so the layout matches above. If `pyyaml` is not available, build the YAML string manually with proper indentation — do not silently fall back to JSON.

## Step 3 — Write the manifest

Write the YAML to `<dataset_path>.manifest.yaml` in the same directory as the dataset.

If the manifest already exists:
- If the new `sha256` matches the existing manifest: do not overwrite, report "manifest already up to date".
- If the new `sha256` differs: overwrite, but include in the report: "previous sha256 was `<old>`, dataset has changed".

## Step 4 — Report to the user

Print exactly one line summarizing the result:

```
Manifest written to <path> — sha256:<first 12 hex>... rows:<N> cols:<M>
```

Do not dump the full manifest contents into the conversation — the user opens the file if they want detail.

## Constraints

- All Python code in the inspection script must follow the project's standards: type hints on every function, NumPy-style docstrings.
- Do not modify the original dataset — read-only operation.
- The manifest must be deterministic: same input file → same `sha256` and same `dtypes_fingerprint`. Do not include random seeds, hostnames, or anything non-reproducible (other than the `generated.at` timestamp, which is the only intentionally-varying field).
- Report errors loudly: a corrupt CSV that pandas refuses to parse should fail with a clear message, not produce a manifest with `null` schema.
