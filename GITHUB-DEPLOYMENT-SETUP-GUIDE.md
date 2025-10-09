# GitHub Deployment Setup Guide - EDI Platform to Azure

**Last Updated:** January 10, 2025  
**Status:** Ready for Execution  
**Prerequisites:** Azure App Registration credentials received from operations team

---

## Overview

This guide provides step-by-step instructions to configure GitHub for deploying the EDI Platform infrastructure (`edi-azure-infrastructure`) and all related services to Azure using GitHub Actions.

## What's New in This Guide (January 2025)

✅ **Updated Repository List** - Now includes all 10 actual EDI Platform repositories:
- `edi-azure-infrastructure` (Infrastructure as Code)
- `edi-platform-core` (Core shared libraries)
- `edi-mappers` (EDI mappers)
- `edi-sftp-connector` (SFTP connector)
- `edi-admin-portal-backend` (Admin API)
- `edi-admin-portal-frontend` (Admin UI)
- `edi-database-controlnumbers` (Control numbers DB)
- `edi-database-eventstore` (Event store DB)
- `edi-database-sftptracking` (SFTP tracking DB)
- `edi-partner-configs` (Partner configs)

✅ **Updated .NET Version** - Scripts now configure .NET 9.0.x (current platform version)

✅ **New Secrets Configuration Script** - Automated script for setting Azure credentials

✅ **Simplified Workflow** - Streamlined process with better error handling

---

## Prerequisites

### 1. Azure Credentials (from Operations Team)

You must have received the following from your operations/DevOps team:

| Credential | Example Format | Description |
|------------|----------------|-------------|
| **Tenant ID** | `76888a14-162d-4764-8e6f-c5a34addbd87` | Azure AD Tenant ID |
| **Client ID** | `12345678-abcd-1234-...` | App Registration Application ID |
| **Client Secret** | `abc123...` | App Registration Secret |
| **DEV Subscription ID** | `0f02cf19-be55-4aab-983b-951e84910121` | DEV Azure Subscription |
| **PROD Subscription ID** | `85aa9a59-7b1c-49d2-84ba-0640040bc097` | PROD Azure Subscription |

📋 **Checklist**: Before proceeding, confirm you have:
- [ ] Azure Tenant ID
- [ ] Azure Client ID (Application ID)
- [ ] Azure Client Secret
- [ ] DEV Subscription ID
- [ ] PROD Subscription ID
- [ ] Confirmation that App Registration has Contributor role on target resource groups

### 2. Local Environment

- [ ] **GitHub CLI (gh)** installed
  ```powershell
  # Check if installed
  gh --version
  
  # If not installed
  winget install GitHub.cli
  ```

- [ ] **GitHub CLI authenticated**
  ```powershell
  # Login
  gh auth login
  
  # Verify
  gh auth status
  ```

- [ ] **PowerShell 5.1 or higher**
  ```powershell
  $PSVersionTable.PSVersion
  ```

- [ ] **Admin permissions** on all target GitHub repositories

---

## Step 1: Configure GitHub Repository Variables

Variables are **non-sensitive** configuration values like resource names, Azure regions, and naming conventions.

### Option A: Automated Script (Recommended)

```powershell
# Navigate to scripts directory
cd c:\repos\edi-platform\scripts

# Run the configuration script (NEW VERSION with updated repos)
.\configure-github-variables-v2.ps1

# The script will:
# - Verify GitHub CLI is installed and authenticated
# - Check access to all 10 repositories
# - Prompt for confirmation before making changes
# - Configure 50+ variables across all repositories
# - Display a summary report
```

**Expected Output:**
```
═══ GitHub Variables Configuration for EDI Platform ═══
Target Organization: PointCHealth
Target Repositories: edi-azure-infrastructure, edi-platform-core, ...
Variables to Configure: 50

═══ Pre-flight Validation ═══
✅ GitHub CLI found: gh version 2.40.0
✅ Authenticated to GitHub

═══ Repository Access Verification ═══
✅ Admin access confirmed for edi-azure-infrastructure
✅ Admin access confirmed for edi-platform-core
...

Proceed with variable configuration? (Y/N): Y

═══ Configuring edi-azure-infrastructure ═══
✅ Set AZURE_LOCATION
✅ Set AZURE_LOCATION_SHORT
...
```

