---
name: execute-notebook
description: Execute a Jupyter notebook headless with kernel restart using papermill. Captures all cell outputs and overwrites the notebook with the executed version. Fails loudly on the first error with cell index and traceback.
argument-hint: "[notebook-path]"
allowed-tools: Read Write Bash
---

# Skill: Execute Notebook

Run the notebook at `$ARGUMENTS` end-to-end with a fresh kernel and persist its outputs back into the file. This is the "restart and run all" hygiene step recommended by Rule 7 of *Ten Simple Rules for Writing and Sharing Computational Analyses in Jupyter Notebooks* (Rule et al. 2019), automated.

## Step 1 — Pre-flight checks

Before invoking papermill, verify:

1. **Notebook exists**: resolve `$ARGUMENTS` to an absolute path. If the file does not exist, fail with: `Notebook not found: <path>`.
2. **Papermill installed**: run `python3 -c "import papermill"` via `Bash`. If it raises `ModuleNotFoundError`, fail with:
   ```
   papermill not installed. Install with one of:
     pip install papermill
     poetry add papermill
   ```
3. **Dependencies sanity check** (warning, not error): if neither `requirements.txt` nor `pyproject.toml` exists in the notebook's directory or cwd, print:
   ```
   WARNING: no requirements.txt or pyproject.toml found. Consider running /freeze-deps first.
   ```
   Do not abort on this warning — proceed with execution.
4. **Kernel availability**: read the notebook JSON, extract `metadata.kernelspec.name`. Run `jupyter kernelspec list --json` via `Bash` and verify the kernel exists. If not, fail with:
   ```
   Kernel '<name>' not available. Available kernels:
   <output of jupyter kernelspec list>
   ```

## Step 2 — Execute the notebook

Invoke papermill from the **notebook's directory** as cwd (so relative paths like `data/foo.csv` resolve correctly):

```bash
cd <notebook_dir>
python3 -c "
import papermill as pm
import time
start = time.time()
try:
    pm.execute_notebook(
        input_path='<notebook_filename>',
        output_path='<notebook_filename>',  # overwrite
        kernel_name='<kernel_from_metadata>',
        progress_bar=False,
        log_output=False,
        request_save_on_cell_execute=True,  # save partial output on failure
    )
    print(f'OK {time.time() - start:.1f}')
except pm.PapermillExecutionError as exc:
    print(f'FAIL {time.time() - start:.1f} cell={exc.cell_index} ename={exc.ename} evalue={exc.evalue}')
    raise
"
```

Capture stdout/stderr. Distinguish the OK and FAIL cases for the report.

## Step 3 — Report to the user

### On success

```
Notebook executed in <T>s — <N> cells run, 0 errors. Updated <path>.
```

`<N>` = count of `cell_type == 'code'` cells in the post-execution notebook.

After printing the success line, also check the notebook for placeholder observation markers (`<!-- annotate-findings: pending -->`). If any are present, append:

```
Hint: <K> placeholder observation cells detected. Run /annotate-findings <path> to fill them with real findings derived from the executed outputs.
```

This closes the workflow loop: `/analyze-dataset` writes structure with placeholders → `/execute-notebook` runs the code → `/annotate-findings` writes real findings based on outputs.

### On failure

Read the partially-saved notebook (papermill writes it even on failure thanks to `request_save_on_cell_execute=True`). Find the first cell with `outputs` containing an `error` output. Extract:

- `ename` and `evalue`
- The first 5 non-empty lines of the failing cell's `source`

Print:

```
Notebook execution FAILED at cell <index> (took <T>s before error).
Error: <ename>: <evalue>
First lines of failing cell:
    <line 1>
    <line 2>
    ...
Partial output saved to <path>. Inspect that cell, fix the issue, and re-run /execute-notebook.
```

If the failure is a `ModuleNotFoundError`, append:
```
Hint: missing module. Run /freeze-deps and ensure all dependencies are installed.
```

## Constraints

- All Python code in helper scripts must use type hints + NumPy-style docstrings.
- Do **not** modify the notebook except via papermill (no manual cell injection).
- Do **not** retry on failure — let the user inspect and re-run explicitly.
- Do **not** suppress traceback information; the goal is loud failure with actionable detail.
- The notebook's existing kernel metadata is authoritative — do not silently fall back to a different kernel.

## Out of scope (v1)

- **Parametrization**: papermill supports `parameters` cell injection, but accepting params via `$ARGUMENTS` complicates the contract. Future skill `/execute-notebook-with-params` or extension of this one.
- **Notebook validation**: not checking notebook format version or schema. Papermill handles that internally.
- **Concurrent execution of multiple notebooks**: this skill executes one at a time. Use a wrapper or shell loop for batches.
