---
name: rna-data
description: >
  Read and extract RNA quantification data (Qubit, TapeStation, qPCR) from
  experiment folders on OneDrive. Use when the user wants to read Metadata.xlsx,
  TapeStation CSVs, or Qubit values from experiment folders.
---

# RNA Quantification Data Reading Skill

Utilities for reading and extracting RNA quantification metrics (Qubit, TapeStation, etc.) from experiment data files on OneDrive.

**Data source**: OneDrive synced SharePoint folders (see `read-from-sharepoint` skill for paths, retry logic, and read-only policy).

## Finding experiment folders

```python
from pathlib import Path
import time, pandas as pd

ONEDRIVE_BASE = Path(
    "/Users/jaymoore/Library/CloudStorage/"
    "OneDrive-SharedLibraries-ImperialCollegeLondon"
)
SCRNA_ROOT = ONEDRIVE_BASE / "Skene lab - WB - 07 scRNA-seq"
SCTIP_ROOT = ONEDRIVE_BASE / "Skene lab - WB - 03 scTIP-Seq Development"

def read_with_retry(read_fn, max_retries=3, delays=(2, 5, 10)):
    """Retry a read that may fail due to OneDrive sync latency."""
    last_err = None
    for attempt in range(max_retries):
        try:
            result = read_fn()
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

def find_folder(root: Path, prefix: str) -> Path | None:
    """Find folder in root directory starting with prefix."""
    try:
        candidates = read_with_retry(
            lambda: [p for p in root.iterdir() if p.is_dir() and p.name.startswith(prefix)]
        )
    except (FileNotFoundError, OSError):
        return None
    if not candidates:
        return None
    return max(candidates, key=lambda p: len(list(p.rglob("*.xlsx"))))
```

## Metadata.xlsx

```python
folder = find_folder(SCRNA_ROOT, "SK550")
meta_files = read_with_retry(lambda: list(folder.glob("*Metadata*.xlsx")))
df = read_with_retry(lambda: pd.read_excel(meta_files[0], sheet_name="Sheet1"))
```

**Common columns** (names may vary by lab):
| Column | Description |
|--------|-------------|
| `Aim` | Enzyme / condition name (long form) |
| `Sample` | Sample number or identifier |
| `RNA Qubit Conc (ng/ul)` | RNA yield measurement via Qubit |
| `Final Lib Qubit Conc (ng/ul)` | DNA library yield |
| `TapeStation HS D5000 100-1000 bp Conc (ng/ul)` | TapeStation concentration (specific window) |
| `TapeStation HS D5000 peak (bp)` | Peak fragment size from TapeStation |
| `PCR.cycles` | PCR cycling protocol |

## TL / "Too low" Qubit values

```python
def qubit_to_float(val) -> float | None:
    if val is None or (isinstance(val, float) and pd.isna(val)):
        return None
    s = str(val).strip().lower()
    if s in ("tl", "too low", "/", "", "nan"):
        return 0.0
    try:
        return float(s)
    except ValueError:
        return 0.0
```

## TapeStation sampleTable CSV

When Metadata.xlsx is missing or empty, read the TapeStation sampleTable CSV:

```python
csvs = read_with_retry(lambda: list(folder.rglob("*sampleTable.csv")))
df = read_with_retry(lambda: pd.read_csv(csvs[0], encoding="latin-1"))
df = df[~df["Sample Description"].str.lower().str.contains("ladder", na=False)]
```

Concentration is in **pg/ul**. Convert to ng/ul: `conc_pg / 1000`.

## Enzyme name normalisation

```python
IVT_NORM = {
    "hiscribet7 ivt kit (neb)": "HiScribe",
    "hiscribet7 1h": "HiScribe", "hiscribet7 2h": "HiScribe",
    "megascript t7 ivt kit (thermofisher)": "Mega",
    "t7 ribomax ivt kit (promega)": "RiboMax",
    "t7 ribomax 1h": "RiboMax", "t7 ribomax 2h": "RiboMax",
    "hi-t7 ivt kit (neb)": "Hi-T7", "hi-t7 kit": "Hi-T7",
    "standard kit": "Standard",
    "negative control": "Neg ctl",
}

RT_NORM = {
    "smart mmlv rtase kit": "SMART",
    "maxima h minus": "Maxima",
    "sigma-aldrich m-mlv reverse transcriptase": "Sigma",
    "sigma-aldrich m-mlv rt": "Sigma",
    "superscript iv": "SuperScript",
    "superscrip iv": "SuperScript",
    "negative control": "Neg ctl",
}

PCR_NORM = {
    "superfi ii": "SuperFi",
    "phusion high-fidelity": "Phusion",
    "primestar max": "PrimeStar",
    "nebNext ultra ii q5": "NEBNext",
    "negative control": "Neg ctl",
}
```

## Reusable helper: build_panel_data

```python
def build_panel_data(experiments, aim_norm, qubit_col, order):
    """Read Metadata.xlsx for a list of (exp_id, root) pairs.
    Returns (data_dict, exp_ids_dict) keyed by normalised enzyme label."""
    data    = {e: [] for e in order}
    exp_ids = {e: [] for e in order}
    for exp_id, root in experiments:
        folder = find_folder(root, exp_id)
        if not folder:
            continue
        meta_files = read_with_retry(lambda: list(folder.glob("*Metadata*.xlsx")))
        if not meta_files:
            continue
        df = read_with_retry(lambda: pd.read_excel(meta_files[0], sheet_name="Sheet1"))
        for _, row in df.iterrows():
            aim   = str(row.get("Aim", "")).strip()
            label = aim_norm.get(aim.lower())
            if label not in order:
                continue
            val = qubit_to_float(row.get(qubit_col))
            if val is None:
                continue
            data[label].append(val)
            exp_ids[label].append(exp_id)
    for e in order:
        if not data[e]:
            data[e] = None
    return data, exp_ids
```

## Tips

1. **All reads use retry logic** — OneDrive files may need time to sync from cloud
2. **Check units** — Qubit is ng/ul; TapeStation conc is pg/ul (divide by 1000)
3. **Note dilutions** — Document any sample dilutions before measurement
4. **Never write to OneDrive** — all output goes to the working directory
