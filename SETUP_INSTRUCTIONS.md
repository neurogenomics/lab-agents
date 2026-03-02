# Lab-Agents Setup Instructions

## Part 1: Azure AD App Registration (for SharePoint MCP)

### Step-by-Step

1. **Open Azure Portal**
   - Go to https://portal.azure.com
   - Sign in with your Imperial Azure account

2. **Navigate to App Registrations**
   - Left sidebar → "App registrations"
   - Click "New registration"

3. **Register Application**
   - **Name**: `lab-agents-sharepoint-ro`
   - **Supported account types**: "Accounts in this organizational directory only"
   - Click "Register"

4. **Configure API Permissions**
   - You're now in the app's overview page
   - Left sidebar → "API permissions"
   - Click "Add a permission"
   - Select "Microsoft Graph"
   - Choose "Delegated permissions"
   - Search for and select:
     - [ ] `Sites.Read.All`
     - [ ] `Files.Read.All`
   - Click "Add permissions"
   - You'll see them listed (may show "Not granted for...")
   - Click "Grant admin consent for [your org]"
   - Confirm the prompt

5. **Create Client Secret**
   - Left sidebar → "Certificates & secrets"
   - Under "Client secrets", click "New client secret"
   - **Description**: `lab-agents-sharepoint`
   - **Expires**: Choose 24 months or longer
   - Click "Add"
   - **Copy the Value** (not the ID) — you'll paste this into settings.json
   - ⚠️ **Note**: You can only see this once! Save it somewhere safe

6. **Collect Your Credentials**
   - Left sidebar → "Overview"
   - Copy these values to a text file:
     - **Client ID**: (shown as "Application (client) ID")
     - **Tenant ID**: (shown as "Directory (tenant) ID")
     - **Client Secret**: (from step 5)

### Troubleshooting

- **"Grant admin consent" button is disabled**: Your account may not have admin rights. Contact your Azure admin.
- **Permissions show as "Not granted"**: Wait a few minutes after granting admin consent, or sign out and back in.
- **Can't find Microsoft Graph**: Make sure you're clicking "Microsoft Graph", not "Azure Service Management" or other options.

---

## Part 2: Labstep Read-Only Account Setup

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
   - You can test the token locally:
   ```bash
   curl -H "Authorization: Bearer YOUR_TOKEN" \
        https://labstep.com/api/v3/experiments?limit=1
   ```
   - Should return JSON with experiment data (no error)

### Troubleshooting

- **Can't add members**: Your account may need admin rights in Labstep. Contact your Labstep workspace admin.
- **Token not working**: Verify the user has Viewer access to at least one workspace.
- **"Unauthorized" error**: Double-check the token was copied completely (no extra spaces).

---

## Part 3: Fill in `.claude/settings.json`

Once you have all three Azure credentials and the Labstep token:

1. **Open `.claude/settings.json`** in your editor (in `/Users/jaymoore/Documents/JAY_PhD/imperial/lab-agents/`)

2. **Replace placeholders**:

   ```json
   {
     "mcpServers": {
       "sharepoint": {
         "command": "uvx",
         "args": ["msgraph-mcp"],
         "env": {
           "AZURE_CLIENT_ID": "YOUR_AZURE_CLIENT_ID_HERE",          // ← Replace
           "AZURE_TENANT_ID": "YOUR_AZURE_TENANT_ID_HERE",          // ← Replace
           "AZURE_CLIENT_SECRET": "YOUR_AZURE_CLIENT_SECRET_HERE"   // ← Replace
         }
       }
     },
     "skillsConfig": {
       "labstep": {
         "apiKey": "YOUR_LABSTEP_READONLY_API_KEY_HERE",             // ← Replace
         "baseUrl": "https://labstep.com/api/v3"
       }
     }
   }
   ```

3. **Save the file**

4. **Secure the file** (optional, but recommended):
   ```bash
   chmod 600 /Users/jaymoore/Documents/JAY_PhD/imperial/lab-agents/.claude/settings.json
   ```

---

## Part 4: Verification

Once you've filled in the credentials, cd into the lab-agents directory and test:

### Test 1: Skills Load
```bash
cd /Users/jaymoore/Documents/JAY_PhD/imperial/lab-agents
claude "What skills are available?"
```
**Expected**: Agent lists all 5 skills and their purposes

### Test 2: SharePoint MCP
```bash
claude "Can you search for a file named 'scRNA' in our SharePoint?"
```
**Expected**: Agent finds files in SharePoint (or says none found), not an auth error

### Test 3: Labstep Read
```bash
claude "Fetch the last 3 experiments from Labstep"
```
**Expected**: Agent retrieves experiments, not an auth error

### Test 4: Write Guard (Labstep)
```bash
claude "Create a new Labstep experiment called 'Test Experiment'"
```
**Expected**: Agent refuses OR asks for confirmation: "To proceed, please confirm write: `confirm write`"

### Test 5: Sync Data
```bash
claude "sync sharepoint"
```
**Expected**: Agent runs rsync and updates the local mirror

---

## Quick Reference

| Component | Credential | Where It Goes |
|-----------|-----------|---------------|
| Azure Client ID | Alphanumeric (36 chars) | `.claude/settings.json` → `mcpServers.sharepoint.env.AZURE_CLIENT_ID` |
| Azure Tenant ID | Alphanumeric (36 chars) | `.claude/settings.json` → `mcpServers.sharepoint.env.AZURE_TENANT_ID` |
| Azure Client Secret | Long string | `.claude/settings.json` → `mcpServers.sharepoint.env.AZURE_CLIENT_SECRET` |
| Labstep API Token | Alphanumeric string | `.claude/settings.json` → `skillsConfig.labstep.apiKey` |

---

## Next Steps

1. ✅ Follow Parts 1 & 2 above to set up Azure and Labstep
2. ✅ Fill in Part 3 (`.claude/settings.json`)
3. ✅ Run verification tests in Part 4
4. ✅ Share credentials with other team members (securely) if they need access
5. ✅ Update `SETUP_CHECKLIST.md` when complete

---

## Support

- **Azure issues**: Check the [Azure App Registration docs](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app)
- **Labstep issues**: See [Labstep API docs](https://docs.labstep.com/docs/getting-started)
- **msgraph-mcp issues**: See [GitHub repo](https://github.com/eesb99/msgraph-mcp)
- **General questions**: Check `CLAUDE.md` for team policies and skill documentation
