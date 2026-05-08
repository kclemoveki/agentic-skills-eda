# Example Sections for Analytical Notebooks

These are examples of well-written notebook sections to use as reference for tone, depth, and style.

---

## Example: Good Markdown Narrative Cell (after EDA)

```markdown
### Observations

- The dataset spans 17 seasons (2004-2021) with 672 recorded goals.
- The `minute` column ranges from 1 to 93, confirming it includes stoppage time goals.
- `competition` has 12 unique values, but the top 3 (La Liga, Champions League, Copa del Rey) account for 89% of all goals.
- There are no missing values in critical columns (`date`, `minute`, `opponent`), but `assist` is null for 15% of records — likely solo goals or penalties.
- The `date` column is a string — we'll parse it to datetime for temporal analysis.
```

**Why this is good:** It's specific, quantitative, and ends with an actionable decision.

---

## Example: Good Hypothesis Cell

```markdown
### Hypothesis 2: Messi scores more frequently in the second half than the first

**Rationale:** Visual inspection of the minute distribution shows a right skew, suggesting
more goals after the 45th minute. This could indicate that opponents tire defensively,
or that Messi's game reading improves as matches develop.

**Test:** Chi-squared goodness-of-fit comparing observed first-half vs second-half counts
against the null hypothesis of equal distribution.
```

```python
def test_half_distribution(
    df: pd.DataFrame,
    minute_col: str = "minute",
    split_minute: int = 45,
) -> dict[str, float]:
    """Test whether goals are equally distributed between first and second half.

    Parameters
    ----------
    df : pd.DataFrame
        Goals dataframe.
    minute_col : str
        Column containing the match minute.
    split_minute : int
        Minute that separates first from second half.

    Returns
    -------
    dict[str, float]
        Dictionary with chi2 statistic and p-value.
    """
    first_half = (df[minute_col] <= split_minute).sum()
    second_half = (df[minute_col] > split_minute).sum()

    chi2, p_value = stats.chisquare([first_half, second_half])

    print(f"First half goals:  {first_half}")
    print(f"Second half goals: {second_half}")
    print(f"Chi-squared: {chi2:.2f}, p-value: {p_value:.4f}")

    if p_value < 0.05:
        print("→ Reject H0: The distribution is significantly unequal.")
    else:
        print("→ Fail to reject H0: No significant difference between halves.")

    return {"chi2": chi2, "p_value": p_value}


results = test_half_distribution(df)
```

```markdown
**Result:** With a chi-squared value of 18.73 (p < 0.001), we reject the null hypothesis.
Messi scores significantly more in the second half. This aligns with the theory that
his positional intelligence becomes more dangerous as defenses lose shape late in matches.
```

**Why this is good:** States the hypothesis, explains rationale, shows the test, and interprets in context.

---

## Example: Good Advanced Visualization Cell

```python
def create_season_minute_heatmap(
    df: pd.DataFrame,
    season_col: str = "season",
    minute_col: str = "minute",
    bin_size: int = 15,
) -> plt.Figure:
    """Create a heatmap of goals by season and match minute bin.

    Parameters
    ----------
    df : pd.DataFrame
        Goals dataframe.
    season_col : str
        Column containing the season identifier.
    minute_col : str
        Column containing the match minute.
    bin_size : int
        Size of minute bins (e.g., 15 for 0-15, 15-30, etc.).

    Returns
    -------
    plt.Figure
        The matplotlib figure containing the heatmap.
    """
    df_temp = df.copy()
    df_temp["minute_bin"] = pd.cut(
        df_temp[minute_col],
        bins=range(0, 96, bin_size),
        labels=[f"{i}-{i+bin_size}" for i in range(0, 90, bin_size)],
    )

    pivot = df_temp.pivot_table(
        index=season_col,
        columns="minute_bin",
        aggfunc="size",
        fill_value=0,
    )

    fig, ax = plt.subplots(figsize=(14, 8))
    sns.heatmap(
        pivot,
        annot=True,
        fmt="d",
        cmap="YlOrRd",
        linewidths=0.5,
        ax=ax,
    )
    ax.set_title("Goals by Season and Match Minute", fontsize=14, fontweight="bold")
    ax.set_xlabel("Match Minute (binned)")
    ax.set_ylabel("Season")
    plt.tight_layout()
    return fig


fig = create_season_minute_heatmap(df)
plt.show()
```

**Why this is good:** Typed function with docstring, clear variable names, proper figure sizing, annotated heatmap, and tight_layout.

---

## Example: Good Model Interpretation Cell

```markdown
### Interpretation

The Random Forest classifier achieves 71% accuracy predicting whether Messi will score
2+ goals in a match (hat-trick potential). The top 3 features by importance are:

1. **opponent_rank** (0.34) — Weaker opponents significantly increase multi-goal probability
2. **is_home** (0.21) — Home advantage contributes to higher-scoring matches
3. **competition_encoded** (0.18) — La Liga matches show higher multi-goal frequency than Champions League

**Caveats:**
- This is a small dataset with class imbalance (only 12% of matches have 2+ goals)
- The model is exploratory and NOT suitable for prediction — it merely quantifies feature relationships
- Cross-validation on 5 folds shows high variance (accuracy range: 0.63–0.78)
```

**Why this is good:** Leads with the result, ranks features with numbers, and explicitly states caveats.

---

## Anti-Patterns to Avoid

**Bad markdown cell:**
```markdown
Now let's make a bar chart.
```
→ Says nothing about WHY or WHAT we expect to find.

**Bad code cell (no types, no docstring):**
```python
def plot(data, col):
    plt.figure()
    data[col].hist()
    plt.show()
```
→ Missing type hints, docstring, figure size, title, axis labels, and return type.

**Bad conclusion:**
```markdown
## Conclusion
The analysis is complete.
```
→ Says nothing. Summarize actual findings.
