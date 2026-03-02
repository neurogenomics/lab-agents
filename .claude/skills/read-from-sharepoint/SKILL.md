---
name: read-from-sharepoint
description: >
  Read-only access to lab data files from synced OneDrive (SharePoint) folders.
  Use when any skill or task needs to read experiment data files (Metadata.xlsx,
  TapeStation CSVs/PDFs, qPCR files, EVOS images, Experiment Summaries).
  Trigger on: reading SharePoint data, loading experiment files, accessing OneDrive data.
---

# Read from SharePoint (OneDrive) Skill

Provides strictly read-only access to lab data via locally-synced OneDrive folders.

## STRICT READ-ONLY POLICY

**NEVER write, move, rename, copy-into, or delete ANY file under OneDrive paths.**

These folders are shared team resources synced from SharePoint. Any modification
propagates to all team members. Violations risk data loss for the entire lab.

Allowed operations:
- `open()` with mode `"r"` or `"rb"` only
- `pd.read_excel()`, `pd.read_csv()` (read-only by default)
- `Document()` from python-docx (read-only load)
- `ls`, `find`, `stat` via shell (read-only)

Forbidden operations:
- Any `open()` with `"w"`, `"a"`, `"x"` modes
- `shutil.copy/move`, `os.rename`, `os.remove`, `pathlib.Path.unlink/rename/write_*`
- `rsync`, `cp`, `mv`, `rm` targeting OneDrive paths
- Saving/writing any file back to OneDrive paths

## OneDrive base paths

```python
from pathlib import Path

ONEDRIVE_BASE = Path(
    "/Users/jaymoore/Library/CloudStorage/"
    "OneDrive-SharedLibraries-ImperialCollegeLondon"
)

LIBRARIES = {
    "scrna": ONEDRIVE_BASE / "Skene lab - WB - 07 scRNA-seq",
    "sctip": ONEDRIVE_BASE / "Skene lab - WB - 03 scTIP-Seq Development",
}
```

## Retry logic for cloud-synced files

OneDrive files may not be immediately available — they can be cloud-only placeholders
that need time to download. Always wrap file reads with retry logic:

```python
import time

def read_with_retry(read_fn, max_retries=3, delays=(2, 5, 10)):
    """
    Retry a read operation that may fail due to OneDrive sync latency.

    read_fn: callable that performs the read (e.g. lambda: pd.read_excel(path))
    Returns the result of read_fn on success.
    Raises the last exception after all retries are exhausted.
    """
    last_err = None
    for attempt in range(max_retries):
        try:
            result = read_fn()
            # Guard against empty results from partially-synced files
            if result is None:
                raise ValueError("Read returned None — file may still be syncing")
            return result
        except (FileNotFoundError, PermissionError, OSError, ValueError) as e:
            last_err = e
            if attempt < max_retries - 1:
                wait = delays[attempt] if attempt < len(delays) else delays[-1]
                print(f"  Retry {attempt + 1}/{max_retries} in {wait}s: {e}")
                time.sleep(wait)
    raise last_err
```

### Usage examples

```python
# Reading an Excel file with retry
df = read_with_retry(lambda: pd.read_excel(metadata_path, sheet_name="Sheet1"))

# Reading a CSV with retry
ts_df = read_with_retry(lambda: pd.read_csv(csv_path, encoding="latin-1"))

# Reading a docx with retry
doc = read_with_retry(lambda: Document(str(docx_path)))

# Listing directory contents with retry (folder may not be synced yet)
contents = read_with_retry(lambda: list(folder.iterdir()))
```

## Finding experiment folders

```python
def find_experiment_folder(experiment_id: str, library: str = "scrna") -> Path | None:
    """
    Find an experiment folder by ID prefix in the OneDrive library.
    Returns the best match (most data files) or None.
    """
    root = LIBRARIES.get(library)
    if root is None or not root.exists():
        return None

    try:
        candidates = read_with_retry(
            lambda: [p for p in root.iterdir() if p.is_dir() and p.name.startswith(experiment_id)]
        )
    except (FileNotFoundError, OSError):
        return None

    if not candidates:
        return None
    if len(candidates) == 1:
        return candidates[0]
    # Prefer folder with most data files
    return max(candidates, key=lambda p: len(list(p.rglob("*.xlsx"))))
```

## Common file patterns in experiment folders

```
<experiment_id>_<description>/
├── *Metadata*.xlsx          — Sample metadata, Qubit, TapeStation values
├── EVOS Images/             — SYBR-stained cell images (.heic/.png/.tif)
├── Tapestation/             — TapeStation PDFs and/or sampleTable CSVs
├── qPCR/                    — qPCR .eds, .xlsx, plots, R scripts
└── <experiment_id> - Experiment Summary.docx  — Completed summary (if exists)
```

## Important notes

- **Always output the full OneDrive path** when reporting which file was read, so the user can verify
- **If a file fails after all retries**, report the error clearly and suggest the user check that OneDrive is synced (System Settings > OneDrive > sync status)
- **Never cache or copy** OneDrive files to local directories without explicit user request
- **All output files** (generated summaries, plots, etc.) must be saved to the **working directory**, never to OneDrive paths
