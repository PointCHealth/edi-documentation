# GitHub Actions Setup Guide

**Document Version:** 1.0  
**Last Updated:** October 6, 2025  
**Status:** Production Ready  
**Owner:** EDI Platform Team (DevOps)

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Azure Authentication Setup](#2-azure-authentication-setup)
3. [Repository Configuration](#3-repository-configuration)
4. [GitHub Environments](#4-github-environments)
5. [Secrets and Variables](#5-secrets-and-variables)
6. [Branch Protection](#6-branch-protection)
7. [Verification](#7-verification)

---

## 1. Prerequisites

### 1.1 Required Access

- **Azure:** Subscription Owner or User Access Administrator role
- **GitHub:** Repository Admin access to all repositories
- **Azure CLI:** Version 2.53.0 or later installed locally
- **GitHub CLI:** `gh` CLI installed and authenticated

### 1.2 Required Information

Gather the following before starting:

| Information | Example | Where to Find |
|-------------|---------|---------------|
| Azure Tenant ID | `12345678-1234-1234-1234-123456789012` | `az account show --query tenantId` |
| Dev Subscription ID | `abcd1234-...` | Azure Portal → Subscriptions |
| Prod Subscription ID | `efgh5678-...` | Azure Portal → Subscriptions |
| Repository Full Name | `PointCHealth/edi-platform-core` | GitHub repository URL |

### 1.3 Tools Installation

```powershell
# Install Azure CLI (if not installed)
winget install Microsoft.AzureCLI

# Install GitHub CLI
winget install GitHub.cli

# Install Bicep
az bicep install

# Verify installations
az --version
gh --version
az bicep version
```

---

## 2. Azure Authentication Setup

### 2.1 Overview

We use **OpenID Connect (OIDC)** for passwordless authentication between GitHub Actions and Azure. This provides:

- ✅ No secrets stored in GitHub
- ✅ Short-lived tokens (valid only during workflow execution)
- ✅ Automatic token rotation
- ✅ Federated identity trust

### 2.2 Create Azure AD App Registrations

#### Option A: Separate Apps per Environment (Recommended)

Create one app per environment for fine-grained access control:

```powershell
# Login to Azure
az login

# Variables - UPDATE THESE VALUES
$tenantId = az account show --query tenantId -o tsv
$devSubscriptionId = "your-dev-subscription-id"
$prodSubscriptionId = "your-prod-subscription-id"
$repo = "PointCHealth/edi-platform-core"  # Update for each repository

# Create Dev Environment App
$appNameDev = "github-actions-edi-dev"
$appIdDev = az ad app create --display-name $appNameDev --query appId -o tsv
az ad sp create --id $appIdDev

Write-Host "Dev App Created: $appIdDev" -ForegroundColor Green

# Create Prod Environment App
$appNameProd = "github-actions-edi-prod"
$appIdProd = az ad app create --display-name $appNameProd --query appId -o tsv
az ad sp create --id $appIdProd

Write-Host "Prod App Created: $appIdProd" -ForegroundColor Green

# Save these values for later
Write-Host "`nSave these values:" -ForegroundColor Yellow
Write-Host "AZURE_TENANT_ID: $tenantId"
Write-Host "AZURE_CLIENT_ID_DEV: $appIdDev"
Write-Host "AZURE_CLIENT_ID_PROD: $appIdProd"
Write-Host "AZURE_SUBSCRIPTION_ID_DEV: $devSubscriptionId"
Write-Host "AZURE_SUBSCRIPTION_ID_PROD: $prodSubscriptionId"
```

#### Option B: Single App for All Environments (Simpler)

Use one app with federated credentials for each environment:

```powershell
$appName = "github-actions-edi-platform"
$appId = az ad app create --display-name $appName --query appId -o tsv
az ad sp create --id $appId
Write-Host "App Created: $appId" -ForegroundColor Green
```

### 2.3 Create Federated Credentials

#### For Dev Environment

```powershell
$appId = "your-app-id-from-above"
$repo = "PointCHealth/edi-platform-core"

# Create federated credential for dev environment
az ad app federated-credential create `
  --id $appId `
  --parameters @"
{
  \"name\": \"github-dev-environment\",
  \"issuer\": \"https://token.actions.githubusercontent.com\",
  \"subject\": \"repo:$repo:environment:dev\",
  \"description\": \"GitHub Actions OIDC for dev environment\",
  \"audiences\": [
    \"api://AzureADTokenExchange\"
  ]
}
"@

Write-Host "✅ Federated credential created for dev environment" -ForegroundColor Green
```

#### For Test Environment

```powershell
az ad app federated-credential create `
  --id $appId `
  --parameters @"
{
  \"name\": \"github-test-environment\",
  \"issuer\": \"https://token.actions.githubusercontent.com\",
  \"subject\": \"repo:$repo:environment:test\",
  \"description\": \"GitHub Actions OIDC for test environment\",
  \"audiences\": [
    \"api://AzureADTokenExchange\"
  ]
}
"@

Write-Host "✅ Federated credential created for test environment" -ForegroundColor Green
```

#### For Prod Environment

```powershell
az ad app federated-credential create `
  --id $appId `
  --parameters @"
{
  \"name\": \"github-prod-environment\",
  \"issuer\": \"https://token.actions.githubusercontent.com\",
  \"subject\": \"repo:$repo:environment:prod\",
  \"description\": \"GitHub Actions OIDC for prod environment\",
  \"audiences\": [
    \"api://AzureADTokenExchange\"
  ]
}
"@

Write-Host "✅ Federated credential created for prod environment" -ForegroundColor Green
```

#### For Pull Request Validation (No Environment)

```powershell
az ad app federated-credential create `
  --id $appId `
  --parameters @"
{
  \"name\": \"github-pull-request\",
  \"issuer\": \"https://token.actions.githubusercontent.com\",
  \"subject\": \"repo:$repo:pull_request\",
  \"description\": \"GitHub Actions OIDC for pull request validation\",
  \"audiences\": [
    \"api://AzureADTokenExchange\"
  ]
}
"@

Write-Host "✅ Federated credential created for pull requests" -ForegroundColor Green
```

#### For Main Branch Deployments

```powershell
az ad app federated-credential create `
  --id $appId `
  --parameters @"
{
  \"name\": \"github-main-branch\",
  \"issuer\": \"https://token.actions.githubusercontent.com\",
  \"subject\": \"repo:$repo:ref:refs/heads/main\",
  \"description\": \"GitHub Actions OIDC for main branch\",
  \"audiences\": [
    \"api://AzureADTokenExchange\"
  ]
}
"@

Write-Host "✅ Federated credential created for main branch" -ForegroundColor Green
```

### 2.4 Assign Azure RBAC Permissions

#### Dev Environment Permissions

```powershell
$appId = "your-app-id"
$devSubscriptionId = "your-dev-subscription-id"
$devResourceGroup = "rg-edi-dev-eastus2"

# Contributor role to dev resource group
az role assignment create `
  --assignee $appId `
  --role "Contributor" `
  --scope "/subscriptions/$devSubscriptionId/resourceGroups/$devResourceGroup"

Write-Host "✅ Granted Contributor to dev resource group" -ForegroundColor Green

# Test resource group (in same subscription)
$testResourceGroup = "rg-edi-test-eastus2"
az role assignment create `
  --assignee $appId `
  --role "Contributor" `
  --scope "/subscriptions/$devSubscriptionId/resourceGroups/$testResourceGroup"

Write-Host "✅ Granted Contributor to test resource group" -ForegroundColor Green
```

#### Prod Environment Permissions

```powershell
$appIdProd = "your-prod-app-id"
$prodSubscriptionId = "your-prod-subscription-id"
$prodResourceGroup = "rg-edi-prod-eastus2"

# Contributor role to prod resource group
az role assignment create `
  --assignee $appIdProd `
  --role "Contributor" `
  --scope "/subscriptions/$prodSubscriptionId/resourceGroups/$prodResourceGroup"

Write-Host "✅ Granted Contributor to prod resource group" -ForegroundColor Green

# Reader role at subscription level for cost queries
az role assignment create `
  --assignee $appIdProd `
  --role "Reader" `
  --scope "/subscriptions/$prodSubscriptionId"

Write-Host "✅ Granted Reader to prod subscription" -ForegroundColor Green
```

#### Key Vault Permissions (Optional)

If workflows need to manage Key Vault secrets:

```powershell
$keyVaultName = "kv-edi-dev-eastus2"
$appId = "your-app-id"

az role assignment create `
  --assignee $appId `
  --role "Key Vault Secrets Officer" `
  --scope "/subscriptions/$devSubscriptionId/resourceGroups/$devResourceGroup/providers/Microsoft.KeyVault/vaults/$keyVaultName"

Write-Host "✅ Granted Key Vault Secrets Officer" -ForegroundColor Green
```

---

## 3. Repository Configuration

### 3.1 Create GitHub Secrets

Navigate to **Repository Settings → Secrets and variables → Actions → New repository secret**

```powershell
# Use GitHub CLI to add secrets
$repo = "PointCHealth/edi-platform-core"
$tenantId = "your-tenant-id"
$clientId = "your-client-id"
$devSubId = "your-dev-subscription-id"
$prodSubId = "your-prod-subscription-id"

# Add repository secrets
gh secret set AZURE_TENANT_ID --body "$tenantId" --repo $repo
gh secret set AZURE_CLIENT_ID --body "$clientId" --repo $repo
gh secret set AZURE_SUBSCRIPTION_ID_DEV --body "$devSubId" --repo $repo
gh secret set AZURE_SUBSCRIPTION_ID_PROD --body "$prodSubId" --repo $repo

Write-Host "✅ Repository secrets configured" -ForegroundColor Green
```

**Secrets to Configure:**

| Secret Name | Value | Used By |
|-------------|-------|---------|
| `AZURE_TENANT_ID` | Your Azure AD tenant ID | All workflows |
| `AZURE_CLIENT_ID` | App registration client ID | All workflows (if single app) |
| `AZURE_SUBSCRIPTION_ID_DEV` | Dev subscription ID | Dev/Test deployments |
| `AZURE_SUBSCRIPTION_ID_PROD` | Prod subscription ID | Prod deployments |

### 3.2 Create GitHub Variables

Navigate to **Repository Settings → Secrets and variables → Actions → Variables**

```powershell
# Add repository variables
gh variable set DEV_RESOURCE_GROUP --body "rg-edi-dev-eastus2" --repo $repo
gh variable set TEST_RESOURCE_GROUP --body "rg-edi-test-eastus2" --repo $repo
gh variable set PROD_RESOURCE_GROUP --body "rg-edi-prod-eastus2" --repo $repo
gh variable set AZURE_LOCATION --body "eastus2" --repo $repo

Write-Host "✅ Repository variables configured" -ForegroundColor Green
```

**Variables to Configure:**

| Variable Name | Value | Purpose |
|---------------|-------|---------|
| `DEV_RESOURCE_GROUP` | `rg-edi-dev-eastus2` | Dev deployment target |
| `TEST_RESOURCE_GROUP` | `rg-edi-test-eastus2` | Test deployment target |
| `PROD_RESOURCE_GROUP` | `rg-edi-prod-eastus2` | Prod deployment target |
| `AZURE_LOCATION` | `eastus2` | Default Azure region |

### 3.3 Automation Script for All Repositories

Create a PowerShell script to automate configuration across all repositories:

```powershell
# configure-all-repos.ps1

$repos = @(
    "PointCHealth/edi-platform-core",
    "PointCHealth/edi-mappers",
    "PointCHealth/edi-sftp-connector",
    "PointCHealth/edi-partner-configs"
)

$tenantId = "your-tenant-id"
$clientId = "your-client-id"
$devSubId = "your-dev-subscription-id"
$prodSubId = "your-prod-subscription-id"

foreach ($repo in $repos) {
    Write-Host "`nConfiguring $repo..." -ForegroundColor Cyan

    # Secrets
    gh secret set AZURE_TENANT_ID --body "$tenantId" --repo $repo
    gh secret set AZURE_CLIENT_ID --body "$clientId" --repo $repo
    gh secret set AZURE_SUBSCRIPTION_ID_DEV --body "$devSubId" --repo $repo
    gh secret set AZURE_SUBSCRIPTION_ID_PROD --body "$prodSubId" --repo $repo

    # Variables
    gh variable set DEV_RESOURCE_GROUP --body "rg-edi-dev-eastus2" --repo $repo
    gh variable set TEST_RESOURCE_GROUP --body "rg-edi-test-eastus2" --repo $repo
    gh variable set PROD_RESOURCE_GROUP --body "rg-edi-prod-eastus2" --repo $repo
    gh variable set AZURE_LOCATION --body "eastus2" --repo $repo

    Write-Host "✅ $repo configured" -ForegroundColor Green
}

