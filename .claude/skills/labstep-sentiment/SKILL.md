---
name: labstep-sentiment
description: Analyse sentiment and outcome of Labstep experiments. Use when the user wants to understand how experiments went, whether results were positive or negative, or wants a sentiment/outcome summary across experiments.
---

# Labstep Experiment Sentiment Skill

Analyse the sentiment and scientific outcome of experiments stored in Labstep. The primary rich text content lives in the `state` field of the **root sub-experiment**, accessible via the REST API directly (not via labstepPy).

## Authentication

```python
import json, requests, labstep
from labstep.service.config import configService

from pathlib import Path
ROOT = Path(__file__).resolve().parents[4]  # project root

with open(ROOT / 'config.json') as f:
    cfg = json.load(f)

user = labstep.authenticate(cfg['email'], cfg['api_key'])
api_key = cfg['api_key']
base = configService.getHost()          # https://api.labstep.com
headers = {"apikey": api_key}
```

## How to Get Rich Text Body Per Experiment

The rich text you see in the Labstep UI (aims, objectives, notes) is stored as a **ProseMirror JSON document** in the `state` field of the root sub-experiment. Access it via REST:

```python
import requests

def get_experiment_body(workflow_id, headers, base):
    """Fetch the plain-text body of an experiment workflow."""
    # 1. Get the workflow to find the root experiment ID
    wf = requests.get(f"{base}/api/generic/experiment-workflow/{workflow_id}", headers=headers).json()
    root = wf.get('root_experiment', {})
    root_id = root.get('id') if isinstance(root, dict) else None
    if not root_id:
        return ""

    # 2. Fetch the root experiment
    exp = requests.get(f"{base}/api/generic/experiment/{root_id}", headers=headers).json()
    state = exp.get('state')
    if not state:
        return ""

    # 3. Recursively extract plain text from ProseMirror doc
    return extract_text(state).strip()

def extract_text(node):
    """Recursively extract plain text from a ProseMirror node."""
    if isinstance(node, str):
        return node
    if isinstance(node, dict):
        text = node.get('text', '')
        if text:
            return text
        children = node.get('content', [])
        result = ''.join(extract_text(c) for c in children)
        if node.get('type') in ('paragraph', 'heading', 'blockquote', 'list_item', 'hard_break'):
            result += '\n'
        return result
    if isinstance(node, list):
        return ''.join(extract_text(n) for n in node)
    return ''
```

## What to Collect Per Experiment

Combine the rich text body with structured metadata:

```python
def get_experiment_text(e, headers, base):
    parts = [f"Name: {e.name}"]

    # Rich text body (aims, objectives, conclusions)
    body = get_experiment_body(e.id, headers, base)
    if body:
        parts.append(f"Body:\n{body}")

    # Structured data field values
    try:
        for field in e.getDataFields():
            label = getattr(field, 'label', '') or ''
            value = getattr(field, 'value', '') or ''
            if value:
                parts.append(f"{label}: {value}")
    except:
        pass

    # Protocol names
    try:
        for protocol in e.getProtocols():
            if protocol.name:
                parts.append(f"Protocol: {protocol.name}")
    except:
        pass

    return "\n".join(parts)
```

## Sentiment Analysis Approach

Analyse each experiment's text and assign:

**Sentiment** (one of):
- `Positive` — experiment worked, results as expected, optimisation succeeded
- `Negative` — experiment failed, troubleshooting needed, poor yield/quality
- `Neutral` — routine/inconclusive, no clear outcome stated, template/blank entries
- `Mixed` — partial success, some conditions worked and others didn't

**Confidence** (one of): `High`, `Medium`, `Low`
- High: explicit outcome language in body text ("worked", "failed", "good yield", "no signal")
- Medium: indirect signals from name or partial body text
- Low: name only, no body text available

**Key signals:**
- Positive: "success", "worked", "optimised", "good yield", "clean", "as expected", "best condition", "sent for sequencing"
- Negative: "failed", "no signal", "low yield", "troubleshoot", "issue", "problem", "degraded", "contamination", "cancelled"
- Neutral: blank body, template entries (`_YYYYMMDD`), reference/training experiments
- Mixed: comparisons with clear winners and losers, "some conditions worked"

**Name-based fallback when body is empty:**
- "Repeat of:" → prior attempt had issues (lean Negative/Mixed)
- "_YYYYMMDD" → blank template (Neutral)
- "Troubleshoot" → Negative
- "Best condition" / "Final" → Positive

## Output Format

```python
results = []
for e in experiments:
    text = get_experiment_text(e, headers, base)
    sentiment, confidence, reasoning = analyse_sentiment(text)
    results.append({
        'id': e.id,
        'custom_identifier': getattr(e, 'custom_identifier', ''),
        'name': e.name,
        'author': e.author.get('name', '') if isinstance(e.author, dict) else '',
        'created_at': (e.created_at or '')[:10],
        'sentiment': sentiment,
        'confidence': confidence,
        'reasoning': reasoning,
        'body_text': text[:500],
    })
```

## Excel Export

```python
import openpyxl
from openpyxl.styles import Font, PatternFill, Alignment

SENTIMENT_COLOURS = {
    'Positive': 'C6EFCE',
    'Negative': 'FFC7CE',
    'Mixed':    'FFEB9C',
    'Neutral':  'D9D9D9',
}

wb = openpyxl.Workbook()
ws = wb.active
ws.title = "Sentiment Analysis"
headers = ["ID", "Custom ID", "Name", "Author", "Created At", "Sentiment", "Confidence", "Reasoning", "Body Text"]
ws.append(headers)

header_fill = PatternFill(start_color="4472C4", end_color="4472C4", fill_type="solid")
for cell in ws[1]:
    cell.fill = header_fill
    cell.font = Font(bold=True, color="FFFFFF")
    cell.alignment = Alignment(horizontal="center")

for row in results:
    ws.append([row['id'], row['custom_identifier'], row['name'], row['author'],
               row['created_at'], row['sentiment'], row['confidence'],
               row['reasoning'], row['body_text']])
    colour = SENTIMENT_COLOURS.get(row['sentiment'], 'FFFFFF')
    ws.cell(ws.max_row, 6).fill = PatternFill(start_color=colour, end_color=colour, fill_type="solid")

for col in ws.columns:
    w = max((len(str(c.value)) if c.value else 0) for c in col)
    ws.column_dimensions[col[0].column_letter].width = min(w + 2, 60)

wb.save('/Users/jaymoore/Documents/JAY_PhD/imperial/experimental-patent/outputs/sentiment_analysis.xlsx')
```

## Notes

- The `state` field is only on the **root sub-experiment** (`is_root=True`), not on the workflow object itself
- This requires **2 REST calls per experiment** (workflow → root experiment), so batch carefully
- `state` is `None` for experiments with no body written — fall back to name-based inference
- The ProseMirror doc contains headings, paragraphs, tables, bullet lists — all extracted as plain text
