---
name: analyze-dataset
description: Generate a structured analytical Jupyter notebook from any dataset path. Produces EDA, statistical tests, visualizations, and optional modeling following notebook best practices.
argument-hint: "[dataset-path]"
allowed-tools: Read Write Bash Edit Glob Grep
---

# Skill: Analyze Dataset

Generate a professional, reproducible Jupyter notebook that analyzes the dataset at `$ARGUMENTS`.

## Step 0 — Look for sibling artifacts (composition with other skills)

Before reading the dataset, check the current working directory and the dataset's directory for artifacts produced by other skills in the suite. These are **inputs that improve this skill's output** when present:

- **`quality_report_<dataset_stem>.md`** (produced by `/quality-report`): if it exists, read it. Use its `Bloqueantes`, `Recomendaciones de limpieza`, and dimension scores as authoritative guidance for the cleaning steps in Section 2 of the notebook. The report has already inspected the data — leverage that work. Cite findings from the report in the notebook's Section 2 markdown (e.g. *"según el reporte de calidad, `Date` requiere parseo a datetime"*).
- **`<dataset>.manifest.yaml`** (produced by `/snapshot-data`): if it exists, read it. Verify that the dataset's current SHA-256 matches the manifest. If they differ, print a clear notice in the notebook's Section 1: *"⚠️ Dataset has changed since manifest was created (sha256 mismatch). Analysis is on the new version."* If they match, mention the manifest as proof of reproducibility.

If neither exists, proceed normally — these are optional enhancers, not requirements.

## Step 1 — Understand the data

Before writing any notebook cell:
1. Read the file to understand format (CSV, Parquet, JSON, Excel)
2. Identify columns, types, and cardinality
3. Determine the nature of the data: temporal, categorical, numerical, mixed
4. **Probe edge cases for any column you plan to parse**: run `df[col].unique()` (or `df[col].value_counts().head(50)` if cardinality is high) for every string column you intend to convert (dates, scores, mangled numerics, etc.). Identify ALL distinct patterns. Never trust `df.head()` alone — the first 5 rows are not representative.
5. Decide which analyses make sense for THIS specific dataset. If a quality report from Step 0 is available, prioritize the cleaning issues it flagged.

## Step 2 — Create the notebook

Create a `.ipynb` notebook file. Name it: `analysis_<dataset_name>.ipynb`

Follow this structure strictly. Each section is a group of cells. Always alternate between **markdown cells** (narrative) and **code cells** (analysis). The markdown tells the story; the code provides evidence.

### Section 1: Setup & Data Loading (3-4 cells)
- Markdown: title, objective, dataset description
- Code: imports (pandas, numpy, matplotlib, seaborn, scipy, sklearn as needed)
- Code: load dataset, display shape, dtypes, first rows
- Markdown: initial observations about the data

### Section 2: Data Quality Assessment (3-4 cells)
- Markdown: explain what we're checking and why
- Code: missing values (count + percentage), duplicates, unique values per column
- Code: visualize missing patterns if relevant (heatmap or bar)
- Markdown: summary of data quality findings, cleaning decisions

### Section 3: Univariate Analysis (4-6 cells)
- Markdown: explain approach
- For numerical columns: histograms + KDE, box plots, descriptive stats
- For categorical columns: value counts, bar charts
- For temporal columns: time series line plots
- **Placeholder observations cell** (see "Observation Cells" rules below) — do not invent findings
- Use varied chart types: histogram, boxplot, bar chart, countplot

### Section 4: Bivariate & Multivariate Analysis (4-6 cells)
- Markdown: what relationships we're exploring and why
- Correlation heatmap for numerical features
- Scatter plots for interesting pairs
- Grouped bar charts or box plots for categorical vs numerical
- Cross-tabulation or heatmap for categorical vs categorical (if relevant)
- **Placeholder observations cell**

