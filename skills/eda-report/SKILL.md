---
name: eda-report
description: Synthesize an executed and annotated EDA notebook into a 1-3 page executive markdown report. Selects 3-6 key findings, removes code, structures into 7 canonical sections, and produces a stakeholder-ready document. Closes the EDA pipeline after /annotate-findings.
argument-hint: "[notebook-path]"
allowed-tools: Read Write Bash
---

# Skill: EDA Report

Take the executed and annotated notebook at `$ARGUMENTS` and produce an executive markdown report at `reports/eda_report_<dataset_stem>.md`. The report is for **non-technical stakeholders** — managers, product owners, business leads — who need the conclusions without reading 50+ cells.

This skill closes the loop opened by `/analyze-dataset` (structure with placeholders), `/execute-notebook` (run the code) and `/annotate-findings` (write real observations). If the report comes before any of those three, fail with a clear hint.

## Principle

A report is **synthesis, not transcription**. The notebook may have 30 observations across 9 sections; the report has 3-6 findings. The skill must **select**, **rank**, and **rewrite** — never just copy markdown cells verbatim.

Two non-negotiable rules:
1. **Every quantitative claim must trace back to a real cell output** in the notebook. Never invent numbers (same rule as `/annotate-findings`).
2. **Zero code in the report body**. Code lives in the notebook, which is linked from the Appendix.

## Step 1 — Validate input

1. Verify the notebook file exists and is readable.
2. Verify it has been **executed**: at least one code cell with non-empty `outputs` and a non-null `execution_count`. If pristine, fail with: `Notebook not executed. Run /execute-notebook first.`
3. Verify it has been **annotated**: NO markdown cell should contain the literal marker `<!-- annotate-findings: pending -->`. If markers remain, fail with: `Notebook has unfilled placeholders. Run /annotate-findings first.`

## Step 2 — Extract content from the notebook

**First, ensure the output directory tree exists.** Create `reports/` and `reports/figures/` (relative to the current working directory) if they do not already exist. Use `mkdir -p reports/figures` via `Bash` or `Path("reports/figures").mkdir(parents=True, exist_ok=True)` in the Python script. This is a hard prerequisite for the rest of Step 2 — without these directories, the image extraction below fails silently.

Then run a Python one-shot via `Bash` that:

1. Loads the `.ipynb` as JSON.
2. For each markdown cell, capture its content and the index of the immediately preceding section header (`## N. Title`).
3. For each code cell, capture:
   - The cell's `source` (read-only — won't be reproduced in the report, but used to understand what generated each output).
   - All `outputs` of types `stream`, `execute_result`, `display_data`. For `display_data` containing `image/png`, decode the base64 payload and save the image to `reports/figures/<dataset_stem>_cell_<index>.png`. Remember the path for cross-referencing in Step 4.
4. Capture sibling artifacts if present in the notebook's directory:
   - `<dataset>.manifest.yaml` — extract `sha256` (first 12 hex), `rows`, `cols`, `generated.at`.
   - `quality_report_<dataset>.md` — extract overall light (🟢🟡🔴) and top blockers.

After Step 2 completes, verify that `reports/figures/` is non-empty if the notebook had image outputs. If the notebook had images but no files were written, fail loudly — do not proceed to Step 3 with broken figure references.

## Step 3 — Map notebook content to the 7 report sections

The notebook has up to 9 sections; the report has 7. Apply this mapping:

| Notebook section | Report section |
|---|---|
| 1 (Setup) | _skipped_ — too technical |
| 2 (Data Quality) | 3 (Data Overview) — synthesize: rows, cols, missing %, key cleaning decisions |
| 3-7 (Univariate, Bivariate, Temporal, Hypothesis, Viz) | 4 (Key Findings) — **select** 3-6 of the most important observations |
| 8 (Modeling, if present) | 4 (Key Findings) — at most 1 finding referencing the model |
| 9 (Conclusions) | seeds 1 (TL;DR) and 6 (Recommendations) |

The hard part is **selection**. From all annotated observations across sections 3-7, prioritize findings that satisfy AT LEAST ONE:

