---
name: annotate-findings
description: Read an executed Jupyter notebook and rewrite placeholder observation cells (marked `<!-- annotate-findings: pending -->`) with real findings derived from the actual cell outputs. Closes the loop after /execute-notebook so observations reference real numbers, not pre-execution speculation.
argument-hint: "[notebook-path]"
allowed-tools: Read Write Bash
---

# Skill: Annotate Findings

Take the notebook at `$ARGUMENTS` (which must already have been executed) and replace every placeholder observation cell with real findings derived from the executed outputs of the surrounding code cells.

This skill closes the loop opened by `/analyze-dataset` (which writes placeholders) and `/execute-notebook` (which produces real outputs). It eliminates the most common failure of generated notebooks: markdown that claims results that the code never actually proved.

## Principle

A finding is only valid if it traces back to an executed output. This skill **never invents numbers**. If the preceding code cell has no output, or the output cannot be interpreted (binary blobs, image data only), the placeholder gets a clear note saying so — never fabrication.

## Step 1 — Validate input

1. Verify the notebook file exists.
2. Verify it has been executed: at least one code cell must have non-empty `outputs` and a non-null `execution_count`. If the notebook looks pristine (no executions), fail with: `Notebook does not appear to have been executed. Run /execute-notebook first.`
3. Locate all markdown cells containing the literal marker `<!-- annotate-findings: pending -->`. If none are found, report `No placeholder cells found — nothing to annotate.` and exit cleanly.

## Step 2 — For each placeholder, gather context

For each placeholder markdown cell at index `i`:

1. Extract the **"Qué buscar" bullets** from the placeholder (they list what to inspect — use them as the question prompt).
2. Walk **backwards** from cell `i` collecting code cells until you hit either a section header (`## N.`) or another placeholder. These are the code cells whose outputs feed this observation.
3. From each gathered code cell, extract:
   - The cell's `source` (so the skill can read what was computed).
   - All `outputs` of types `stream` (stdout/stderr text), `execute_result` (text/plain representation of last expression), and `display_data` (descriptions of figures — read the `text/plain` fallback if present, otherwise note "figure produced").
4. Concatenate sources + outputs into a context block per placeholder.

## Step 3 — Generate real findings

For each placeholder, write a new markdown cell body in this structure:

```markdown
### Observaciones

- <specific finding 1 with concrete number from output>
- <specific finding 2 with concrete number>
- <specific finding 3>
- <if applicable: a brief interpretation tying findings to the original "qué buscar" prompts>
```

Strict rules for the new content:
- Every quantitative claim must come from the output. If the output prints `chi² = 12.34, p = 0.0004`, you can write *"chi² = 12.34 (p = 0.0004), suficientemente bajo para rechazar H0 con α = 0.05"*. You **cannot** invent values.
- For figures (no numeric output), describe what the chart's structure implies based on the *code* that generated it (e.g. "the heatmap shows goal counts by season × competition; the cell with `pivot_table` aggregating Season × Competition was used"). If the chart's content cannot be inferred from outputs alone, write *"(figura generada — inspeccionar visualmente para conclusiones específicas)"*. Honest acknowledgement.
- For hypothesis test results: report observed statistic + p-value + decision (reject/fail to reject H0 at α = 0.05) + a one-line plain-language interpretation.
- Keep the language consistent with the rest of the notebook (Spanish narrative, English code identifiers).
- 3–5 bullets typical, max 7. Concise.

## Step 4 — Replace and write back

1. For each placeholder, replace its `source` array with the newly-generated body. **Remove the `<!-- annotate-findings: pending -->` marker** (the cell is now complete).
2. Preserve cell IDs and metadata.
3. Write the notebook back to the same path (overwrite). Do not create a new file.

## Step 5 — Report to the user

Print exactly one summary line:

```
Annotated <K> placeholder cells in <path>. Inspect to verify findings match expectations.
```

If any placeholder could not be filled (e.g., no preceding executed code cells found), include:
```
WARNING: <N> placeholders could not be annotated (no executed context found):
  - cell <index>: <reason>
```

Do not print the new content into the conversation — the user opens the notebook to review.

## Constraints

- All Python helper code uses type hints + NumPy-style docstrings.
- Do NOT modify code cells. Only markdown placeholders.
- Do NOT insert new cells. Only rewrite existing ones.
- If a placeholder lacks any preceding executed code (e.g. it's right after a section header with no code yet), leave it as-is and flag it in the WARNING block.
- Idempotency: running this skill twice on the same notebook should be a no-op the second time (no markers left to update).

## Out of scope

- This skill does NOT execute code. If the notebook hasn't been executed, it fails fast.
- This skill does NOT reorder cells, change titles, or touch code logic.
- This skill does NOT verify that the executed outputs are statistically valid — it just transcribes them honestly into prose.
