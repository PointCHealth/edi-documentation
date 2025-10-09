# GitHub Secrets and Variables Configuration Guide

## Purpose
This document provides instructions for configuring GitHub repository secrets and variables after Azure access has been granted by the operations team.

**Prerequisites:** Operations team must complete the steps in `OPERATIONS-AZURE-ACCESS-REQUEST.md` first and provide the required Azure credentials.

---

## Overview

Once operations provides the Azure Service Principal credentials, the development team needs to configure GitHub secrets and variables across all EDI Platform repositories to enable automated deployments.

## Required Information from Operations

You will receive the following from the operations team:

| Item | Description | Example |
|------|-------------|---------|
| **AZURE_TENANT_ID** | Azure AD Tenant ID | `76888a14-162d-4764-8e6f-c5a34addbd87` |
| **AZURE_CLIENT_ID** | Service Principal Application ID | `12345678-abcd-...` |
| **AZURE_CLIENT_SECRET** | Service Principal Secret | `abc123...` |
| **AZURE_SUBSCRIPTION_ID_DEV** | DEV Subscription ID | `0f02cf19-be55-4aab-983b-951e84910121` |
| **AZURE_SUBSCRIPTION_ID_PROD** | PROD Subscription ID | `85aa9a59-7b1c-49d2-84ba-0640040bc097` |

---

## Repositories Requiring Configuration

The following repositories need secrets and variables configured:

1. **PointCHealth/edi-azure-infrastructure** - Infrastructure as Code
2. **PointCHealth/edi-sftp-connector** - SFTP ingestion function
3. **PointCHealth/edi-platform-core** - Core processing functions
4. **PointCHealth/edi-mappers** - EDI mapping functions
5. **PointCHealth/edi-database-controlnumbers** - Control numbers database migrations
6. **PointCHealth/edi-database-eventstore** - Event store database migrations
7. **PointCHealth/edi-database-sftptracking** - SFTP tracking database migrations

---

## Method 1: Automated Configuration (Recommended)

### Using PowerShell Script

Use the existing configuration script to set up all repositories at once:

```powershell
# Navigate to scripts directory
cd c:\repos\edi-platform\scripts

# Run the configuration script
.\configure-github-variables.py

# Follow the prompts to enter the Azure credentials provided by operations
```

The script will:
- ✅ Configure secrets in all 7 repositories
- ✅ Set up repository variables
- ✅ Verify GitHub CLI authentication
- ✅ Validate that all repositories are accessible

---

## Method 2: Manual Configuration

### Step 1: Configure Repository Secrets

For **each repository listed above**, add the following secrets:

**Navigate to:** Repository → Settings → Secrets and variables → Actions → Secrets

#### Using GitHub CLI

```powershell
# Set variables
$tenantId = "76888a14-162d-4764-8e6f-c5a34addbd87"  # From operations
$clientId = "your-client-id-from-operations"
$clientSecret = "your-client-secret-from-operations"
$devSubId = "0f02cf19-be55-4aab-983b-951e84910121"
$prodSubId = "85aa9a59-7b1c-49d2-84ba-0640040bc097"

# Configure each repository
$repos = @(
    "PointCHealth/edi-azure-infrastructure",
    "PointCHealth/edi-sftp-connector",
    "PointCHealth/edi-platform-core",
    "PointCHealth/edi-mappers",
    "PointCHealth/edi-database-controlnumbers",
    "PointCHealth/edi-database-eventstore",
    "PointCHealth/edi-database-sftptracking"
)

foreach ($repo in $repos) {
    Write-Host "Configuring $repo..." -ForegroundColor Cyan
    
    # Add secrets
    gh secret set AZURE_TENANT_ID --body "$tenantId" --repo $repo
    gh secret set AZURE_CLIENT_ID --body "$clientId" --repo $repo
    gh secret set AZURE_CLIENT_SECRET --body "$clientSecret" --repo $repo
    gh secret set AZURE_SUBSCRIPTION_ID_DEV --body "$devSubId" --repo $repo
    gh secret set AZURE_SUBSCRIPTION_ID_PROD --body "$prodSubId" --repo $repo
    
    Write-Host "✅ $repo secrets configured" -ForegroundColor Green
}
```

#### Using GitHub Web UI

For each repository:

1. Go to **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Add each secret:

| Secret Name | Value Source |
|-------------|--------------|
| `AZURE_TENANT_ID` | From operations team |
| `AZURE_CLIENT_ID` | From operations team |
| `AZURE_CLIENT_SECRET` | From operations team |
| `AZURE_SUBSCRIPTION_ID_DEV` | From operations team |
| `AZURE_SUBSCRIPTION_ID_PROD` | From operations team |

