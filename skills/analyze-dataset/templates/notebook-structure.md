# Notebook Structure Template

This is the cell-by-cell skeleton the skill should follow. Each item is one notebook cell.
Adapt sections based on the actual dataset — skip what doesn't apply, expand what does.

## Cell Sequence

```
[MD]  # Analysis of {Dataset Name}
      Brief description of the dataset, source, and analysis objective.

[MD]  ## 1. Setup & Data Loading

[CODE] # Imports and configuration
       import pandas as pd
       import numpy as np
       import matplotlib.pyplot as plt
       import seaborn as sns
       from scipy import stats
       import warnings
       warnings.filterwarnings('ignore')
       sns.set_theme(style="whitegrid", palette="husl")
       plt.rcParams['figure.dpi'] = 100

[CODE] # Load dataset
       df = pd.read_csv("path/to/data.csv")  # adapt format
       print(f"Shape: {df.shape}")
       print(f"Columns: {list(df.columns)}")
       df.head()

[CODE] df.dtypes
       df.describe(include='all')

[MD]  **Initial observations:**
      - The dataset has X rows and Y columns
      - Key columns are: ...
      - Detected temporal/categorical/numerical columns: ...

[MD]  ## 2. Data Quality

[CODE] # Missing values
       missing = df.isnull().sum()
       missing_pct = (missing / len(df) * 100).round(2)
       pd.DataFrame({'missing': missing, 'pct': missing_pct}).query('missing > 0').sort_values('pct', ascending=False)

[CODE] # Duplicates
       print(f"Duplicate rows: {df.duplicated().sum()}")
       print(f"Unique values per column:")
       df.nunique()

[MD]  **Cleaning decisions:**
      - ...

[MD]  ## 3. Univariate Analysis

[MD]  We explore the individual distribution of the most relevant variables.

[CODE] # Numerical distributions — histograms + boxplots
       # (adapt columns to actual data)

[CODE] # Categorical distributions — bar charts
       # (adapt columns to actual data)

[MD]  **Observations:**
      - ...

[MD]  ## 4. Bivariate & Multivariate Analysis

[MD]  We look for relationships between variables that reveal non-obvious patterns.

[CODE] # Correlation heatmap (numerical)

[CODE] # Scatter or grouped analysis

[CODE] # Cross-tabulation or grouped comparison

[MD]  **Observations:**
      - ...

[MD]  ## 5. Temporal Analysis
      (only if date/time columns exist)

[CODE] # Time series trend

[CODE] # Rolling average or seasonal pattern

[CODE] # Heatmap month × year or similar

[MD]  **Observations:**
      - ...

[MD]  ## 6. Hypothesis Testing

[MD]  We formulate hypotheses based on previous observations and validate them statistically.

[MD]  ### Hypothesis 1: ...

[CODE] # Statistical test + visualization

[MD]  **Result:** ...

[MD]  ### Hypothesis 2: ...

[CODE] # Statistical test + visualization

[MD]  **Result:** ...

[MD]  ### Hypothesis 3: ...

[CODE] # Statistical test + visualization

[MD]  **Result:** ...

[MD]  ## 7. Advanced Visualizations

[MD]  We use more complex charts to reveal deeper patterns.

[CODE] # Advanced viz 1 (heatmap pivot, radar, network, etc.)

[CODE] # Advanced viz 2

[CODE] # Advanced viz 3

[MD]  **Observations:**
      - ...

[MD]  ## 8. Exploratory Modeling

[MD]  We apply a simple model to [classify/predict/cluster] ...
      This model is exploratory — not optimized for production.

[CODE] # Feature preparation

[CODE] # Model training + evaluation

[CODE] # Feature importance or cluster visualization

[MD]  **Interpretation:**
      - ...

[MD]  ## 9. Conclusions & Next Steps

[MD]  ### Key Findings
      1. ...
      2. ...
      3. ...
      4. ...
      5. ...

      ### Limitations
      - ...

      ### Suggested Follow-up Analyses
      - ...
```

## Code Standards

- All functions MUST include type hints for all parameters and return values
- All functions MUST include docstrings (NumPy style) explaining purpose, parameters, and return values
- Example:
  ```python
  def plot_distribution(
      df: pd.DataFrame,
      column: str,
      ax: plt.Axes | None = None,
  ) -> plt.Axes:
      """Plot histogram with KDE overlay for a numerical column.

      Parameters
      ----------
      df : pd.DataFrame
          Source dataframe.
      column : str
          Column name to plot.
      ax : plt.Axes | None, optional
          Axes to plot on. Creates new figure if None.

      Returns
      -------
      plt.Axes
          The axes with the plot.
      """
      if ax is None:
          fig, ax = plt.subplots(figsize=(12, 6))
      sns.histplot(df[column], kde=True, ax=ax)
      ax.set_title(f"Distribution of {column}")
      return ax
  ```

- Another example with more complex types:
  ```python
  def summarize_by_group(
      df: pd.DataFrame,
      group_col: str,
      value_col: str,
      agg_funcs: list[str] = ["mean", "median", "std"],
  ) -> pd.DataFrame:
      """Compute grouped aggregation statistics.

      Parameters
      ----------
      df : pd.DataFrame
          Source dataframe.
      group_col : str
          Column to group by.
      value_col : str
          Column to aggregate.
      agg_funcs : list[str], optional
          Aggregation functions to apply. Defaults to mean, median, std.

      Returns
      -------
      pd.DataFrame
          Aggregated statistics per group.
      """
      return df.groupby(group_col)[value_col].agg(agg_funcs).round(2)
  ```

## Adaptation Guidelines

- **No date columns?** Skip Section 5 entirely.
- **Very few numerical columns?** Reduce Section 4, expand categorical analysis in Section 3.
- **No natural target variable?** Use clustering in Section 8 instead of classification/regression.
- **Too many columns (>30)?** Focus on top 10 most interesting, mention the rest briefly.
- **Very small dataset (<100 rows)?** Skip modeling, focus on descriptive statistics and visualization.
- **Very large dataset (>1M rows)?** Sample for visualizations, use full data for statistics.
