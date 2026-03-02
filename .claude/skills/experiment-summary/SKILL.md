---
name: experiment-summary
description: >
  Generate a filled-in Experiment Summary .docx from lab data.
  Invoke with /experiment-summary or when the user asks to
  "write up an experiment", "generate experiment summary", or
  "fill in the summary template".
user_invocable: true
---

# /experiment-summary

Generate a completed Experiment Summary `.docx` by pulling data from Labstep, SharePoint, and local QC files, then filling in the lab template.

## Invocation

```
/experiment-summary <experiment_id_or_name>
```

Examples:
- `/experiment-summary SK550`
- `/experiment-summary "New lysis buffer test"`

If no argument is given, ask the user which experiment to summarise.

---

## Workflow

Follow these steps in order. At each step, tell the user what you found and what's still missing.

### Step 1 — Identify the experiment

Parse the argument as an experiment ID (e.g. `SK550`) or a search term.

### Step 2 — Pull metadata from Labstep

Use the `labstep` skill's API to fetch experiment details:

```python
import json, labstep
from pathlib import Path

with open(Path("config.json")) as f:
    cfg = json.load(f)
user = labstep.authenticate(cfg["api_key"])

# Search by name/ID
experiments = user.getExperiments(search_query="SK550", count=5)
exp = experiments[0]  # or let the user pick if ambiguous

# Extract what we can
name = exp.name
data_fields = exp.getDataFields()
protocols = exp.getProtocols()
comments = exp.getComments()
tables = exp.getTables()
```

From Labstep, attempt to extract:
| Field | Source |
|-------|--------|
| **Model** | Data field named "Model" or "Cell line", or parse from experiment name |
| **Aim** | Data field named "Aim", or the experiment description/entry |
| **Methods** | Linked protocol body, or data field named "Methods" |
| **Discussion** | Comments or data field named "Discussion" / "Notes" |

If a field can't be found in Labstep, note it as missing — you'll ask the user in Step 5.

### Step 3 — Pull QC data from local files (rna-quant)

Search for the experiment folder in the local SharePoint mirror or working directory:

```python
import pandas as pd
from pathlib import Path

def find_folder(root: Path, prefix: str) -> Path | None:
    candidates = [p for p in root.iterdir() if p.is_dir() and p.name.startswith(prefix)]
    if not candidates:
        return None
    return max(candidates, key=lambda p: len(list(p.rglob("*.xlsx"))))

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

**Search order for experiment folder:**
1. Current working directory
2. Any subdirectories of the working directory
3. The local SharePoint mirror (use `sync-data` paths if available)

**From the experiment folder, extract:**

#### Qubit concentrations
```python
meta_files = list(folder.glob("*Metadata*.xlsx"))
if meta_files:
    df = pd.read_excel(meta_files[0], sheet_name="Sheet1")
    qubit_col = "RNA Qubit Conc (ng/ul)"  # or "Final Lib Qubit Conc (ng/ul)"
    # Try multiple column names — labs use different headers
    for col in ["RNA Qubit Conc (ng/ul)", "Final Lib Qubit Conc (ng/ul)",
                "Qubit Conc (ng/ul)", "Qubit (ng/ul)"]:
        if col in df.columns:
            qubit_col = col
            break
    samples = []
    for _, row in df.iterrows():
        name = str(row.get("Aim", row.get("Sample", f"Sample {_ + 1}")))
        conc = qubit_to_float(row.get(qubit_col))
        samples.append({"name": name, "conc": conc if conc is not None else ""})
```

#### TapeStation data
```python
# From Metadata.xlsx
ts_conc_col = "TapeStation HS D5000 100-1000 bp Conc (ng/ul)"
ts_peak_col = "TapeStation HS D5000 peak (bp)"
if ts_conc_col in df.columns and df[ts_conc_col].notna().any():
    tapestation = {
        "conc": str(df[ts_conc_col].dropna().iloc[0]),
        "size": str(df[ts_peak_col].dropna().iloc[0]) if ts_peak_col in df.columns else "",
    }
else:
    # Fall back to TapeStation sampleTable CSV
    csvs = list(folder.rglob("*sampleTable.csv"))
    if csvs:
        ts_df = pd.read_csv(csvs[0], encoding="latin-1")
        ts_df = ts_df[~ts_df["Sample Description"].str.lower().str.contains("ladder", na=False)]
        # Extract concentration (pg/µl → ng/µl)
        tapestation = {
            "conc": f"{ts_df['Conc. [pg/µl]'].mean() / 1000:.2f} ng/µl (mean)",
            "size": "",
        }
