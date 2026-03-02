---
name: sync-data
description: >
  Outputs rsync commands to sync SharePoint (OneDrive) data to the local
  experimental-patent mirror. Use this skill whenever the user mentions
  syncing data, updating the mirror, syncing SharePoint, or runs /sync-data.
  Trigger on phrases like "sync data", "sync sharepoint", "update the mirror",
  "pull latest data", "rsync sharepoint".
---

Output these two rsync commands for the user to run in their terminal:

```bash
rsync -av --progress \
  '/Users/jaymoore/Library/CloudStorage/OneDrive-SharedLibraries-ImperialCollegeLondon/Skene lab - WB - 03 scTIP-Seq Development/' \
  '/Users/jaymoore/Documents/JAY_PhD/imperial/experimental-patent/Sharepoint Mirror 25 Feb 2026/03 scTIP-Seq Development/'
```

```bash
rsync -av --progress \
  '/Users/jaymoore/Library/CloudStorage/OneDrive-SharedLibraries-ImperialCollegeLondon/Skene lab - WB - 07 scRNA-seq/' \
  '/Users/jaymoore/Documents/JAY_PhD/imperial/experimental-patent/Sharepoint Mirror 25 Feb 2026/07 scRNA-seq/'
```

Briefly note: run them sequentially, and make sure OneDrive is synced first before running so all files are locally available.
