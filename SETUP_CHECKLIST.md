# Lab-Agents Environment Setup Checklist

## ✅ Completed

- [x] **Skills** — All 6 skills configured:
  - ✅ `labstep/`
  - ✅ `labstep-sentiment/`
  - ✅ `pptx/` (with scripts/ subdirectory)
  - ✅ `read-from-sharepoint/`
  - ✅ `nucleic-acid-analysis/`
  - ✅ `experiment-summary/`

- [x] **CLAUDE.md** — Team policies and skill documentation

- [x] **settings.json template** — Placeholders ready for credentials

---

## 🔄 Setup Steps

### Labstep Read-Only Account
- [ ] Create new Labstep user (e.g., `lab-agent-readonly@imperial.ac.uk`)
- [ ] Add to workspaces with **Viewer** role only
- [ ] Generate API token
- [ ] Copy token and fill in `.claude/settings.json`:
  - `skillsConfig.labstep.apiKey`

### OneDrive Sync
- [ ] OneDrive installed and signed in with Imperial account
- [ ] Skene lab shared libraries syncing locally
- [ ] Experiment folders visible (e.g. `SK550_...`)

**Reference**: See SETUP_INSTRUCTIONS.md for detailed steps

---

## 🧪 Verification (After Setup)

1. **Skills trigger correctly**
   ```
   claude "List the skills available here"
   ```

2. **Labstep read-only works**
   ```
   claude "Fetch the last 3 experiments from Labstep"
   ```

3. **OneDrive data readable**
   ```
   claude "What are the Qubit concentrations for SK550?"
   ```

4. **Labstep write guard works**
   ```
   claude "Create a new experiment in Labstep called 'Test'"
   # Should prompt for confirmation: "confirm write"
   ```

---

## 📝 Notes

- OneDrive data is accessed directly via locally-synced folders (no API needed)
- All write operations require explicit user confirmation
- Each skill has its own `SKILL.md` with examples and detailed documentation
- See `CLAUDE.md` for full team policies

---

## Support

If you hit issues:
1. Check `.claude/settings.json` for valid credential format
2. Verify Labstep account has Viewer role access
3. Verify OneDrive sync status (System Settings > OneDrive)
4. Review individual skill `SKILL.md` files for troubleshooting