### Option B: Manual Configuration

If you prefer to configure specific repositories or troubleshoot issues:

```powershell
# Configure only edi-azure-infrastructure
.\configure-github-variables-v2.ps1 -Repository edi-azure-infrastructure

# Skip validation checks (if you've already verified)
.\configure-github-variables-v2.ps1 -SkipValidation
```

### Option C: Python Script (Alternative)

```powershell
# Using Python script (also updated)
cd c:\repos\edi-platform\scripts
python configure-github-variables.py

# Configure specific repository
python configure-github-variables.py --repo edi-azure-infrastructure

# Dry run (see what would be done without making changes)
python configure-github-variables.py --dry-run
```

### Verification

```powershell
# List all variables in a repository
gh variable list --repo PointCHealth/edi-azure-infrastructure

# Get specific variable value
gh variable get AZURE_LOCATION --repo PointCHealth/edi-azure-infrastructure
```

**Expected Variables (50 total):**
- Azure Core: `AZURE_LOCATION`, `AZURE_LOCATION_SHORT`, `PROJECT_NAME`, etc.
- Resource Groups: `DEV_RESOURCE_GROUP`, `TEST_RESOURCE_GROUP`, `PROD_RESOURCE_GROUP`
- Resource Prefixes: `STORAGE_ACCOUNT_PREFIX`, `FUNCTION_APP_PREFIX`, etc.
- Service Bus: `SB_QUEUE_INBOUND`, `SB_TOPIC_ROUTING`, etc.
- Build Config: `DOTNET_VERSION` (9.0.x), `NODE_VERSION`, `BICEP_VERSION`
- Monitoring: `LOG_RETENTION_DAYS_DEV/TEST/PROD`
- Tagging: `TAG_ENVIRONMENT_DEV`, `TAG_PROJECT`, etc.

---

## Step 2: Configure GitHub Repository Secrets

Secrets are **sensitive** credentials that should never be committed to code.

### NEW: Automated Secrets Configuration Script

```powershell
# Navigate to scripts directory
cd c:\repos\edi-platform\scripts

# Run the secrets configuration script
.\configure-github-secrets.py

# The script will:
# 1. Verify GitHub CLI and authentication
# 2. Check access to all repositories
# 3. Securely prompt for Azure credentials
# 4. Set secrets in all repositories
# 5. Display summary (secrets are never shown in output)
```

**Interactive Prompts:**
```
═══ Azure Credentials Input ═══

Please provide Azure App Registration credentials from operations team
⚠️  These values are sensitive - ensure you're in a secure environment

Azure Tenant ID (format: 76888a14-162d-4764-8e6f-c5a34addbd87): [enter value]
Azure Client ID / Application ID: [enter value]
Azure Client Secret: [masked input]
DEV Subscription ID: [enter value]
PROD Subscription ID: [enter value]

✅ All credentials provided

Credentials Summary (masked):
  Tenant ID: 76888a14...
  Client ID: 12345678...
  Client Secret: ********
  DEV Subscription: 0f02cf19...
  PROD Subscription: 85aa9a59...

Proceed with secret configuration for 10 repositories? (Y/N):
```

### Configure Specific Repository Only

```powershell
# Configure only edi-azure-infrastructure
.\configure-github-secrets.py -Repository edi-azure-infrastructure
```

### Manual Secret Configuration

If you prefer manual setup or need to troubleshoot:

```powershell
# Set individual secrets
gh secret set AZURE_TENANT_ID --repo PointCHealth/edi-azure-infrastructure
# Paste value when prompted

gh secret set AZURE_CLIENT_ID --repo PointCHealth/edi-azure-infrastructure
gh secret set AZURE_CLIENT_SECRET --repo PointCHealth/edi-azure-infrastructure
gh secret set AZURE_SUBSCRIPTION_ID_DEV --repo PointCHealth/edi-azure-infrastructure
gh secret set AZURE_SUBSCRIPTION_ID_PROD --repo PointCHealth/edi-azure-infrastructure
```

