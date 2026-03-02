# Lab-Agents Environment Setup Checklist

## ✅ Completed

- [x] **Step 1: Copy Skills** — All 5 skills copied from experimental-patent:
  - ✅ `labstep/`
  - ✅ `labstep-sentiment/`
  - ✅ `pptx/` (with scripts/ subdirectory)
  - ✅ `sctipseq-data/`
  - ✅ `sync-data/`

- [x] **Step 4: Create CLAUDE.md** — Team policies and skill documentation

- [x] **Step 2 & 3: Create settings.json template** — Placeholders ready for credentials

---

## 🔄 In Progress (Requires Your Action)

### Azure AD App Registration
- [ ] Register app in Azure Portal
- [ ] Set delegated permissions: `Sites.Read.All`, `Files.Read.All`
- [ ] Grant admin consent
- [ ] Generate client secret
- [ ] Copy credentials and fill in `.claude/settings.json`:
  - `AZURE_CLIENT_ID`
  - `AZURE_TENANT_ID`
  - `AZURE_CLIENT_SECRET`

**Reference**: See SETUP_INSTRUCTIONS.md for detailed steps

### Labstep Read-Only Account
- [ ] Create new Labstep user (e.g., `lab-agent-readonly@imperial.ac.uk`)
- [ ] Add to workspaces with **Viewer** role only
- [ ] Generate API token
- [ ] Copy token and fill in `.claude/settings.json`:
  - `skillsConfig.labstep.apiKey`

**Reference**: See SETUP_INSTRUCTIONS.md for detailed steps

---

## 🧪 Verification (After Credentials Added)

Once you've filled in the credentials, test the following:

1. **Skills trigger correctly**
   ```
   cd /Users/jaymoore/Documents/JAY_PhD/imperial/lab-agents
   claude "List the skills available here"
   ```

2. **SharePoint MCP works**
   ```
   claude "Search for a file in our SharePoint"
   ```

3. **Labstep read-only works**
   ```
   claude "Fetch experiment ID 12345 from Labstep"
   ```

4. **Labstep write guard works**
   ```
   claude "Create a new experiment in Labstep called 'Test'"
   # Should prompt for confirmation: "confirm write"
   ```

5. **sync-data skill works**
   ```
   claude "sync sharepoint"
   # Should update local mirror
   ```

---

## 📝 Notes

- Local SharePoint mirror is at: `../experimental-patent/Sharepoint Mirror 25 Feb 2026/`
- All write operations require explicit user confirmation
- Each skill has its own `SKILL.md` with examples and detailed documentation
- See `CLAUDE.md` for full team policies

---

## Support

If you hit issues:
1. Check `.claude/settings.json` for valid credential format
2. Verify Azure app permissions in Azure Portal
3. Verify Labstep account has Viewer role access
4. Check network connectivity to Labstep and Azure
5. Review individual skill `SKILL.md` files for troubleshooting
