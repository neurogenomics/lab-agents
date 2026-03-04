---
description: Security and access control policies for lab data systems
applyTo: "**/*"
author: Jay Moore
version: 1.0.0
---

# Security & Access Control

## Labstep (Read-Only)

Agents will NOT create, update, or delete Labstep entries unless the user explicitly confirms with the phrase "confirm write".

- The Labstep skill uses a dedicated read-only service account (`lab-agent-readonly@imperial.ac.uk`)
- API token is scoped and monitored; flag any write attempts
- If asked to modify a Labstep entry, request explicit confirmation:
  "I can [describe the change]. To proceed, please confirm write: `confirm write`"

## OneDrive / SharePoint (Strictly Read-Only)

**NEVER write, move, rename, copy-into, or delete ANY file under OneDrive paths.**

These folders are shared team resources synced from SharePoint. Any modification propagates to all team members and risks data loss.

- **Allowed**: `open("r")`, `pd.read_excel()`, `pd.read_csv()`, `Document()` (read-only load), `ls`, `find`, `stat`
- **Forbidden**: Any write, move, rename, delete, or copy-into operation targeting OneDrive paths
- **All output files** must be saved to the **working directory**, never to OneDrive
- **Retry on failure**: OneDrive files may be cloud-only placeholders — use retry logic with delays (2s, 5s, 10s)