### Verification

```powershell
# List secrets (values are hidden)
gh secret list --repo PointCHealth/edi-azure-infrastructure
```

**Expected Output:**
```
AZURE_CLIENT_ID         Updated 2025-01-10
AZURE_CLIENT_SECRET     Updated 2025-01-10
AZURE_SUBSCRIPTION_ID_DEV   Updated 2025-01-10
AZURE_SUBSCRIPTION_ID_PROD  Updated 2025-01-10
AZURE_TENANT_ID         Updated 2025-01-10
```

---

## Step 3: Test Azure Authentication

Create a simple test workflow to verify Azure authentication works before deploying actual infrastructure.

### Create Test Workflow

```powershell
# Navigate to edi-azure-infrastructure repository
cd c:\repos\edi-platform\edi-azure-infrastructure

# Create workflows directory if it doesn't exist
New-Item -ItemType Directory -Force -Path .github\workflows
```

Create `.github/workflows/test-azure-auth.yml`:

```yaml
name: Test Azure Authentication

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  test-auth:
    name: Test Azure Authentication
    runs-on: ubuntu-latest
    
    steps:
      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: |
            {
              "clientId": "${{ secrets.AZURE_CLIENT_ID }}",
              "clientSecret": "${{ secrets.AZURE_CLIENT_SECRET }}",
              "subscriptionId": "${{ secrets.AZURE_SUBSCRIPTION_ID_DEV }}",
              "tenantId": "${{ secrets.AZURE_TENANT_ID }}"
            }
      
      - name: Verify Authentication
        run: |
          echo "Testing Azure CLI authentication..."
          az account show
          az group list --query "[?name=='${{ vars.DEV_RESOURCE_GROUP }}']" --output table
      
      - name: Test Summary
        run: |
          echo "✅ Azure authentication successful!" >> $GITHUB_STEP_SUMMARY
          echo "✅ Can access DEV subscription" >> $GITHUB_STEP_SUMMARY
          echo "✅ Can list resource groups" >> $GITHUB_STEP_SUMMARY
```

### Run Test Workflow

```powershell
# Commit and push
git add .github/workflows/test-azure-auth.yml
git commit -m "Add Azure authentication test workflow"
git push

# Run workflow manually
gh workflow run test-azure-auth.yml --repo PointCHealth/edi-azure-infrastructure

# Watch workflow run
gh run watch

# Or view in browser
gh workflow view test-azure-auth.yml --web
```

### Expected Results

✅ **Success**: Workflow completes with green checkmarks
- Azure Login step succeeds
- `az account show` displays subscription details
- `az group list` returns resource groups (or empty if not created yet)

❌ **Failure Scenarios**:

| Error | Cause | Solution |
|-------|-------|----------|
| `AADSTS7000215: Invalid client secret` | Wrong secret or expired | Verify secret value, request new one from operations |
| `AADSTS70001: Application not found` | Wrong Client ID | Verify Client ID matches App Registration |
| `AuthorizationFailed` | Missing permissions | Contact operations to grant Contributor role |
| `Secret not found` | Secret not configured | Re-run `configure-github-secrets.py` |

---

## Step 4: Create Infrastructure Deployment Workflow

Once authentication test passes, create the actual infrastructure deployment workflow.

### Basic Infrastructure Workflow

Create `.github/workflows/deploy-infrastructure.yml`:

```yaml
name: Deploy Infrastructure

on:
  push:
    branches: [main]
    paths:
      - 'bicep/**'
      - '.github/workflows/deploy-infrastructure.yml'
  pull_request:
    branches: [main]
    paths:
      - 'bicep/**'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'dev'
        type: choice
        options:
          - dev
          - test
          - prod

permissions:
  id-token: write
  contents: read

jobs:
  validate:
    name: Validate Bicep
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Bicep CLI
        run: |
          curl -Lo bicep https://github.com/Azure/bicep/releases/latest/download/bicep-linux-x64
          chmod +x ./bicep
          sudo mv ./bicep /usr/local/bin/bicep
          bicep --version
      
      - name: Validate Bicep Templates
        run: |
          bicep build bicep/main.bicep --outfile main.json
          echo "✅ Bicep validation passed"
  
  deploy-dev:
    name: Deploy to DEV
    runs-on: ubuntu-latest
    needs: validate
    if: github.event_name == 'push' || (github.event_name == 'workflow_dispatch' && github.event.inputs.environment == 'dev')
    environment: dev
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: |
            {
              "clientId": "${{ secrets.AZURE_CLIENT_ID }}",
              "clientSecret": "${{ secrets.AZURE_CLIENT_SECRET }}",
              "subscriptionId": "${{ secrets.AZURE_SUBSCRIPTION_ID_DEV }}",
              "tenantId": "${{ secrets.AZURE_TENANT_ID }}"
            }
      
      - name: Deploy Bicep
        uses: azure/arm-deploy@v1
        with:
          subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID_DEV }}
          resourceGroupName: ${{ vars.DEV_RESOURCE_GROUP }}
          template: ./bicep/main.bicep
          parameters: >
            environment=dev
            location=${{ vars.AZURE_LOCATION }}
            projectName=${{ vars.PROJECT_NAME }}
          failOnStdErr: false
      
      - name: Deployment Summary
        run: |
          echo "✅ Infrastructure deployed to DEV" >> $GITHUB_STEP_SUMMARY
          echo "Resource Group: ${{ vars.DEV_RESOURCE_GROUP }}" >> $GITHUB_STEP_SUMMARY
          echo "Location: ${{ vars.AZURE_LOCATION }}" >> $GITHUB_STEP_SUMMARY
```

---

## Step 5: Configure GitHub Environments (Optional but Recommended)

Environments provide deployment protection rules for production.

### Create Environments via GitHub CLI

```powershell
# Note: Environment creation requires repository admin access via GitHub UI
# The gh CLI doesn't support creating environments yet

# Navigate to repository settings in browser:
gh repo view PointCHealth/edi-azure-infrastructure --web
# Then go to Settings > Environments > New environment
```

### Environment Configuration

#### Development Environment
- **Name:** `dev`
- **Protection Rules:** None (auto-deploy)
- **Secrets:** None (uses repository secrets)
- **Variables:** 
  - `ENVIRONMENT` = `dev`
  - `RESOURCE_GROUP` = `rg-edi-dev-eastus2`

#### Test Environment
- **Name:** `test`
- **Protection Rules:**
  - ✅ Required reviewers: 1
  - ✅ Wait timer: 0 minutes
- **Deployment branches:** `main` only

#### Production Environment
- **Name:** `prod`
- **Protection Rules:**
  - ✅ Required reviewers: 2
  - ✅ Wait timer: 5 minutes
  - ✅ Prevent administrators from bypassing
- **Deployment branches:** `main` and `hotfix/*`

---

## Step 6: Verification Checklist

Before considering setup complete, verify:

### Variables
- [ ] Run: `gh variable list --repo PointCHealth/edi-azure-infrastructure`
- [ ] Verify at least 50 variables are configured
- [ ] Check critical variables: `AZURE_LOCATION`, `DEV_RESOURCE_GROUP`, `DOTNET_VERSION`

### Secrets
- [ ] Run: `gh secret list --repo PointCHealth/edi-azure-infrastructure`
- [ ] Verify 5 secrets exist: `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_SUBSCRIPTION_ID_DEV`, `AZURE_SUBSCRIPTION_ID_PROD`

### Authentication
- [ ] Test workflow (`test-azure-auth.yml`) runs successfully
- [ ] Azure Login step completes without errors
- [ ] Can list resource groups in DEV subscription

### Workflows
- [ ] Infrastructure deployment workflow exists
- [ ] Workflow triggers on push to main and manual dispatch
- [ ] Workflow uses repository variables correctly (`${{ vars.AZURE_LOCATION }}`)
- [ ] Workflow uses repository secrets correctly (`${{ secrets.AZURE_CLIENT_ID }}`)

