---
name: experiment-summary
description: Fill in an Experiment Summary document from QC data (Qubit, TapeStation, qPCR, imaging). Use when the user asks to write up an experiment, generate an experiment summary, or fill in a summary template.
---

# Experiment Summary Skill

Generates a filled-in Experiment Summary `.docx` from the lab template and QC data.

## Template location

```
support/Experiment Summary Template.docx
```

## Template structure

The template has these sections that need to be populated:

### 1. Header fields (paragraphs)
| Field | Content |
|-------|---------|
| **Model** | Cell line or model system (e.g. "HEK293", "iPSC-derived neurons") |
| **Aim** | One-sentence experiment objective |

### 2. Summary of methods (paragraph)
Brief protocol description — what was done, key reagents, conditions.

### 3. Qubit Concentrations (Table 0)
HS DNA measurements. The table has two columns: `#` (sample name) and `ng/μl` (concentration).

Default sample rows in template:
- Standard 1, Standard 2
- Chomzynski Full Atrandi
- Chomzynski Short with PK
- Chomzynski short no PK
- Patch-seq short no PK
- Patch-seq short with PK

**Important**: The sample names and row count will vary per experiment. Rows should be added/removed to match the actual samples. Use the `rna-quant` skill's `qubit_to_float()` helper for parsing values — handle "TL" / "Too low" as 0.0.

### 4. Imaging (Table 1)
EVOS Imaging (SYBR stained). This table has image placeholders — insert paths or references to microscopy images. Leave empty if no imaging was done.

### 5. TapeStation (Table 2)
Single-cell table with structured text fields:
- **Dilution**: e.g. "1:5", "Undiluted"
- **Spectrum**: Description or path to electropherogram image
- **Size**: Fragment size in bp
- **Conc.**: Concentration in ng/µl or pg/µl (specify units)
- **Peak molarity (accounted for dilution)**: Final molarity value

### 6. qPCR Side Reaction (Table 3)
qPCR results — paste Ct values, amplification curves, or summary. Leave empty if not applicable.

### 7. Discussion (paragraph)
Comparison to past experiments, interpretation of results, next steps.

## How to fill in the template

```python
from docx import Document
from pathlib import Path
from copy import deepcopy

TEMPLATE = Path("support/Experiment Summary Template.docx")

def fill_experiment_summary(
    output_path: str,
    model: str,
    aim: str,
    methods: str,
    qubit_samples: list[dict],  # [{"name": "Sample 1", "conc": 12.3}, ...]
    tapestation: dict | None = None,  # {"dilution": "", "size": "", "conc": "", "molarity": ""}
    qpcr: str | None = None,
    discussion: str = "",
):
    doc = Document(str(TEMPLATE))

    # 1. Fill header fields
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

    # 2. Fill Qubit table (Table 0)
    qubit_table = doc.tables[0]
    # Keep header rows (0-1), remove template sample rows (2+)
    while len(qubit_table.rows) > 2:
        qubit_table._tbl.remove(qubit_table.rows[2]._tr)
    # Add actual samples
    for sample in qubit_samples:
        row = qubit_table.add_row()
        row.cells[0].text = sample["name"]
        row.cells[1].text = str(sample.get("conc", ""))

    # 3. Fill TapeStation (Table 2)
    if tapestation:
        ts_cell = doc.tables[2].rows[1].cells[0]
        ts_cell.text = (
            f"Dilution: {tapestation.get('dilution', '')}\n\n"
            f"Spectrum: {tapestation.get('spectrum', '')}\n\n"
            f"Size: {tapestation.get('size', '')}\n"
            f"Conc.: {tapestation.get('conc', '')}\n"
            f"Peak molarity (accounted for dilution): {tapestation.get('molarity', '')}"
        )

    # 4. Fill qPCR (Table 3)
    if qpcr:
        doc.tables[3].rows[1].cells[0].text = qpcr

    doc.save(output_path)
    return output_path
```

## Usage examples

### From user-provided data
```
User: "Write up the experiment summary for SK550. Model is HEK293T,
       aim was testing new lysis buffer. Qubit: Sample A = 15.2 ng/ul,
       Sample B = 8.7 ng/ul, Sample C = TL"
```

The agent should:
1. Parse the values from the user's message
2. Call `fill_experiment_summary()` with the parsed data
3. Save to a sensible output path (e.g. `SK550_Experiment_Summary.docx`)

### From Metadata.xlsx (using rna-quant skill)
```
User: "Generate an experiment summary for SK550 from its metadata"
```

The agent should:
1. Use the `rna-quant` skill to locate and read Metadata.xlsx
2. Extract Qubit concentrations, TapeStation values
3. Ask the user for Model, Aim, and Methods (these aren't in metadata)
4. Fill in the template and save

### Minimal summary
```
User: "Quick experiment summary — just Qubit values for my 3 samples"
```

The agent should:
1. Ask for sample names and concentrations
2. Fill only the Qubit table, leave other sections empty
3. Save the document

## Dependencies

```
pip install python-docx
```

## Notes

- Always save output as a new file — never overwrite the template
- The Qubit table rows should match the actual experiment samples, not the template defaults
- If the user provides images, note their file paths in the Imaging table but do not embed them programmatically (python-docx image insertion can break formatting)
- Use the `rna-quant` skill for parsing Qubit/TapeStation values when reading from experiment data files