```

### Step 4 — Pull files from SharePoint (if local data is missing)

If the experiment folder wasn't found locally, try the SharePoint MCP:

```
Use the sharepoint MCP tools:
- search_files: find "SK550 Metadata" or "SK550 TapeStation"
- get_file_content: download the Metadata.xlsx or sampleTable.csv
```

Save any downloaded files to a temporary location and parse as in Step 3.

### Step 5 — Confirm with user

Present a summary of what was found and what's missing:

```
## Experiment Summary for SK550

**Found:**
- Model: HEK293T (from Labstep)
- Aim: Testing new lysis buffer (from Labstep)
- Qubit: 5 samples loaded from Metadata.xlsx
- TapeStation: Conc 2.45 ng/µl, peak 350 bp

**Still needed:**
- Summary of methods (not found in Labstep)
- Discussion

Please provide the missing fields, or type "skip" to leave them blank.
```

Wait for the user to fill in missing fields or confirm.

### Step 6 — Generate the document

```python
from docx import Document
from pathlib import Path

TEMPLATE = Path("support/Experiment Summary Template.docx")

def fill_experiment_summary(
    output_path: str,
    model: str,
    aim: str,
    methods: str,
    qubit_samples: list[dict],
    tapestation: dict | None = None,
    qpcr: str | None = None,
    discussion: str = "",
):
    doc = Document(str(TEMPLATE))

    # Fill header paragraphs
    for p in doc.paragraphs:
        if p.text.startswith("Model:"):
            p.clear()
            p.add_run(f"Model: {model}")
        elif p.text.startswith("Aim:"):
            p.clear()
            p.add_run(f"Aim: {aim}")
        elif p.text.startswith("Summary of methods:"):
            p.clear()
            p.add_run(f"Summary of methods: {methods}")
        elif p.text.startswith("Discussion"):
            p.clear()
            p.add_run(f"Discussion and comparison to past experiments\n{discussion}")

    # Fill Qubit table (Table 0) — clear template rows, add actual data
    qubit_table = doc.tables[0]
    while len(qubit_table.rows) > 2:
        qubit_table._tbl.remove(qubit_table.rows[2]._tr)
    for sample in qubit_samples:
        row = qubit_table.add_row()
        row.cells[0].text = str(sample["name"])
        row.cells[1].text = str(sample.get("conc", ""))

    # Fill TapeStation (Table 2)
    if tapestation:
        ts_cell = doc.tables[2].rows[1].cells[0]
        ts_cell.text = (
            f"Dilution: {tapestation.get('dilution', '')}\n\n"
            f"Spectrum: {tapestation.get('spectrum', '')}\n\n"
            f"Size: {tapestation.get('size', '')}\n"
            f"Conc.: {tapestation.get('conc', '')}\n"
            f"Peak molarity (accounted for dilution): {tapestation.get('molarity', '')}"
        )

    # Fill qPCR (Table 3)
    if qpcr:
        doc.tables[3].rows[1].cells[0].text = qpcr

    doc.save(output_path)
    return output_path
```

### Step 7 — Save and report

- Save as `<experiment_id>_Experiment_Summary.docx` in the current working directory
- Tell the user the output path
- Never overwrite the template

---

## Data source priority

For each field, prefer data in this order:

| Field | 1st choice | 2nd choice | 3rd choice |
|-------|-----------|-----------|-----------|
| Model | Labstep data field | User input | — |
| Aim | Labstep data field / entry | User input | — |
| Methods | Labstep linked protocol | User input | — |
| Qubit | Local Metadata.xlsx | SharePoint file | User input |
| TapeStation | Local Metadata.xlsx or CSV | SharePoint file | User input |
| qPCR | Local files | Labstep tables | User input |
| Imaging | Local image files | — | User input |
| Discussion | Labstep comments | User input | — |

---

## Template location

```
support/Experiment Summary Template.docx
```

The template has 4 tables:
- **Table 0**: Qubit concentrations (sample name + ng/µl)
- **Table 1**: EVOS Imaging (image placeholders)
- **Table 2**: TapeStation HS D5000 (dilution, spectrum, size, conc, molarity)
- **Table 3**: qPCR side reaction

---

## Dependencies

```
pip install python-docx labstep pandas openpyxl
```

---

## Important notes

- **Never overwrite the template** — always save as a new file
- **Labstep is read-only** — do not create or modify Labstep entries (see CLAUDE.md)
- **SharePoint is read-only** — only use `search_files` and `get_file_content`
- Qubit "TL" / "Too low" values → treat as 0.0
- TapeStation CSV concentrations are in pg/µl — divide by 1000 for ng/µl
- If multiple data sources disagree, prefer the local file data and flag the discrepancy to the user
