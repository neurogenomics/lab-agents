# Lab-Agents

A shared Claude Code environment for the Imperial College London Neurogenomics lab, with integrated access to Labstep (lab notebook), OneDrive/SharePoint (shared data files), and local data analysis tools.

## Skills

### 1. labstep

Query experiments, protocols, inventory, and other entities from the Labstep electronic lab notebook (read-only).

```
> Show me the last 10 experiments
> What protocols are linked to SK550?
> Search for experiments mentioning "lysis buffer"
> List all resources tagged "antibody"
```

### 2. labstep-sentiment

Analyse the sentiment and scientific outcome of Labstep experiments — whether they succeeded, failed, or were inconclusive.

```
> Analyse the sentiment of all experiments from January 2026
> Which experiments failed in the last 3 months?
> Give me a sentiment summary across all IVT optimisation experiments
> Export sentiment analysis to Excel
```

### 3. nucleic-acid-analysis

Read and extract RNA quantification data (Qubit, TapeStation, qPCR) from experiment folders on OneDrive.

```
> What are the Qubit concentrations for SK543?
> Show me the TapeStation results for SK134
> Compare RNA Qubit yields between SK443 and SK447
> Compare HiScribe vs RiboMax RNA yields across all IVT experiments
> Are there any TapeStation warnings or alerts in SK172?
> What are the TapeStation peak sizes for each sample in SK134?
```

### 4. read-from-sharepoint

Read-only access to lab data files from synced OneDrive (SharePoint) folders. Auto-discovers all Skene lab shared libraries.

```
> List all experiment folders in the scRNA-seq library
> Find the experiment folder for SK457
> What files are in the SK550 folder?
> Show me the Metadata.xlsx for SK297
```

### 5. experiment-summary

Generate a filled-in Experiment Summary `.docx` from Labstep metadata and OneDrive QC data. User-invocable via `/experiment-summary`.

```
> /experiment-summary SK550
> Write up an experiment summary for SK443
> Generate the experiment summary for "New lysis buffer test"
```

### 6. pptx

Generate and modify PowerPoint presentations from data or text.

```
> Create a slide deck summarising the IVT optimisation results
> Add a slide with the Qubit bar chart to my presentation
> Read the contents of results_deck.pptx
```

## Quick Start

### Prerequisites
- Claude Code CLI installed
- OneDrive syncing the Skene lab shared libraries
- Labstep API key configured in `.claude/settings.json`

### Setup

1. Follow the detailed setup guide in `SETUP_INSTRUCTIONS.md`
2. Add your Labstep API key to `.claude/settings.json`
3. Ensure OneDrive is syncing (System Settings > OneDrive)
4. Verify: `claude "What skills are available?"`

## Security

- **OneDrive is strictly read-only** — agents will never write, move, rename, or delete files under OneDrive paths
- **Labstep is read-only** — write operations require explicit user confirmation (`confirm write`)
- **All output files** are saved to the working directory, never to shared locations
- Credentials stored in `.claude/settings.json` (git-ignored)

## Folder Structure

```
lab-agents/
├── .claude/
│   ├── settings.json          # Credentials (git-ignored)
│   └── skills/
│       ├── labstep/
│       ├── labstep-sentiment/
│       ├── pptx/
│       ├── read-from-sharepoint/
│       ├── nucleic-acid-analysis/
│       └── experiment-summary/
├── support/                   # Templates (Experiment Summary Template.docx)
├── CLAUDE.md                  # Team policies & skill docs
├── SETUP_INSTRUCTIONS.md      # Detailed setup guide
├── SETUP_CHECKLIST.md         # Progress tracker
└── README.md                  # This file
```

## Documentation

- **`CLAUDE.md`** — Team policies, security guidelines, and skill reference
- **`SETUP_INSTRUCTIONS.md`** — Labstep and OneDrive setup guide
- **`.claude/skills/*/SKILL.md`** — Individual skill documentation

## Links

- [Neurogenomics Lab GitHub](https://github.com/neurogenomics)
- [Labstep Docs](https://docs.labstep.com/)
- [Claude Code Docs](https://docs.anthropic.com/en/docs/claude-code)