### All Repositories
- [ ] `edi-azure-infrastructure` - Variables ✓, Secrets ✓
- [ ] `edi-platform-core` - Variables ✓, Secrets ✓
- [ ] `edi-mappers` - Variables ✓, Secrets ✓
- [ ] `edi-sftp-connector` - Variables ✓, Secrets ✓
- [ ] `edi-admin-portal-backend` - Variables ✓, Secrets ✓
- [ ] `edi-admin-portal-frontend` - Variables ✓, Secrets ✓
- [ ] `edi-database-controlnumbers` - Variables ✓, Secrets ✓
- [ ] `edi-database-eventstore` - Variables ✓, Secrets ✓
- [ ] `edi-database-sftptracking` - Variables ✓, Secrets ✓
- [ ] `edi-partner-configs` - Variables ✓, Secrets ✓

---

## Troubleshooting

### Issue: Script fails with "Repository not found"

**Cause:** Repository name doesn't match or you don't have access

**Solution:**
```powershell
# Verify repository exists
gh repo view PointCHealth/edi-azure-infrastructure

# Check your access level
gh api /repos/PointCHealth/edi-azure-infrastructure --jq .permissions
```

### Issue: "Permission denied" when setting variables/secrets

**Cause:** You don't have admin permissions on the repository

**Solution:** Request admin access from repository owner or organization admin

### Issue: Azure Login fails with "Invalid client secret"

**Cause:** Secret value is incorrect or has expired

**Solution:**
1. Verify the secret value with operations team
2. Check if the App Registration secret has expired in Azure Portal
3. Request new secret if needed
4. Re-run `configure-github-secrets.py`

### Issue: Variables not appearing in workflow

**Cause:** Variable names are case-sensitive or misspelled

**Solution:**
```yaml
# ✅ Correct
${{ vars.AZURE_LOCATION }}

# ❌ Incorrect
${{ vars.azure_location }}
${{ vars.AZURELOCATION }}
```

### Issue: Python script fails with exit code 1

**Cause:** Missing GitHub CLI or authentication

**Solution:**
```powershell
# Install GitHub CLI
winget install GitHub.cli

# Authenticate
gh auth login

# Re-run script
python configure-github-variables.py
```

---

## Next Steps

After completing this setup:

1. **Deploy Infrastructure**
   - Run infrastructure deployment workflow
   - Verify Azure resources are created
   - Review deployment logs in Azure Portal

2. **Configure Additional Repositories**
   - Repeat process for other repositories (mappers, connectors, databases)
   - Use same Azure credentials for all repositories

3. **Set Up CI/CD Pipelines**
   - Create build workflows for .NET projects
   - Create deployment workflows for Azure Functions
   - Set up database migration workflows

4. **Monitor Deployments**
   - Set up Azure Application Insights
   - Configure GitHub Actions notifications
   - Review deployment metrics

---

## Related Documentation

- **[GITHUB-SECRETS-CONFIGURATION.md](./GITHUB-SECRETS-CONFIGURATION.md)** - Detailed secrets configuration
- **[20-github-actions-setup.md](./20-github-actions-setup.md)** - Complete GitHub Actions setup
- **[21-cicd-workflows.md](./21-cicd-workflows.md)** - CI/CD workflow examples
- **[22-deployment-procedures.md](./22-deployment-procedures.md)** - Deployment procedures

---

## Security Best Practices

✅ **DO:**
- Store credentials in GitHub Secrets, never in code
- Use environment protection rules for production
- Rotate secrets quarterly
- Enable secret scanning in repository settings
- Use managed identities where possible (future enhancement with OIDC)

❌ **DON'T:**
- Commit secrets to Git
- Log secret values in workflow output
- Share secrets via email or chat
- Use the same credentials across environments

---

## Support

**For Setup Issues:**
- Check troubleshooting section above
- Review GitHub Actions workflow logs
- Verify Azure credentials with operations team

**For Azure Access Issues:**
- Contact operations/DevOps team
- Reference App Registration details
- Verify service principal has required roles

**For Workflow Issues:**
- Check workflow syntax: `gh workflow list`
- View workflow runs: `gh run list`
- Check specific run: `gh run view <run-id>`

---

**Document Version:** 2.0  
**Last Updated:** January 10, 2025  
**Next Review:** After first successful deployment

