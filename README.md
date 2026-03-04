# Lab-Agents

A shared Claude Code environment for the Imperial College London Neurogenomics lab, with integrated access to Labstep (lab notebook), OneDrive/SharePoint (shared data files), and local data analysis tools.

## Quick Start

### Prerequisites
- Claude Code CLI installed
- [APM](https://github.com/microsoft/apm) installed
- Labstep API key configured in `.claude/settings.json`

### Setup

1. Install skills with APM:
   ```bash
   apm install
   ```
2. Add your Labstep API key to `.claude/settings.json`
3. Verify: `claude "What skills are available?"`

## Skills

### 1. labstep

Query experiments, protocols, inventory, and other entities from the Labstep electronic lab notebook (read-only).

```
> Show me the last 10 experiments
> What protocols are linked to SK543?
> Search for experiments mentioning "lysis buffer"
> List all resources tagged "antibody"
```

### 2. labstep-sentiment

Summarise key findings and researcher thoughts from Labstep experiments.

```
> Summarise findings from all experiments in January 2026
> What were the key results from the IVT optimisation experiments?
> Give me a research summary across all lysis buffer experiments
> Export experiment summaries to Excel
```

### 3. nucleic-acid-analysis

Read and extract RNA quantification data (Qubit, TapeStation, qPCR) from experiment folders on OneDrive.

```
> What are the Qubit concentrations for SK443?
> Show me the TapeStation results for SK134
> Compare RNA Qubit yields between SK443 and SK447
> Compare HiScribe vs RiboMax RNA yields across all IVT experiments
> Are there any TapeStation warnings or alerts in SK172?
> What are the TapeStation peak sizes for each sample in SK297?
```

### 4. read-from-sharepoint

Read-only access to lab data files from synced OneDrive (SharePoint) folders. Auto-discovers all Skene lab shared libraries.

```
> List all experiment folders in the scRNA-seq library
> Find the experiment folder for SK457
> What files are in the SK443 folder?
> Show me the Metadata.xlsx for SK297
```

### 5. experiment-summary

Generate a filled-in Experiment Summary `.docx` from Labstep metadata and OneDrive QC data. User-invocable via `/experiment-summary`.

```
> /experiment-summary SK443
> Write up an experiment summary for SK297
> Generate the experiment summary for "IVT kit comparison"
```

### 6. pptx

Generate and modify PowerPoint presentations from data or text.

```
> Create a slide deck summarising the IVT optimisation results
> Add a slide with the Qubit bar chart to my presentation
> Read the contents of results_deck.pptx
```

## Links

- [Neurogenomics Lab GitHub](https://github.com/neurogenomics)
- [Labstep API Docs](https://apidoc.labstep.com)
- [Claude Code Docs](https://docs.anthropic.com/en/docs/claude-code)
