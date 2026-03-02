# Lab-Agents Setup Instructions

## Part 1: Labstep Read-Only Account Setup

### Step-by-Step

1. **Create New Labstep User**
   - Log in to Labstep as an admin
   - Settings → Team members → "Add a member"
   - **Full name**: `Lab Agent Readonly`
   - **Email**: Create a shared inbox or use a generic email like `lab-agent-readonly@imperial.ac.uk`
   - Click "Send invite"
   - **Accept the invite** (you may need to do this from the email account)

2. **Set Workspace Permissions**
   - In Labstep admin settings → Workspaces
   - For each workspace the agent should access:
     - Add the new user
     - Set their role to **Viewer** (not Editor, not Admin)
     - Save

3. **Generate API Token**
   - From the `lab-agent-readonly` user account:
     - Settings (top right) → "API tokens"
     - Click "Generate token"
     - **Name**: `lab-agents-readonly`
     - **Copy the token** — you'll paste this into settings.json
     - ⚠️ **Note**: Store this securely; it won't be shown again

4. **Verify Token Works (Optional)**
   ```bash
   curl -H "Authorization: Bearer YOUR_TOKEN" \
        https://labstep.com/api/v3/experiments?limit=1
   ```
   Should return JSON with experiment data (no error)

### Troubleshooting

- **Can't add members**: Your account may need admin rights in Labstep. Contact your Labstep workspace admin.
- **Token not working**: Verify the user has Viewer access to at least one workspace.
- **"Unauthorized" error**: Double-check the token was copied completely (no extra spaces).

---

## Part 2: OneDrive Setup

Ensure OneDrive is installed and syncing the Skene lab shared libraries:

1. **Install OneDrive** (if not already installed)
   - macOS: Download from the Mac App Store
   - Windows: Pre-installed on Windows 10/11

2. **Sign in** with your Imperial College London account

3. **Sync shared libraries**
   - In OneDrive, go to "Shared libraries"
   - Find and sync folders prefixed with `Skene lab -` (e.g. `Skene lab - WB - 07 scRNA-seq`)
   - These will appear under:
     - macOS: `~/Library/CloudStorage/OneDrive-SharedLibraries-ImperialCollegeLondon/`
     - Windows: `~/OneDrive - Imperial College London/`

4. **Verify sync**
   - Check System Settings > OneDrive for sync status
   - Confirm experiment folders (e.g. `SK550_...`) appear locally

---

## Part 3: Fill in `.claude/settings.json`

Once you have the Labstep token:

1. **Open `.claude/settings.json`** in your editor

2. **Replace the placeholder**:

   ```json
   {
     "skillsConfig": {
       "labstep": {
         "apiKey": "YOUR_LABSTEP_READONLY_API_KEY_HERE",
         "email": "lab-agent-readonly@imperial.ac.uk",
         "baseUrl": "https://labstep.com/api/v3"
       }
     }
   }
   ```

3. **Save the file**

4. **Secure the file** (optional, but recommended):
   ```bash
   chmod 600 .claude/settings.json
   ```

---

## Part 4: Verification

Once you've filled in the credentials, test the following:

### Test 1: Skills Load
```bash
cd /path/to/lab-agents
claude "What skills are available?"
```
**Expected**: Agent lists all 6 skills and their purposes

### Test 2: Labstep Read
```bash
claude "Fetch the last 3 experiments from Labstep"
```
**Expected**: Agent retrieves experiments, not an auth error

### Test 3: OneDrive Data
```bash
claude "What are the Qubit concentrations for SK550?"
```
**Expected**: Agent reads Metadata.xlsx from OneDrive and returns values

### Test 4: Write Guard (Labstep)
```bash
claude "Create a new Labstep experiment called 'Test Experiment'"
```
**Expected**: Agent refuses OR asks for confirmation: "To proceed, please confirm write: `confirm write`"

---

## Quick Reference

| Component | Credential | Where It Goes |
|-----------|-----------|---------------|
| Labstep API Token | Alphanumeric string | `.claude/settings.json` → `skillsConfig.labstep.apiKey` |

---

## Support

- **Labstep issues**: See [Labstep API docs](https://docs.labstep.com/docs/getting-started)
- **OneDrive sync issues**: System Settings > OneDrive > sync status
- **General questions**: Check `CLAUDE.md` for team policies and skill documentation