### Section 5: Temporal Analysis (3-4 cells, if applicable)
- Only include if the dataset has date/time columns
- Markdown: temporal questions we're investigating
- Time series trends (line charts)
- Rolling averages to show trends
- Seasonal or periodic patterns (heatmap month×year, day-of-week radar, etc.)
- **Placeholder observations cell**

### Section 6: Hypothesis Testing (3-4 cells)
- Markdown: formulate 2-3 hypotheses based on observations from previous sections (the *hypotheses themselves* are pre-execution — that's fine, they describe what we're testing)
- Code: for each hypothesis, run an appropriate statistical test:
  - Categorical vs categorical: chi-squared test
  - Numerical between 2 groups: t-test or Mann-Whitney
  - Numerical between 3+ groups: ANOVA or Kruskal-Wallis
  - Correlation significance: Pearson/Spearman with p-value
- Visualize each test with the appropriate plot (box plot, violin plot, grouped bar)
- **Placeholder result cell** for each hypothesis — do not pre-write the test conclusion (e.g. *"rejected H0 with p<0.001"*) since the actual p-value comes from execution

### Section 7: Advanced Visualizations (3-4 cells)
- Markdown: explain what we want to reveal
- Choose 2-3 from this list based on what fits the data:
  - Heatmap (pivot table of two dimensions)
  - Treemap or sunburst (hierarchical data)
  - Radar/polar chart (multi-dimensional comparison)
  - Network graph (relationships between entities)
  - Cumulative line chart (accumulation over time)
  - Violin plot (distribution comparison)
  - Stacked bar chart (composition over time/categories)
  - Pair plot (multi-feature relationships)
- Markdown: what these visualizations reveal that simpler charts didn't

