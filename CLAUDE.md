# Lab-Agents Team Environment

## Overview

This is a shared agent environment for the Imperial Lab team. Agents have access to:
- **Labstep**: Lab notebook (read-only by default)
- **SharePoint**: Shared document access (read-only)
- **Local data**: Synchronized copies of SharePoint data, lab-specific analysis skills

---

## Available Skills

All skills are located in `.claude/skills/` and are auto-triggered based on context. Refer to each skill's `SKILL.md` for detailed usage:

### 1. **labstep**
- **Purpose**: Fetch and query Labstep experiments, protocols, and inventory
- **Trigger**: Questions about experiments, protocols, or lab records
- **Access**: Read-only (see below)
- **MCP**: Direct API integration with Labstep

### 2. **labstep-sentiment**
- **Purpose**: Analyze sentiment/themes in lab notes
- **Trigger**: Questions about trends or patterns in experiment notes
- **Dependency**: Requires `labstep` skill access

### 3. **pptx**
- **Purpose**: Generate and modify PowerPoint presentations
- **Trigger**: Requests to create slides, generate reports, or present data
- **Scripts**: Includes `/scripts` subdirectory with helper utilities

### 4. **sync-data**
- **Purpose**: Synchronize local SharePoint mirror via rsync
- **Trigger**: "sync sharepoint" or "update data mirror"
- **Mirror location**: `../experimental-patent/Sharepoint Mirror 25 Feb 2026/`

### 5. **sctipseq-data**
- **Purpose**: Query and analyze single-cell RNA-seq data
- **Trigger**: Questions about scRNA-seq datasets, analysis
- **Data source**: Local SharePoint mirror (kept in sync by `sync-data`)

---

## Security & Read-Only Policies

### Labstep (Read-Only)

**Standing rule: Agents will NOT create, update, or delete Labstep entries unless the user explicitly confirms with the phrase "confirm write".**

- The Labstep skill is configured for read-only access via a dedicated read-only service account
- The API token is scoped and monitored; any write attempts should be flagged
- **If you are asked to modify a Labstep entry**, ask the user for explicit confirmation:
  - "I can [describe the change]. To proceed, please confirm write: `confirm write`"

### SharePoint (Read-Only)

- **msgraph-mcp** integration enforces read-only at two layers:
  1. **OAuth token**: Only `Sites.Read.All` and `Files.Read.All` scopes — write operations are not possible at the API level
  2. **Skill surface**: Only read tools are exposed (`search_files`, `get_file_content`)
- Local mirror is kept in sync via `sync-data` skill; prefer querying local copies when available

---

## Labstep Credentials

The `labstep` skill uses a dedicated read-only service account:
- **Account**: `lab-agent-readonly@imperial.ac.uk`
- **Permission level**: Workspace **Viewer** role
- **API key location**: `.claude/settings.json` (configured by admin)

---

## Team Workflows

### Data Pipeline
1. Use `sync-data` skill to pull latest from SharePoint → local mirror
2. Query `sctipseq-data` or other analysis skills against local data
3. Use `pptx` skill to generate reports/presentations
4. All modifications happen locally or in user-controlled files; no write-back to Labstep or SharePoint without explicit user request + confirmation

### Lab Queries
1. Use `labstep` skill to fetch experiments/protocols
2. Use `labstep-sentiment` for trend analysis
3. Results are read-only; modifications require user confirmation

---

## Contacting Admin

If you encounter issues with Labstep or SharePoint access:
- Check `.claude/settings.json` for valid credentials
- Verify network connectivity to Labstep and Azure
- Report access errors to the lab tech lead

---

## Questions?

Each skill's `SKILL.md` file contains detailed documentation and examples.
