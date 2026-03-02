# Lab-Agents

A shared Claude agent environment for the Imperial College London Neurogenomics lab, with integrated access to Labstep (lab notebook), SharePoint (shared documents), and local data analysis tools.

## Features

- **5 Built-in Skills**:
  - `labstep`: Query experiments, protocols, and lab records (read-only)
  - `labstep-sentiment`: Analyze trends in lab notes
  - `pptx`: Generate and modify PowerPoint presentations
  - `sctipseq-data`: Query and analyze single-cell RNA-seq datasets
  - `sync-data`: Sync SharePoint data to local mirror

- **Read-Only Access**: All Labstep and SharePoint access is read-only by design
  - Write operations require explicit user confirmation
  - Azure AD app scoped to read-only permissions
  - Labstep uses dedicated viewer-only service account

- **Local Data Pipeline**: SharePoint data synchronized to local mirror for fast querying

## Quick Start

### Prerequisites
- Claude CLI installed (`pip install anthropic-cli` or equivalent)
- Azure AD app registration (for SharePoint access)
- Labstep read-only account setup

### Setup

1. **Follow the detailed setup guide**:
   ```bash
   # Read this first:
   cat SETUP_INSTRUCTIONS.md
   ```

2. **Register Azure AD app** for SharePoint (Part 1 of SETUP_INSTRUCTIONS.md)

3. **Create Labstep read-only account** (Part 2 of SETUP_INSTRUCTIONS.md)

4. **Fill in credentials**:
   ```bash
   # Edit this file with your credentials:
   vi .claude/settings.json
   ```

5. **Verify everything works**:
   ```bash
   cd /Users/jaymoore/Documents/JAY_PhD/imperial/lab-agents
   claude "What skills are available?"
   # Should list all 5 skills
   ```

## Documentation

- **`CLAUDE.md`** — Team policies, security guidelines, and skill documentation
- **`SETUP_INSTRUCTIONS.md`** — Step-by-step setup for Azure AD and Labstep
- **`SETUP_CHECKLIST.md`** — Progress tracking and verification tests
- **`.claude/skills/*/SKILL.md`** — Individual skill documentation

## Security

✅ **Read-only by design**:
- SharePoint access via `msgraph-mcp` with `Sites.Read.All`, `Files.Read.All` scopes only
- Labstep uses dedicated viewer-only service account
- All write operations require explicit user confirmation ("confirm write")
- Credentials stored in `.claude/settings.json` (git-ignored)

## Folder Structure

```
lab-agents/
├── .claude/
│   ├── settings.json          # Credentials (git-ignored)
│   └── skills/                # 5 available skills
│       ├── labstep/
│       ├── labstep-sentiment/
│       ├── pptx/
│       ├── sctipseq-data/
│       └── sync-data/
├── CLAUDE.md                  # Team policies
├── SETUP_INSTRUCTIONS.md      # Detailed setup guide
├── SETUP_CHECKLIST.md         # Progress tracker
├── README.md                  # This file
└── .gitignore                 # Protects sensitive files
```

## Team Usage

All agents in this environment inherit the same security policies:
- Skills are auto-triggered based on context
- SharePoint access is read-only (no downloads of confidential docs)
- Labstep access is read-only (no experiment modifications)
- Write operations require explicit confirmation with phrase: "confirm write"

See `CLAUDE.md` for full policies.

## Support & Troubleshooting

- **Azure setup issues**: See SETUP_INSTRUCTIONS.md Part 1
- **Labstep setup issues**: See SETUP_INSTRUCTIONS.md Part 2
- **Credential issues**: Check `.claude/settings.json` format
- **Skill-specific help**: See individual `SKILL.md` files in `.claude/skills/`

## Links

- [Imperial College London](https://www.imperial.ac.uk/)
- [Neurogenomics Lab GitHub](https://github.com/neurogenomics)
- [Labstep Docs](https://docs.labstep.com/)
- [Azure AD App Registration](https://docs.microsoft.com/en-us/azure/active-directory/develop/)
- [Claude Documentation](https://claude.ai/docs)

---

**Last updated**: March 2, 2026