### Step 2: Configure Repository Variables

For **each repository**, add the following variables:

**Navigate to:** Repository → Settings → Secrets and variables → Actions → Variables

#### Using GitHub CLI

```powershell
foreach ($repo in $repos) {
    Write-Host "Configuring variables for $repo..." -ForegroundColor Cyan
    
    # Add variables
    gh variable set DEV_RESOURCE_GROUP --body "rg-edi-dev-eastus2" --repo $repo
    gh variable set TEST_RESOURCE_GROUP --body "rg-edi-test-eastus2" --repo $repo
    gh variable set PROD_RESOURCE_GROUP --body "rg-edi-prod-eastus2" --repo $repo
    gh variable set AZURE_LOCATION --body "eastus2" --repo $repo
    gh variable set PROJECT_NAME --body "edi-platform" --repo $repo
    
    Write-Host "✅ $repo variables configured" -ForegroundColor Green
}

Write-Host "`n✅ All repositories configured successfully!" -ForegroundColor Green
```

#### Using GitHub Web UI

For each repository:

1. Go to **Settings** → **Secrets and variables** → **Actions** → **Variables** tab
2. Click **New repository variable**
3. Add each variable:

| Variable Name | Value | Purpose |
|---------------|-------|---------|
| `DEV_RESOURCE_GROUP` | `rg-edi-dev-eastus2` | Dev deployment target |
| `TEST_RESOURCE_GROUP` | `rg-edi-test-eastus2` | Test deployment target |
| `PROD_RESOURCE_GROUP` | `rg-edi-prod-eastus2` | Prod deployment target |
| `AZURE_LOCATION` | `eastus2` | Default Azure region |
| `PROJECT_NAME` | `edi-platform` | Project identifier |

---

## GitHub Environments Configuration

### Create Environments

For repositories that deploy applications (not just infrastructure), create three environments:

**Applicable Repositories:**
- `edi-platform-core`
- `edi-sftp-connector`
- `edi-mappers`

**Navigate to:** Repository → Settings → Environments

### Environment: dev

**Protection Rules:** None (auto-deploy)

**Environment Variables:**

| Variable | Value |
|----------|-------|
| `ENVIRONMENT` | `dev` |
| `RESOURCE_GROUP` | `rg-edi-dev-eastus2` |

```powershell
$repo = "PointCHealth/edi-platform-core"
gh api --method PUT "/repos/$repo/environments/dev/variables/ENVIRONMENT" \
  --field name="ENVIRONMENT" --field value="dev"
gh api --method PUT "/repos/$repo/environments/dev/variables/RESOURCE_GROUP" \
  --field name="RESOURCE_GROUP" --field value="rg-edi-dev-eastus2"
```

### Environment: test

**Protection Rules:**
- ✅ Required reviewers: 1 reviewer from platform team
- ✅ Wait timer: 0 minutes

**Deployment branches:** `main` only

**Environment Variables:**

| Variable | Value |
|----------|-------|
| `ENVIRONMENT` | `test` |
| `RESOURCE_GROUP` | `rg-edi-test-eastus2` |

```powershell
gh api --method PUT "/repos/$repo/environments/test/variables/ENVIRONMENT" \
  --field name="ENVIRONMENT" --field value="test"
gh api --method PUT "/repos/$repo/environments/test/variables/RESOURCE_GROUP" \
  --field name="RESOURCE_GROUP" --field value="rg-edi-test-eastus2"