### Section 8: Lightweight Modeling (3-4 cells, optional but recommended)
- Markdown: define a simple predictive or clustering question
- Choose ONE based on what fits:
  - Classification: logistic regression or random forest (if there's a natural target)
  - Regression: linear regression (if there's a continuous target)
  - Clustering: K-Means + PCA visualization (if no clear target)
- Show feature importance or cluster characteristics
- **Placeholder interpretation cell** — caveats about the exploratory nature ARE pre-writable; specific results (accuracy, importance values) are not

### Section 9: Conclusions & Next Steps (1-2 cells)
- **Placeholder cell** — top findings depend on real outputs. Limitations and suggested follow-up analyses CAN be pre-written (they are about scope, not results).

## Visualization Rules

- Use `matplotlib` + `seaborn` as primary. `plotly` only if interactivity adds clear value.
- Set a consistent style at the start: `sns.set_theme(style="whitegrid", palette="husl")`
- Every plot MUST have: title, axis labels, and readable font sizes
- Use `fig, ax = plt.subplots(figsize=(W, H))` — never rely on default tiny sizes
- Prefer `figsize=(12, 6)` for wide charts, `(10, 8)` for square/heatmaps, `(8, 8)` for radar/network
- Add annotations on charts where they add insight (e.g., mark max/min values, add value labels on bars)
- Use color meaningfully: sequential for ordered data, categorical for groups, diverging for +/- values
- When showing many categories (>10), show only top N and group the rest as "Other"

## Narrative Rules

- Write markdown cells in **Spanish** (the user's language)
- Every markdown cell should explain the WHY, not just the WHAT
- Bad: "A continuación se muestra un histograma"
- Good: "Analizamos la distribución de X para detectar si hay sesgo o valores atípicos que afecten el análisis"
- After each analysis block, include an "Observaciones" markdown cell with bullet points of findings
- Use headers (##, ###) to create a clear hierarchy
- The notebook should read as a coherent story from top to bottom

## Code Rules

- Keep cells short: one concept per cell, max 30 lines
- No repeated code — if a pattern repeats 3+ times, define a helper function in an early cell
- Handle warnings: `import warnings; warnings.filterwarnings('ignore')` in setup
- Use `display()` instead of bare expressions when showing multiple DataFrames
- Parse dates automatically when loading: `parse_dates` parameter
- All text in code (variable names, comments) in English; all markdown narrative in Spanish
- **ALL functions MUST have type hints** on every parameter and return value (use `X | None` not `Optional[X]`)
- **ALL functions MUST have NumPy-style docstrings** explaining purpose, parameters, and return values
- See `templates/notebook-structure.md` and `examples/example-sections.md` for typed+documented function examples

## Observation Cells — pre-execution honesty

The notebook is generated **before any code cell has been executed**. That means specific numerical findings (means, p-values, correlations, accuracy scores) are unknown at write time. Do NOT fabricate them.

For every "Observaciones" / "Resultados" / "Interpretación" markdown cell that depends on output, write a **placeholder** in this exact format:

```markdown
### Observaciones

<!-- annotate-findings: pending -->

> Esta sección se completará tras la ejecución del notebook. Una vez que las celdas de código hayan corrido, ejecutá `/annotate-findings` para que esta celda se reescriba con hallazgos reales basados en los outputs.

**Qué buscar al ejecutar:**
- <bullet describing what to look at, e.g. "shape of the minute distribution">
- <bullet>
- <bullet>
```

The key elements:
- The HTML comment `<!-- annotate-findings: pending -->` is a **machine-readable marker** that `/annotate-findings` uses to locate cells that need updating. Always include it verbatim.
- The "Qué buscar" bullets ARE pre-writable — they describe *what to look for*, not *what was found*. They become the reading guide for the human or the prompt for `/annotate-findings`.
- Do NOT include any specific number, percentage, p-value, correlation coefficient, or directional claim ("X is higher than Y") in placeholder cells.

What IS legitimately pre-writable:
- Section titles and structural markdown.
- "Why we're doing this" framing.
- Hypothesis statements (they describe what's being tested, not the result).
- Methodology explanations.
- Limitations and follow-up suggestions in Section 9 (those are about scope, not findings).

If you find yourself wanting to write *"el chi² rechaza la hipótesis nula con p<0.001"*, stop — you don't know that yet. Write a placeholder.

## Parsing Rules

These rules are **mandatory** for any function that parses string columns into structured types (dates, scores, numerics with units, etc.). They prevent the most common failure mode of generated notebooks: a parser that worked on `df.head()` but blows up on edge cases buried in the data.

- **Probe before parsing**: every parser function must be written *after* you have seen all distinct patterns of the input column (Step 1 #4). The function must handle every observed pattern, OR explicitly fall back to `None` / `NaN` for unrecognized ones.
- **Defensive conversions**: every type conversion (`int(...)`, `float(...)`, `pd.Timestamp(...)`) inside a parser must be wrapped in `try/except`. On failure, return `None` / `NaN`. **Never raise** — a single bad row must not abort the analysis. Log the offending value with `warnings.warn(...)` so the user can audit later.
- **Validate after parsing**: after applying a parser to a column, print the count of failures (`NaN` / `None` results introduced by parsing). If more than 10% of values failed, print a `WARNING: parser <name> failed on X% of rows in <column> — review patterns in df['<column>'].unique() and extend the parser.` This is non-fatal but visible.
- **Document handled patterns**: the parser docstring must list which input patterns it recognizes (e.g. `"Handles 'X:Y' (clean), 'X:Y AET' (after extra time), 'X:Y ap' (after penalties)"`). A reader of the notebook should know exactly what the parser can and cannot do.

## Output

Save the notebook to the current working directory as `analysis_<dataset_name>.ipynb`.
After creating it, report:
- Number of cells created (code vs markdown)
- Types of visualizations included
- Hypotheses tested
- Whether modeling was included and why/why not

Read the template at `templates/notebook-structure.md` for the cell-by-cell skeleton.
Read `examples/example-sections.md` for examples of well-written narrative cells.