Write-Host "`n✅ All repositories configured successfully!" -ForegroundColor Green
```

---

## 4. GitHub Environments

### 4.1 Create Environments

Navigate to **Repository Settings → Environments → New environment**

Create three environments: `dev`, `test`, `prod`

### 4.2 Configure Dev Environment

**Settings → Environments → dev**

**Deployment protection rules:** None (auto-deploy)

**Environment variables:**

```powershell
gh api --method PUT /repos/$repo/environments/dev/variables/ENVIRONMENT --field name="ENVIRONMENT" --field value="dev"
gh api --method PUT /repos/$repo/environments/dev/variables/RESOURCE_GROUP --field name="RESOURCE_GROUP" --field value="rg-edi-dev-eastus2"
```

| Variable | Value |
|----------|-------|
| `ENVIRONMENT` | `dev` |
| `RESOURCE_GROUP` | `rg-edi-dev-eastus2` |

### 4.3 Configure Test Environment

**Settings → Environments → test**

**Deployment protection rules:**

- ✅ Required reviewers: Select 1 reviewer from platform team
- ✅ Wait timer: 0 minutes
- ☑️ Prevent administrators from bypassing

**Deployment branches:** `main` only

**Environment variables:**

```powershell
gh api --method PUT /repos/$repo/environments/test/variables/ENVIRONMENT --field name="ENVIRONMENT" --field value="test"
gh api --method PUT /repos/$repo/environments/test/variables/RESOURCE_GROUP --field name="RESOURCE_GROUP" --field value="rg-edi-test-eastus2"
```

| Variable | Value |
|----------|-------|
| `ENVIRONMENT` | `test` |
| `RESOURCE_GROUP` | `rg-edi-test-eastus2` |

### 4.4 Configure Prod Environment

**Settings → Environments → prod**

**Deployment protection rules:**

- ✅ Required reviewers: Select 2 reviewers (platform lead + security team member)
- ✅ Wait timer: 5 minutes (cooling-off period)
- ✅ Prevent administrators from bypassing

**Deployment branches:** `main` and `hotfix/*`

**Environment secrets:**

| Secret | Value | Purpose |
|--------|-------|---------|
| `TEAMS_WEBHOOK_URL` | Microsoft Teams webhook URL | Deployment notifications |

**Environment variables:**

```powershell
gh api --method PUT /repos/$repo/environments/prod/variables/ENVIRONMENT --field name="ENVIRONMENT" --field value="prod"
gh api --method PUT /repos/$repo/environments/prod/variables/RESOURCE_GROUP --field name="RESOURCE_GROUP" --field value="rg-edi-prod-eastus2"
gh api --method PUT /repos/$repo/environments/prod/variables/ENABLE_CHANGE_VALIDATION --field name="ENABLE_CHANGE_VALIDATION" --field value="true"
```

| Variable | Value |
|----------|-------|
| `ENVIRONMENT` | `prod` |
| `RESOURCE_GROUP` | `rg-edi-prod-eastus2` |
| `ENABLE_CHANGE_VALIDATION` | `true` |

---

## 5. Secrets and Variables

### 5.1 Secret Management Best Practices

- ✅ **Never commit secrets to Git**
- ✅ **Use environment-scoped secrets** for environment-specific values
- ✅ **Rotate secrets quarterly** (review calendar reminder)
- ✅ **Use Azure Key Vault** for application secrets (not CI/CD secrets)
- ✅ **Enable secret scanning** (Settings → Code security → Secret scanning)

### 5.2 Secrets Inventory

| Secret | Scope | Value Source | Rotation Schedule |
|--------|-------|--------------|-------------------|
| `AZURE_TENANT_ID` | Repository | Azure AD | Never (stable) |
| `AZURE_CLIENT_ID` | Repository | App registration | Never (stable) |
| `AZURE_SUBSCRIPTION_ID_DEV` | Repository | Azure subscription | Never (stable) |
| `AZURE_SUBSCRIPTION_ID_PROD` | Repository | Azure subscription | Never (stable) |
| `TEAMS_WEBHOOK_URL` | Environment (prod) | Teams channel | Annually |

**Note:** OIDC tokens are short-lived and auto-rotated, so no credential rotation needed.

### 5.3 Variables Inventory

| Variable | Scope | Value | Purpose |
|----------|-------|-------|---------|
| `DEV_RESOURCE_GROUP` | Repository | `rg-edi-dev-eastus2` | Dev deployment target |
| `TEST_RESOURCE_GROUP` | Repository | `rg-edi-test-eastus2` | Test deployment target |
| `PROD_RESOURCE_GROUP` | Repository | `rg-edi-prod-eastus2` | Prod deployment target |
| `AZURE_LOCATION` | Repository | `eastus2` | Default region |
| `ENVIRONMENT` | Environment | `dev`/`test`/`prod` | Environment name |
| `RESOURCE_GROUP` | Environment | Resource group name | Environment-specific RG |

---

## 6. Branch Protection

### 6.1 Configure Main Branch Protection

Navigate to **Settings → Branches → Add rule**

**Branch name pattern:** `main`

**Protection Rules:**

- ✅ **Require a pull request before merging**
  - Require approvals: 2
  - Dismiss stale pull request approvals when new commits are pushed
  - Require review from Code Owners

- ✅ **Require status checks to pass before merging**
  - Require branches to be up to date before merging
  - Status checks that are required:
    - `build` (CI build)
    - `test` (Unit tests)
    - `security-scan` (Security scanning)
    - `validate-bicep` (Bicep validation, if infrastructure repo)

- ✅ **Require conversation resolution before merging**

- ✅ **Require signed commits**

- ✅ **Require linear history**

- ✅ **Do not allow bypassing the above settings** (including admins)

- ✅ **Restrict pushes that create matching branches**

### 6.2 Configure Code Owners

Create a `CODEOWNERS` file in the repository root:

```
# CODEOWNERS file for edi-platform-core

# Default owners for everything in the repo
*                           @PointCHealth/platform-team

# Infrastructure
/infra/**                   @PointCHealth/platform-team @PointCHealth/devops-team

# CI/CD Workflows
/.github/workflows/**       @PointCHealth/devops-team

# Security-sensitive files
/secrets/**                 @PointCHealth/security-team
/env/prod.*                 @PointCHealth/platform-lead @PointCHealth/security-team

# Database migrations
/migrations/**              @PointCHealth/data-engineering-team
```

### 6.3 Branch Protection for Other Branches

**Hotfix Branches (`hotfix/*`):**

- ✅ Require pull request with 1 approval
- ✅ Require status checks to pass
- ✅ Allow force pushes (for hotfix rebasing)

**Feature Branches (`feature/*`):**

- No special protection (protected by main branch rules)

---

## 7. Verification

### 7.1 Test Azure OIDC Authentication

Create a test workflow to verify OIDC authentication:

```yaml
# .github/workflows/test-oidc.yml
name: Test OIDC Authentication

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  test-auth:
    name: Test Azure Authentication
    runs-on: ubuntu-latest
    environment: dev

    steps:
      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID_DEV }}

      - name: Verify Authentication
        run: |
          echo "Testing Azure CLI authentication..."
          az account show
          az group list --output table

      - name: Test Summary
        run: |
          echo "✅ Azure OIDC authentication successful!" >> $GITHUB_STEP_SUMMARY
```

**Run the workflow:**

```powershell
gh workflow run test-oidc.yml --repo $repo
```

### 7.2 Verification Checklist

- [ ] Azure AD app registrations created
- [ ] Federated credentials configured for all environments
- [ ] RBAC permissions assigned to resource groups
- [ ] GitHub repository secrets configured
- [ ] GitHub repository variables configured
- [ ] GitHub environments created (dev, test, prod)
- [ ] Environment protection rules configured
- [ ] Branch protection rules enabled on main
- [ ] CODEOWNERS file created
- [ ] Test OIDC workflow runs successfully
- [ ] All team members have appropriate repository access

### 7.3 Troubleshooting

**Issue: OIDC authentication fails**

```
Error: AADSTS70021: No matching federated identity record found
```

**Solution:** Verify federated credential subject matches exactly:

- Environment deployment: `repo:ORG/REPO:environment:ENV_NAME`
- Pull request: `repo:ORG/REPO:pull_request`
- Branch: `repo:ORG/REPO:ref:refs/heads/BRANCH_NAME`

**Issue: Insufficient permissions in Azure**

```
Error: The client does not have authorization to perform action
```

**Solution:** Verify RBAC role assignments:

```powershell
# List role assignments for service principal
az role assignment list --assignee $appId --output table
```

**Issue: GitHub environment not found**

```
Error: The deployment environment 'dev' was not found
```

**Solution:** Create environment in GitHub repository settings.

---

## 8. Next Steps

1. **Implement CI/CD Workflows** → See [21-cicd-workflows.md](./21-cicd-workflows.md)
2. **Test Deployment Procedures** → See [22-deployment-procedures.md](./22-deployment-procedures.md)
3. **Configure Monitoring** → Set up Azure Monitor dashboards and alerts
4. **Create Runbooks** → Document operational procedures

---

## 9. Related Documentation

- **[Deployment Overview](./19-deployment-automation-overview.md)** - High-level deployment architecture
- **[CI/CD Workflows](./21-cicd-workflows.md)** - Workflow implementations
- **[Deployment Procedures](./22-deployment-procedures.md)** - Step-by-step deployment guides
- **[Security & Compliance](./09-security-compliance.md)** - Security controls

---

**Document Maintenance:**
- Review quarterly or when authentication issues occur
- Update after Azure AD app registration changes
- Validate after Azure subscription changes

**Feedback:** Create GitHub issue or contact platform team

---

**Status:** ✅ Ready for implementation
