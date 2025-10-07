# Deployment Procedures

**Document Version:** 1.0  
**Last Updated:** October 6, 2025  
**Status:** Production Ready  
**Owner:** EDI Platform Team (Operations)

---

## Table of Contents

1. [Pre-Deployment Checklist](#1-pre-deployment-checklist)
2. [Infrastructure Deployment](#2-infrastructure-deployment)
3. [Function App Deployment](#3-function-app-deployment)
4. [ADF Pipeline Deployment](#4-adf-pipeline-deployment)
5. [Configuration Deployment](#5-configuration-deployment)
6. [Post-Deployment Validation](#6-post-deployment-validation)
7. [Production Deployment](#7-production-deployment)

---

## 1. Pre-Deployment Checklist

### 1.1 General Prerequisites

Before any deployment, verify:

- [ ] Pull request reviewed and approved (2 approvers)
- [ ] All CI checks passed (build, test, security scan)
- [ ] What-if analysis reviewed (for infrastructure changes)
- [ ] Change ticket created and approved (production only)
- [ ] Stakeholders notified (production only, 48 hours advance)
- [ ] Rollback plan documented
- [ ] Deployment window scheduled (production only)

### 1.2 Environment-Specific Prerequisites

#### Dev Environment

- [ ] No prerequisites (auto-deploy)

#### Test Environment

- [ ] Successful deployment to dev environment
- [ ] Smoke tests passed in dev
- [ ] 1 approver from platform team available

#### Production Environment

- [ ] Successful deployment to test environment
- [ ] Integration tests passed in test
- [ ] Performance testing completed (for major releases)
- [ ] 2 approvers available (platform lead + security team)
- [ ] On-call engineer notified and available
- [ ] Backup of current configuration saved
- [ ] Change control ticket approved
- [ ] Deployment communication sent to stakeholders

---

## 2. Infrastructure Deployment

### 2.1 Infrastructure Deployment (Bicep)

#### Step 1: Verify PR Approval

```powershell
# Check PR status
gh pr view <PR_NUMBER> --repo PointCHealth/edi-platform-core
```

Verify:

- ✅ 2 approvals received
- ✅ CI checks passed
- ✅ What-if results reviewed

#### Step 2: Merge to Main

```powershell
# Merge PR (triggers auto-deploy to dev)
gh pr merge <PR_NUMBER> --squash --repo PointCHealth/edi-platform-core
```

**Expected Result:** GitHub Actions workflow `infra-cd.yml` triggered automatically

#### Step 3: Monitor Dev Deployment

```powershell
# Watch workflow execution
gh run watch --repo PointCHealth/edi-platform-core
```

**Expected Duration:** 5-10 minutes

**Monitor for:**

- ✅ Bicep deployment successful
- ✅ Resource tagging applied
- ✅ Smoke tests passed

#### Step 4: Validate Dev Deployment

```powershell
# Check deployment status in Azure
az deployment group show \
  --resource-group rg-edi-dev-eastus2 \
  --name deploy-<RUN_NUMBER>-<SHA> \
  --query properties.provisioningState
```

**Expected Output:** `"Succeeded"`

#### Step 5: Deploy to Test (Manual)

Navigate to **Actions → Infrastructure CD → Run workflow**

**Inputs:**

- Environment: `test`
- Skip tests: `false`

Click **Run workflow**

**Approval Required:** 1 reviewer from platform team

**Expected Duration:** 10-15 minutes

#### Step 6: Validate Test Deployment

```powershell
# Run integration tests
gh workflow run integration-tests.yml \
  --repo PointCHealth/edi-platform-core \
  --ref main \
  --field environment=test
```

**Expected Result:** All integration tests pass

#### Step 7: Deploy to Production (Manual)

Navigate to **Actions → Infrastructure CD → Run workflow**

**Inputs:**

- Environment: `prod`
- Skip tests: `false`

Click **Run workflow**

**Approval Required:**

- 2 reviewers (platform lead + security team)
- 5-minute wait timer

**Expected Duration:** 20-30 minutes

#### Step 8: Post-Deployment Validation

See [Section 6: Post-Deployment Validation](#6-post-deployment-validation)

---

## 3. Function App Deployment

### 3.1 Function App Deployment (.NET)

#### Step 1: Merge PR to Main

```powershell
# Merge PR (triggers build)
gh pr merge <PR_NUMBER> --squash --repo PointCHealth/edi-platform-core
```

**Expected Result:** `function-cd.yml` workflow triggered

#### Step 2: Monitor Build

```powershell
# Watch build workflow
gh run watch --repo PointCHealth/edi-platform-core
```

**Expected Duration:** 3-5 minutes

**Artifacts Created:**

- `function-app-<RUN_NUMBER>.zip`

#### Step 3: Auto-Deploy to Dev

**Automatic:** Dev deployment triggered on successful build

**Monitor Deployment:**

```powershell
# Check function app status
az functionapp show \
  --name func-edi-router-dev-eastus2 \
  --resource-group rg-edi-dev-eastus2 \
  --query state
```

#### Step 4: Validate Dev Deployment

```powershell
# Test health endpoint
curl https://func-edi-router-dev-eastus2.azurewebsites.net/api/health

# Check Application Insights for errors
az monitor app-insights metrics show \
  --app func-edi-router-dev-eastus2 \
  --metric exceptions/count \
  --start-time $(Get-Date).AddMinutes(-10).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
```

**Expected Result:** Health endpoint returns `200 OK`, no exceptions

#### Step 5: Deploy to Test (Manual)

Navigate to **Actions → Function App CD → Run workflow**

**Inputs:**

- Environment: `test`

Click **Run workflow**

**Approval Required:** 1 reviewer

**Expected Duration:** 5-7 minutes

#### Step 6: Run Integration Tests

```powershell
# Trigger integration test workflow
gh workflow run integration-tests.yml \
  --repo PointCHealth/edi-platform-core \
  --ref main \
  --field environment=test \
  --field function-app=router
```

**Expected Result:** All tests pass

#### Step 7: Deploy to Production (Manual)

Navigate to **Actions → Function App CD → Run workflow**

**Inputs:**

- Environment: `prod`

Click **Run workflow**

**Approval Required:** 2 reviewers + 5-minute wait

**Expected Duration:** 10-15 minutes

#### Step 8: Monitor Production Deployment

```powershell
# Watch deployment logs
az functionapp log tail \
  --name func-edi-router-prod-eastus2 \
  --resource-group rg-edi-prod-eastus2

# Monitor error rate
az monitor metrics list \
  --resource func-edi-router-prod-eastus2 \
  --resource-group rg-edi-prod-eastus2 \
  --resource-type Microsoft.Web/sites \
  --metric Http5xx \
  --interval PT1M
```

**Watch for 10 minutes:** Error rate should remain below baseline

---

## 4. ADF Pipeline Deployment

### 4.1 ADF Development Workflow

#### Step 1: Develop in ADF UI

1. Open Azure Data Factory Studio
2. Connect to **collaboration branch** (`main`)
3. Create/modify pipelines, datasets, linked services
4. Click **Save All** (saves to Git branch)

#### Step 2: Create Pull Request

1. In ADF UI, click **Create Pull Request**
2. Or manually create PR in GitHub from `adf_publish` branch

#### Step 3: Validate PR

**Automatic:** `adf-ci.yml` workflow validates ADF resources

```powershell
# Check validation status
gh pr checks <PR_NUMBER> --repo PointCHealth/edi-platform-core
```

#### Step 4: Merge PR

```powershell
# Merge PR (triggers ARM export)
gh pr merge <PR_NUMBER> --squash --repo PointCHealth/edi-platform-core
```

**Expected Result:** `adf-export.yml` workflow generates ARM templates

#### Step 5: Deploy to Dev (Manual)

Navigate to **Actions → ADF Deployment → Run workflow**

**Inputs:**

- Environment: `dev`

Click **Run workflow**

**Expected Duration:** 3-5 minutes

#### Step 6: Test Pipeline in Dev

```powershell
# Manually trigger pipeline with test file
az datafactory pipeline create-run \
  --factory-name adf-edi-dev-eastus2 \
  --resource-group rg-edi-dev-eastus2 \
  --name PL_Inbound_SFTP_to_Storage \
  --parameters '@test-parameters.json'

# Monitor pipeline run
az datafactory pipeline-run show \
  --factory-name adf-edi-dev-eastus2 \
  --resource-group rg-edi-dev-eastus2 \
  --run-id <RUN_ID>
```

**Expected Result:** Pipeline completes successfully

#### Step 7: Deploy to Test and Production

Follow same manual workflow trigger for test and prod environments

**Important:** Disable production triggers before deployment, re-enable after validation

```powershell
# Disable triggers before deployment
az datafactory trigger stop \
  --factory-name adf-edi-prod-eastus2 \
  --resource-group rg-edi-prod-eastus2 \
  --name TR_Inbound_Schedule

# Re-enable after validation
az datafactory trigger start \
  --factory-name adf-edi-prod-eastus2 \
  --resource-group rg-edi-prod-eastus2 \
  --name TR_Inbound_Schedule
```

---

## 5. Configuration Deployment

### 5.1 Partner Configuration Deployment

#### Step 1: Validate Configuration Files

```powershell
# Run JSON schema validation
pwsh scripts/validate-partner-configs.ps1
```

**Expected Result:** All configurations pass validation

#### Step 2: Merge PR

```powershell
# Merge PR (triggers config deployment)
gh pr merge <PR_NUMBER> --squash --repo PointCHealth/edi-partner-configs
```

#### Step 3: Upload to Blob Storage

**Automatic:** `config-cd.yml` workflow uploads configs to blob storage

**Versioning Strategy:** Configs uploaded with timestamp suffix

```text
config-v2024-10-06-123456/
  ├── partner-001.json
  ├── partner-002.json
  └── routing-rules.json
```

#### Step 4: Update Function App Settings

```powershell
# Update app setting to point to new config version
az functionapp config appsettings set \
  --name func-edi-router-dev-eastus2 \
  --resource-group rg-edi-dev-eastus2 \
  --settings "ConfigVersion=v2024-10-06-123456"
```

#### Step 5: Restart Function Apps (if needed)

```powershell
# Restart to load new configuration
az functionapp restart \
  --name func-edi-router-dev-eastus2 \
  --resource-group rg-edi-dev-eastus2
```

#### Step 6: Validate Configuration

```powershell
# Test with sample transaction
curl -X POST https://func-edi-router-dev-eastus2.azurewebsites.net/api/route \
  -H "Content-Type: application/json" \
  -d @test-message.json
```

---

## 6. Post-Deployment Validation

### 6.1 Infrastructure Validation

```powershell
# Verify all resources deployed successfully
az deployment group show \
  --resource-group rg-edi-<ENV>-eastus2 \
  --name <DEPLOYMENT_NAME> \
  --query properties.outputResources[].id

# Check resource health
az monitor metrics list \
  --resource-group rg-edi-<ENV>-eastus2 \
  --resource <RESOURCE_NAME> \
  --metric Availability \
  --interval PT5M
```

### 6.2 Function App Validation

```powershell
# Test all health endpoints
$functionApps = @(
    "func-edi-router-<ENV>-eastus2",
    "func-edi-validator-<ENV>-eastus2",
    "func-edi-orchestrator-<ENV>-eastus2"
)

foreach ($app in $functionApps) {
    Write-Host "Testing $app..." -ForegroundColor Cyan
    $response = Invoke-WebRequest -Uri "https://$app.azurewebsites.net/api/health" -Method GET
    if ($response.StatusCode -eq 200) {
        Write-Host "✅ $app is healthy" -ForegroundColor Green
    } else {
        Write-Host "❌ $app health check failed" -ForegroundColor Red
    }
}
```

### 6.3 Integration Testing

```powershell
# Run full integration test suite
gh workflow run integration-tests.yml \
  --repo PointCHealth/edi-platform-core \
  --ref main \
  --field environment=<ENV>

# Monitor test results
gh run watch
```

### 6.4 Monitoring Dashboard Check

1. Open **Azure Portal**
2. Navigate to **Application Insights** for the environment
3. Check **Live Metrics**
4. Verify:
   - ✅ Request rate within normal range
   - ✅ Response time within acceptable limits
   - ✅ Failure rate < 1%
   - ✅ No exceptions in last 10 minutes

---

## 7. Production Deployment

### 7.1 Pre-Production Checklist

**48 Hours Before Deployment:**

- [ ] Change ticket created and approved
- [ ] Stakeholder notification sent
- [ ] Deployment runbook reviewed
- [ ] Rollback plan documented
- [ ] Team availability confirmed

**24 Hours Before Deployment:**

- [ ] Final testing completed in test environment
- [ ] Performance validation completed
- [ ] Security scan results reviewed
- [ ] Backup of current state created

**Day of Deployment:**

- [ ] Deployment window confirmed
- [ ] On-call engineer available
- [ ] Communication channel open (Teams/Slack)
- [ ] Monitoring dashboards ready

### 7.2 Production Deployment Procedure

#### Phase 1: Pre-Deployment (15 minutes before)

```powershell
# Send deployment start notification
# Post to Teams channel

# Take snapshot of current metrics
az monitor metrics list \
  --resource-group rg-edi-prod-eastus2 \
  --resource func-edi-router-prod-eastus2 \
  --metric Http5xx \
  --start-time $(Get-Date).AddHours(-1).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
```

#### Phase 2: Deployment Execution

Follow procedures from Sections 2-5 based on component being deployed

**Critical:** Monitor deployment progress continuously

#### Phase 3: Post-Deployment Validation (30 minutes)

```powershell
# Run production smoke tests
pwsh scripts/smoke-tests.ps1 -Environment prod

# Monitor for 30 minutes
# Watch for:
# - Error rate spikes
# - Performance degradation
# - Exception increases
```

#### Phase 4: Completion

```powershell
# Send deployment completion notification
# Post to Teams channel with:
# - Deployment success/failure status
# - Components deployed
# - Any issues encountered
# - Next steps (if issues)
```

### 7.3 Production Deployment Rollback

If any issues detected during validation, immediately execute rollback procedure.

See [23-rollback-procedures.md](./23-rollback-procedures.md) for detailed rollback steps.

---

## 8. Troubleshooting Common Issues

### 8.1 Deployment Fails Due to Bicep Errors

**Symptom:** Bicep deployment returns validation errors

**Solution:**

```powershell
# Review what-if analysis
az deployment group what-if \
  --resource-group rg-edi-<ENV>-eastus2 \
  --template-file infra/bicep/main.bicep \
  --parameters env/<ENV>.parameters.json

# Fix errors in Bicep template
# Re-run CI/CD pipeline
```

### 8.2 Function App Deployment Timeout

**Symptom:** Function app deployment takes >10 minutes

**Solution:**

```powershell
# Check function app status
az functionapp show \
  --name func-edi-router-<ENV>-eastus2 \
  --resource-group rg-edi-<ENV>-eastus2

# Restart function app
az functionapp restart \
  --name func-edi-router-<ENV>-eastus2 \
  --resource-group rg-edi-<ENV>-eastus2

# Retry deployment
```

### 8.3 Configuration Not Loading

**Symptom:** Function app not loading new configuration

**Solution:**

```powershell
# Verify app setting
az functionapp config appsettings list \
  --name func-edi-router-<ENV>-eastus2 \
  --resource-group rg-edi-<ENV>-eastus2 \
  --query "[?name=='ConfigVersion'].value"

# Verify blob exists
az storage blob exists \
  --account-name stedi<ENV>eastus2 \
  --container-name configs \
  --name config-v2024-10-06-123456/partner-001.json

# Restart function app to reload
az functionapp restart \
  --name func-edi-router-<ENV>-eastus2 \
  --resource-group rg-edi-<ENV>-eastus2
```

---

## 9. Related Documentation

- **[Deployment Overview](./19-deployment-automation-overview.md)** - High-level deployment architecture
- **[GitHub Actions Setup](./20-github-actions-setup.md)** - Initial setup
- **[CI/CD Workflows](./21-cicd-workflows.md)** - Workflow implementations
- **[Rollback Procedures](./23-rollback-procedures.md)** - Emergency rollback procedures

---

**Document Maintenance:**

- Review after each production deployment
- Update after process improvements
- Validate procedures quarterly

**Feedback:** Create GitHub issue or contact platform team

---

**Status:** ✅ Ready for use