- A statistical test was significant (p < 0.05) **and** effect size is non-trivial (|r| > 0.3, Cramer's V > 0.15, etc.).
- The magnitude is striking: top-1 dominates >40% of mass, a category accounts for >20%, a temporal segment shows clear shift.
- The finding has a direct, actionable implication for the dataset's likely use case.
- The finding contradicts a common-sense prior (surprise value).

Findings that are inconclusive or sample-size-limited go into section 5 (Limitations), NOT into Key Findings.

## Step 4 — Write the report

Read the literal report skeleton from `templates/report-skeleton.md` (it contains the 7-section structure with placeholders for both Spanish and English variants). Adapt it to the notebook's narrative language and fill the placeholders with content gathered in Steps 2 and 3.

Save the result at `reports/eda_report_<dataset_stem>.md`.

The template includes adaptation guidelines for edge cases (no temporal columns, very small N, 🔴 quality report, etc.) — follow them.

## Writing rules (mandatory)

- **BLUF (Bottom Line Up Front)**: the first bullet of section 1 must answer the dataset's apparent business question. The reader shouldn't need to read past page 1 to get the headline.
- **Quantify every claim**: instead of "high" write "23% (IC95% 19-27%)" or "p = 0.003". Effect sizes preferred over p-values alone.
- **One idea per figure**, with declarative captions. Bad: *"Distribución del minuto"*. Good: *"El 60% de los goles ocurren después del minuto 45"*.
- **No code blocks in the body**. Statistical tests get prose: *"Un chi² de 18.7 con p < 0.001 confirma que..."*, not the code that ran it.
- **Calibrated language**: separate confirmed (data shows X), refuted (data argues against Y), inconclusive (data insufficient on Z). Avoid generic hedges like *"podría sugerir tal vez..."*.
- **No invented numbers**: if a finding's number isn't in any cell output, do not include it. Replace with a qualitative observation.
- **Length**: 1-3 pages of markdown. If you exceed 3 pages, you're transcribing instead of synthesizing.
- **Define jargon explicitly**: the report's reader is a stakeholder who may not know domain abbreviations or statistical terms. Two complementary moves:
  - **Inline definition on first use** for any abbreviation, acronym or domain code (e.g., write `"como local (Venue = 'H', Home)"` the first time `Venue` appears, `"CF (centro delantero)"` the first time `CF` appears).
  - **Glossary section at the end** (before the Appendix) listing terms grouped by category: domain abbreviations, statistical tests, dataset columns. Include only terms that actually appear in the report — do not pad.

## Antipatterns to avoid

These are common report failures the skill must NOT produce:

- **Notebook-dump**: pasting `df.describe()` raw output, code cells, debug prints.
- **Hedge stack**: *"podría sugerir que tal vez existiría una posible relación"* — collapse into one calibrated claim.
- **Silent cherry-picking**: presenting only positive results without mentioning the hypotheses that didn't pan out (mention briefly in section 5).
- **Decorative figures**: a chart that doesn't answer a stated question. Rule: if the chart can be removed without weakening the report, remove it.
- **Vague closings**: *"se necesita más análisis"* without specifying what, why, or who. Section 6 must be concrete.

## Step 5 — Report to the user

Print exactly this summary block:

```
Report written to reports/eda_report_<dataset_stem>.md (<K> findings, <F> figures extracted).

Export options (optional):
  PDF:  pandoc reports/eda_report_<dataset_stem>.md -o reports/eda_report_<dataset_stem>.pdf --resource-path=reports
  DOCX: pandoc reports/eda_report_<dataset_stem>.md -o reports/eda_report_<dataset_stem>.docx --resource-path=reports --reference-doc=<SKILL_DIR>/templates/reference.docx
```

Replace `<SKILL_DIR>` with the absolute path of this skill's directory (e.g. `~/.claude/skills/eda-report` or wherever this skill lives in the project's `skills/`). The `--resource-path=reports` flag is critical — without it pandoc cannot find the figures since they're inside `reports/figures/` while the user typically runs the command from the project root.

The DOCX path uses a reference document at `<SKILL_DIR>/templates/reference.docx`. If the file does not exist, omit the DOCX command from the report (still print the PDF one). The reference document defines the visual style — headings, fonts, colors, page layout — that the resulting `.docx` will inherit. Users provide their own `reference.docx` matching their team's branding (e.g., a Google Doc downloaded as `.docx` with the team's heading styles applied).

Uploading the resulting `.docx` to Google Drive and opening with Google Docs preserves >95% of the formatting (headings hierarchy, colors, embedded images). Custom corporate fonts may be substituted by Google Docs if not in its catalog — prefer fonts available in Google Fonts (Roboto, Open Sans, DM Sans, etc.) for best fidelity.

Do not dump the full report into the conversation — the user opens the file.

## Constraints

- Helper Python code uses type hints + NumPy-style docstrings.
- Do NOT modify the source notebook.
- Do NOT generate sections out of order.
- If the notebook lacks content for a section (e.g., no temporal columns → no temporal findings), skip the corresponding subsection in section 4 instead of fabricating one.
- Idempotency: running the skill twice on the same notebook overwrites the report deterministically (same sections, same findings selected, modulo the timestamp in metadata).

## Out of scope (v1)

- **PDF/HTML rendering**: the skill emits markdown only. Pandoc/Quarto conversion is a follow-up step the user runs manually.
- **Custom branding**: the report is plain markdown. If the user wants logos, headers, or a corporate template, post-process with pandoc + a custom LaTeX template (e.g., Eisvogel).
- **Multi-language output**: the report inherits the notebook's narrative language (Spanish if the notebook is Spanish, English otherwise). No translation.
- **Interactive elements**: no Plotly, no widgets. The report is static markdown for stakeholder consumption.

---

Read `templates/report-skeleton.md` for the literal report structure with section placeholders and adaptation guidelines.
