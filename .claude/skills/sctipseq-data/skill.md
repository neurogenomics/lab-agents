---
name: sctipseq-data
description: Read and extract experimental QC data from scTIP-Seq / scRNA-seq Sharepoint mirror files. Use when the user wants to read Metadata.xlsx, TapeStation CSVs, or Qubit values from SK-numbered experiment folders in the Sharepoint mirror.
---

# scTIP-Seq Sharepoint Data Reading Skill

Knowledge accumulated from working with the Neurogenomics Lab scTIP-Seq experiment data.

## Paths

```python
from pathlib import Path
ROOT       = Path("/Users/jaymoore/Documents/JAY_PhD/imperial/experimental-patent")
SHAREPOINT = ROOT / "Sharepoint Mirror 25 Feb 2026"
SCTIP      = SHAREPOINT / "03 scTIP-Seq Development"   # SK083–SK500s
SCRNA      = SHAREPOINT / "07 scRNA-seq"               # SK157–SK568
```

## Finding experiment folders

Folders are named `SKxxx_description_YYYYMMDD_initials`. Always use prefix match and pick the best candidate:

```python
def find_folder(root: Path, prefix: str) -> Path | None:
    candidates = [p for p in root.iterdir() if p.is_dir() and p.name.startswith(prefix)]
    if not candidates:
        return None
    return max(candidates, key=lambda p: len(list(p.rglob("*.xlsx"))))
```

## Metadata.xlsx

Most experiments have `SKxxx - Metadata.xlsx` with a `Sheet1` containing per-sample rows.

```python
import pandas as pd
folder = find_folder(SCRNA, "SK457")
meta_files = list(folder.glob("*Metadata*.xlsx"))
df = pd.read_excel(meta_files[0], sheet_name="Sheet1")
```

**Key columns:**
| Column | Description |
|--------|-------------|
| `Aim` | Enzyme / condition name (long form, e.g. "SMART MMLV RTase kit") |
| `Sample` | Sample number (1–5 typically) |
| `RNA Qubit Conc (ng/ul)` | IVT RNA yield — "TL" / "Too low" → treat as 0.0 |
| `Final Lib Qubit Conc (ng/ul)` | RT/PCR DNA library yield |
| `TapeStation HS D5000 100-1000 bp Conc (ng/ul)` | RT/PCR TapeStation conc (100–1000 bp window) |
| `TapeStation HS D5000 peak (bp)` | Peak fragment size |
| `TapeStation HS RNA Conc (ng/ul)` | IVT RNA TapeStation conc |
| `TapeStation HS RNA peak (bp)` | IVT RNA peak size |
| `PCR.cycles` | PCR cycling protocol (e.g. "6+15", "16 cycles") |

**Critical: check for empty Metadata before trusting it.**
Some experiments have a Metadata.xlsx with all NaN QC values (e.g. SK463). Always check:

```python
ts_col = "TapeStation HS D5000 100-1000 bp Conc (ng/ul)"
has_data = df[ts_col].notna().any() if ts_col in df.columns else False
if not has_data:
    # Fall back to TapeStation sampleTable CSV
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
df = pd.read_csv(csvs[0], encoding="latin-1")   # latin-1 required — µ in "pg/µl"
# Drop ladder rows
df = df[~df["Sample Description"].str.lower().str.contains("ladder", na=False)]
```

**Columns:** `Well`, `Conc. [pg/µl]`, `Sample Description`, `Alert`, `Observations`

Concentration is in **pg/µl**. Convert to ng/µl: `conc_pg / 1000`.

**Dilution factors** — some runs use diluted samples, multiply accordingly:
- SK502 / SK509 PCR runs: **10× diluted** → `conc_pg * 10 / 1000`
- SK463 RT run: **not diluted** → `conc_pg / 1000`

## Known data co-location quirks

| Situation | What to do |
|-----------|------------|
| SK502 (PCR) has no TapeStation CSV | Both SK502 and SK509 were run together — CSV is in **SK509**'s TapeStation folder. Filter rows by `Sample Description` prefix: `SK502_1`…`SK502_5` |
| SK463 (RT) Metadata.xlsx is all NaN | Read TapeStation CSV directly from `SK463/Tapestation/SK463_TapeStation/` — sample descriptions are plain numbers 1–5 |
| SK449 "data to plot" sheet | Copy-paste error — shows SK443 data. Always use **Sheet1** for SK449 |
| SK502 Experiment Summary.docx | Qubit table mislabelled `509_...` (student copy-paste). Values are SK502 data; TapeStation rows correctly labelled `SK502_` |

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
    "superscrip iv": "SuperScript",   # typo in SK473
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

## Metric choices by panel (enzyme comparison figure)

| Panel | Metric | Reason |
|-------|--------|--------|
| IVT (Panel A) | `RNA Qubit Conc (ng/ul)` from Metadata.xlsx | Direct RNA yield |
| RT (Panel B) | `TapeStation HS D5000 100-1000 bp Conc (ng/ul)` | Matches thesis; Qubit inconsistent across replicates |
| PCR (Panel C) | Qubit for SK486 (Metadata); TapeStation ×10 for SK502/SK509 | No Metadata.xlsx for SK502/SK509 |

## Sample number → enzyme label (RT and PCR)

```python
RT_SAMPLE_LABEL  = {1: "SMART", 2: "Maxima", 3: "Sigma", 4: "SuperScript", 5: "Neg ctl"}
PCR_SAMPLE_LABEL = {1: "SuperFi", 2: "Phusion", 3: "PrimeStar", 4: "NEBNext", 5: "Neg ctl"}
```

These are consistent across all RT experiments (SK457, SK463, SK473) and PCR experiments (SK486, SK502, SK509).

## Excluded experiments and reasons

| Experiment | Excluded from | Reason |
|------------|--------------|--------|
| SK463 | Originally excluded from RT | Metadata.xlsx all NaN — but **TapeStation CSV has valid data**; now included |
| SK494 | PCR panel | Used 16 PCR cycles (single run) vs SK486/502/509 which used 6+15 (two-step); not comparable |
| SK380 | IVT panel | Both enzymes show Qubit TL (0); TapeStation data available but RNA yield negligible |

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

## Key scripts in this project

| Script | Purpose |
|--------|---------|
| `scripts/extract_enzymes.py` | Extracts IVT/RT/PCR/DNase data → `results/enzyme_data.xlsx` |
| `scripts/plot_enzyme_comparison.py` | 3-panel figure reading from enzyme_data.xlsx |
| `figure_replication/plot_enzyme_comparison.py` | Self-contained replication reading directly from Sharepoint files |