```

### Environment: prod

**Protection Rules:**
- ✅ Required reviewers: 2 reviewers (platform lead + security team member)
- ✅ Wait timer: 5 minutes (cooling-off period)
- ✅ Prevent administrators from bypassing

**Deployment branches:** `main` and `hotfix/*`

**Environment Variables:**

| Variable | Value |
|----------|-------|
| `ENVIRONMENT` | `prod` |
| `RESOURCE_GROUP` | `rg-edi-prod-eastus2` |
| `ENABLE_CHANGE_VALIDATION` | `true` |

```powershell
gh api --method PUT "/repos/$repo/environments/prod/variables/ENVIRONMENT" \
  --field name="ENVIRONMENT" --field value="prod"
gh api --method PUT "/repos/$repo/environments/prod/variables/RESOURCE_GROUP" \
  --field name="RESOURCE_GROUP" --field value="rg-edi-prod-eastus2"
gh api --method PUT "/repos/$repo/environments/prod/variables/ENABLE_CHANGE_VALIDATION" \
  --field name="ENABLE_CHANGE_VALIDATION" --field value="true"
```

---

## Verification

### Test Azure Authentication

Create and run a test workflow to verify OIDC authentication works:

```yaml
# .github/workflows/test-azure-auth.yml
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
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID_DEV }}
          client-secret: ${{ secrets.AZURE_CLIENT_SECRET }}

      - name: Verify Authentication
        run: |
          echo "Testing Azure CLI authentication..."
          az account show
          az group list --query "[?name=='rg-edi-dev-eastus2']" --output table

      - name: Test Summary
        run: |
          echo "✅ Azure authentication successful!" >> $GITHUB_STEP_SUMMARY
          echo "✅ Can access DEV resource group" >> $GITHUB_STEP_SUMMARY
```

**Run the test:**

```powershell
gh workflow run test-azure-auth.yml --repo PointCHealth/edi-platform-core
```

### Verification Checklist

- [ ] All 7 repositories have secrets configured
- [ ] All 7 repositories have variables configured
- [ ] GitHub environments created (dev, test, prod) for application repositories
- [ ] Environment protection rules configured
- [ ] Test authentication workflow runs successfully
- [ ] Can list resource groups in DEV subscription
- [ ] Can list resource groups in PROD subscription

---

## Troubleshooting

### Issue: Secret Not Found

**Error in workflow:**
```
Error: Secret AZURE_CLIENT_ID not found
```

**Solution:** Verify secret is added to the correct repository (not organization-level)

```powershell
# List secrets for a repository
gh secret list --repo PointCHealth/edi-platform-core
```

### Issue: Authentication Failure

**Error:**
```
Error: AADSTS7000215: Invalid client secret provided
```

**Solution:** 
1. Verify the client secret value is correct (no extra spaces)
2. Check if the secret has expired
3. Request new secret from operations team if needed

### Issue: Insufficient Permissions

**Error:**
```
Error: The client does not have authorization to perform action
```

**Solution:** Contact operations team to verify:
1. Service Principal has Contributor role on resource groups
2. Role assignments have propagated (can take 5-10 minutes)

---

## Alternative: OIDC Configuration (Future Enhancement)

For enhanced security without secrets, consider migrating to OpenID Connect (OIDC):

**Benefits:**
- ✅ No secrets to manage or rotate
- ✅ Short-lived tokens (more secure)
- ✅ GitHub recommends this approach

**Setup Steps:**

1. Operations creates federated credentials for the Service Principal
2. GitHub secrets only need:
   - `AZURE_TENANT_ID`
   - `AZURE_CLIENT_ID`
   - `AZURE_SUBSCRIPTION_ID_DEV`
   - `AZURE_SUBSCRIPTION_ID_PROD`
3. No `AZURE_CLIENT_SECRET` needed!

See [20-github-actions-setup.md](./20-github-actions-setup.md) for detailed OIDC configuration instructions.

---

## Security Best Practices

### Secret Management

- ✅ **Never commit secrets to Git**
- ✅ **Never log secret values** in workflow outputs
- ✅ **Use environment-scoped secrets** for sensitive production values
- ✅ **Rotate secrets quarterly** (coordinate with operations team)
- ✅ **Enable secret scanning** in repository settings

### Access Control

- ✅ Limit repository admin access to essential team members
- ✅ Use branch protection rules on `main` branch
- ✅ Require pull request reviews before merging
- ✅ Enable code scanning and dependency scanning

### Audit

- ✅ Review GitHub Actions logs regularly
- ✅ Monitor Azure Activity Log for deployment actions
- ✅ Set up alerts for failed deployments
- ✅ Document any manual interventions

---

## Related Documentation

- **[Operations Azure Access Request](./OPERATIONS-AZURE-ACCESS-REQUEST.md)** - What operations needs to provide
- **[GitHub Actions Setup](./20-github-actions-setup.md)** - Complete GitHub Actions configuration
- **[CI/CD Workflows](./21-cicd-workflows.md)** - Workflow implementations
- **[Deployment Procedures](./22-deployment-procedures.md)** - Deployment guides

---

## Support

**For GitHub Configuration Issues:**
- Check GitHub Actions logs: Repository → Actions → [Workflow Run]
- Verify secrets exist: Repository → Settings → Secrets and variables → Actions

**For Azure Access Issues:**
- Contact operations team
- Reference ticket from `OPERATIONS-AZURE-ACCESS-REQUEST.md`

**For Workflow Issues:**
- Review workflow logs for specific error messages
- Check Azure Activity Log for deployment errors
- Consult [22-deployment-procedures.md](./22-deployment-procedures.md)

---

**Document Version:** 1.0  
**Last Updated:** October 7, 2025  
**Next Review:** After successful initial deployment or when credentials change

