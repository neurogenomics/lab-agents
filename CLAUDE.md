# Lab-Agents Team Environment

## Overview

This is a shared agent environment for the Imperial Lab team. Agents have access to:
- **Labstep**: Lab notebook (read-only by default)
- **OneDrive (SharePoint)**: Shared data files via synced OneDrive folders (strictly read-only)

---

## Available Skills

All skills are located in `.claude/skills/` and are auto-triggered based on context. Refer to each skill's `SKILL.md` for detailed usage:

### 1. **labstep**
- **Purpose**: Fetch and query Labstep experiments, protocols, and inventory
- **Trigger**: Questions about experiments, protocols, or lab records
- **Access**: Read-only (see below)

### 2. **labstep-sentiment**
- **Purpose**: Analyze sentiment/themes in lab notes
- **Trigger**: Questions about trends or patterns in experiment notes
- **Dependency**: Requires `labstep` skill access

### 3. **pptx**
- **Purpose**: Generate and modify PowerPoint presentations
- **Trigger**: Requests to create slides, generate reports, or present data
- **Scripts**: Includes `/scripts` subdirectory with helper utilities

### 4. **read-from-sharepoint**
- **Purpose**: Read-only access to lab data from synced OneDrive (SharePoint) folders
- **Trigger**: Any skill or task needing experiment data files
- **Data locations** (auto-detected per OS via `find_onedrive_base()`):
  - `<OneDrive root>/Skene lab - WB - 07 scRNA-seq/`
  - `<OneDrive root>/Skene lab - WB - 03 scTIP-Seq Development/`
- **Retry**: Files may need time to sync from cloud — retries up to 3 times with delays (2s, 5s, 10s)

### 5. **nucleic-acid-analysis**
- **Purpose**: Read and extract RNA quantification data (Qubit, TapeStation, qPCR)
- **Trigger**: Questions about RNA-seq datasets, QC metrics
- **Data source**: OneDrive folders (via `read-from-sharepoint`)

### 6. **experiment-summary**
- **Purpose**: Generate filled-in Experiment Summary .docx from lab data
- **Trigger**: `/experiment-summary` or "write up an experiment"
- **Data sources**: Labstep (metadata) + OneDrive (QC files)

---

## Security & Read-Only Policies

### Labstep (Read-Only)

**Standing rule: Agents will NOT create, update, or delete Labstep entries unless the user explicitly confirms with the phrase "confirm write".**

- The Labstep skill is configured for read-only access via a dedicated read-only service account
- The API token is scoped and monitored; any write attempts should be flagged
- **If you are asked to modify a Labstep entry**, ask the user for explicit confirmation:
  - "I can [describe the change]. To proceed, please confirm write: `confirm write`"

### OneDrive / SharePoint (Strictly Read-Only)

**NEVER write, move, rename, copy-into, or delete ANY file under OneDrive paths.**

These folders are shared team resources synced from SharePoint. Any modification propagates to all team members and risks data loss.

- **Allowed**: `open("r")`, `pd.read_excel()`, `pd.read_csv()`, `Document()` (read-only load), `ls`, `find`, `stat`
- **Forbidden**: Any write, move, rename, delete, or copy-into operation targeting OneDrive paths
- **All output files** (generated summaries, plots, etc.) must be saved to the **working directory**, never to OneDrive
- **Retry on failure**: OneDrive files may be cloud-only placeholders that need time to download — always use retry logic (see `read-from-sharepoint` skill)

---

## Labstep Credentials

The `labstep` skill uses a dedicated read-only service account:
- **Account**: `lab-agent-readonly@imperial.ac.uk`
- **Permission level**: Workspace **Viewer** role
- **API key location**: `.claude/settings.json` (configured by admin)

---

## Team Workflows

### Data Pipeline
1. Read experiment data directly from OneDrive synced folders (via `read-from-sharepoint` skill)
2. Query `nucleic-acid-analysis` or other analysis skills against OneDrive data
3. Use `pptx` skill to generate reports/presentations
4. All output files are saved locally in the working directory — no write-back to OneDrive or Labstep without explicit user request + confirmation

### Lab Queries
1. Use `labstep` skill to fetch experiments/protocols
2. Use `labstep-sentiment` for trend analysis
3. Results are read-only; modifications require user confirmation

---

## Contacting Admin

If you encounter issues with Labstep or OneDrive access:
- Check `.claude/settings.json` for valid credentials
- Verify OneDrive sync status (System Settings > OneDrive)
- Report access errors to the lab tech lead

---

## Questions?

Each skill's `SKILL.md` file contains detailed documentation and examples.
