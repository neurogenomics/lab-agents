---
name: rna-quant
description: Read and extract RNA quantification data from experiment folders. Use when the user wants to read Metadata.xlsx, TapeStation CSVs, or Qubit values from local experiment folders.
---

# RNA Quantification Data Reading Skill

Utilities for reading and extracting RNA quantification metrics (Qubit, TapeStation, etc.) from experiment metadata and QC data files.

## Finding experiment folders

Experiment folders are typically named with an experiment ID prefix. To find a folder by ID:

```python
def find_folder(root: Path, prefix: str) -> Path | None:
    """Find folder in root directory starting with prefix.
    Returns the candidate with the most .xlsx files (likely has Metadata)."""
    candidates = [p for p in root.iterdir() if p.is_dir() and p.name.startswith(prefix)]
    if not candidates:
        return None
    return max(candidates, key=lambda p: len(list(p.rglob("*.xlsx"))))
```

**Usage**: Pass a `Path` to the experiment root directory and an experiment ID prefix.

## Metadata.xlsx

Most experiments have a `Metadata.xlsx` file with a `Sheet1` containing per-sample rows.

```python
import pandas as pd
folder = find_folder(root_path, "experiment_id")
meta_files = list(folder.glob("*Metadata*.xlsx"))
df = pd.read_excel(meta_files[0], sheet_name="Sheet1")
```

**Common columns** (names may vary by lab):
| Column | Description |
|--------|-------------|
| `Aim` | Enzyme / condition name (long form, e.g. "SMART MMLV RTase kit") |
| `Sample` | Sample number or identifier |
| `RNA Qubit Conc (ng/ul)` | RNA yield measurement via Qubit |
| `Final Lib Qubit Conc (ng/ul)` | DNA library yield |
| `TapeStation HS D5000 100-1000 bp Conc (ng/ul)` | TapeStation concentration (specific window) |
| `TapeStation HS D5000 peak (bp)` | Peak fragment size from TapeStation |
| `PCR.cycles` | PCR cycling protocol |

**Best practice: always validate Metadata before trusting values.**

```python
# Check if key columns have data
ts_col = "TapeStation HS D5000 100-1000 bp Conc (ng/ul)"
has_data = df[ts_col].notna().any() if ts_col in df.columns else False
if not has_data:
    # Consider falling back to TapeStation CSV or other QC files
```

## TL / "Too low" Qubit values

RNA Qubit values of "TL" or "Too low" mean below detection limit — treat as 0.0:

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

When Metadata.xlsx is missing or empty, read the TapeStation sampleTable CSV directly.

```python
import pandas as pd
csvs = list(folder.rglob("*sampleTable.csv"))
df = pd.read_csv(csvs[0], encoding="latin-1")   # latin-1 required for special characters
# Drop ladder rows
df = df[~df["Sample Description"].str.lower().str.contains("ladder", na=False)]
```

**Common columns:** `Well`, `Conc. [pg/µl]`, `Sample Description`, `Alert`, `Observations`

Concentration is typically in **pg/µl**. Convert to ng/µl: `conc_pg / 1000`.

**⚠️ Watch for dilution factors** — some runs use diluted samples. Document any dilution:

```python
# If samples were diluted before TapeStation run, multiply accordingly
dilution_factor = 10  # Example: 10× dilution
conc_ng_ul = (conc_pg / 1000) * dilution_factor
```

## Enzyme name normalisation

Aim column values are long-form; normalise to short display labels:

```python
# IVT
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

# RT
RT_NORM = {
    "smart mmlv rtase kit": "SMART",
    "maxima h minus": "Maxima",
    "sigma-aldrich m-mlv reverse transcriptase": "Sigma",
    "sigma-aldrich m-mlv rt": "Sigma",
    "superscript iv": "SuperScript",
    "superscrip iv": "SuperScript",   # watch for common typos
    "negative control": "Neg ctl",
}

# PCR
PCR_NORM = {
    "superfi ii": "SuperFi",
    "phusion high-fidelity": "Phusion",
    "primestar max": "PrimeStar",
    "nebNext ultra ii q5": "NEBNext",
    "negative control": "Neg ctl",
}
```

Always normalise before matching: `aim.strip().lower()`.

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
        meta_files = list(folder.glob("*Metadata*.xlsx"))
        if not meta_files:
            continue
        df = pd.read_excel(meta_files[0], sheet_name="Sheet1")
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

## Tips for working with RNA quantification data

1. **Validate before using** — Always check that Metadata or QC files contain actual data, not just NaN values
2. **Check units** — Qubit is typically ng/µl; TapeStation conc is in pg/µl (divide by 1000)
3. **Note dilutions** — Document any sample dilutions before measurement
4. **Use consistent metrics** — For comparisons, choose one QC metric (Qubit or TapeStation) and stick with it
5. **Handle missing data** — Qubit "TL" (too low) values should be treated as 0 or excluded depending on context
