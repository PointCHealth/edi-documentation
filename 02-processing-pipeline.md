# Healthcare EDI Platform - Processing Pipeline

**Document Version:** 1.0  
**Date:** October 6, 2025  
**Status:** Production  
**Owner:** EDI Platform Team

---

## Overview

### Purpose

The Processing Pipeline subsystem orchestrates validation, transformation, and metadata extraction for EDI files after landing. Built on Azure Data Factory (ADF), it bridges file ingestion and domain routing by enforcing data quality, maintaining audit trails, and preparing files for downstream processing.

**Key Responsibilities:**

- Event-driven pipeline orchestration via Event Grid
- Multi-stage validation (naming, authorization, integrity, structural)
- Metadata extraction and persistence (ingestion records)
- Raw zone persistence with hierarchical partitioning
- Quarantine management for failed files
- Routing trigger preparation

### Key Principles

1. **Event-Driven Architecture**: Storage events trigger pipelines; no scheduled polling
2. **Fail-Fast Validation**: Early rejection minimizes resource consumption
3. **Idempotency**: Duplicate files detected via SHA256 hash comparison
4. **Immutable Raw Storage**: Files never modified after persistence
5. **Observability First**: All pipeline activities logged to Log Analytics
6. **Configuration-Driven**: Partner policies and validation rules externalized

---

## Architecture

### Component Diagram

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                     PROCESSING PIPELINE ARCHITECTURE                     │
└─────────────────────────────────────────────────────────────────────────┘

  ┌──────────────┐
  │ Event Grid   │ Blob Created Event
  │   Topic      │ (sftp-root/inbound/<partner>/<file>)
  └──────┬───────┘
         │
         ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │  ADF Pipeline: pl_ingest_dispatch                                     │
  │  ─────────────────────────────────                                    │
  │  1. Get Metadata Activity (blob properties)                          │
  │  2. Parse filename (partner code, transaction set)                   │
  │  3. Validate naming convention (regex)                               │
  │  4. Check partner authorization                                       │
  │  5. Route to specialized pipeline based on file type                 │
  └──────────────┬────────────┬──────────────────────┬────────────────────┘
                 │            │                      │
        .edi/.txt│      .zip │              Invalid │
                 │            │                      │
                 ▼            ▼                      ▼
  ┌──────────────────┐ ┌──────────────────┐  ┌──────────────────┐
  │ pl_ingest_       │ │ pl_ingest_       │  │ pl_ingest_       │
  │ validate_copy    │ │ zip_expand       │  │ error_handler    │
  └──────────────────┘ └──────────────────┘  └──────────────────┘
         │                     │                      │
         ▼                     ▼                      ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Validation Activities                                                │
  │  ─────────────────────                                                │
  │  • Checksum computation (SHA256)                                     │
  │  • Optional virus scan (Azure Function hook)                         │
  │  • Size validation                                                    │
  │  • Duplicate detection (hash lookup)                                 │
  │  • Structural peek (ISA/IEA envelope check)                          │
  └──────────────────────┬──────────────────────────────────────────────┘
                         │
                    ┌────┴────┐
                    │         │
            Success │         │ Failure
                    ▼         ▼
         ┌──────────────┐  ┌──────────────┐
         │ Copy to Raw  │  │ Move to      │
         │ Zone         │  │ Quarantine   │
         │              │  │              │
         │ Path Pattern:│  │ Path Pattern:│
         │ raw/partner= │  │ quarantine/  │
         │ <code>/      │  │ partner=     │
         │ transaction= │  │ <code>/      │
         │ <set>/       │  │ date=YYYY-   │
         │ ingest_date= │  │ MM-DD/       │
         │ YYYY-MM-DD/  │  │ <file>       │
         └──────┬───────┘  └──────┬───────┘
                │                  │
                ▼                  ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │  pl_ingest_metadata_publish                                           │
  │  ────────────────────────────                                         │
  │  • Write metadata JSON to metadata/ingestion/date=YYYY-MM-DD/        │
  │  • Publish to Log Analytics (EDIIngestion_CL custom table)           │
  │  • Tag blob with metadata (partner, transaction, sensitivity)        │
  └──────────────────────┬──────────────────────────────────────────────┘
                         │
                    Success│
                         ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Routing Trigger (Next Stage)                                         │
  │  ────────────────────────────                                         │
  │  • Invoke func_router_dispatch (Azure Function)                      │
  │  • Pass: ingestionId, rawBlobPath, partnerCode, transactionSet       │
  │  • Function publishes routing messages to Service Bus                │
  └──────────────────────────────────────────────────────────────────────┘
```

### Data Flow (Happy Path)

```text
1. Partner uploads file → SFTP landing zone
2. Event Grid publishes Blob Created event
3. pl_ingest_dispatch triggered with blob URL
4. Get Metadata: Extract blob properties (size, lastModified, name)
5. Parse filename: Extract partnerCode, transactionSet, timestamp
6. Validate naming: Regex match against pattern
7. Authorize partner: Lookup partnerCode in configuration
8. Check file type: Branch on extension (.edi, .zip, other)
9. Compute checksum: SHA256 hash of blob content
10. Duplicate detection: Query metadata store for existing hash
11. Optional virus scan: Call Azure Function with file stream
12. Structural peek: Validate ISA/IEA envelope consistency
13. Copy to raw zone: Persist with partitioned path
14. Tag blob: Apply metadata tags (partner, transaction, PHI, ingestionId)
15. Write metadata: JSON record to metadata/ingestion/date=YYYY-MM-DD/
16. Publish to Log Analytics: Custom table EDIIngestion_CL
17. Trigger routing: Invoke func_router_dispatch with ingestionId
```

### Error Path

```text
1. Validation failure detected (any stage)
2. Set pipeline variable: validationStatus = QUARANTINED
3. Set pipeline variable: quarantineReason = <failure detail>
4. Copy file to quarantine path
5. Write metadata record with failure details
6. Emit alert via Action Group (email/Teams/ServiceNow)
7. Log diagnostic event to Log Analytics (severity=Medium/High)
8. Pipeline completes (does NOT block other files)
```

---

## Configuration

### Pipeline Definitions

**Primary Pipelines:**

| Pipeline Name | Trigger | Purpose | Avg Duration |
|---------------|---------|---------|--------------|
| `pl_ingest_dispatch` | Event Grid | Entry point; routes by file type | 5-15 sec |
| `pl_ingest_validate_copy` | Called by dispatch | Validates and copies single file | 10-45 sec |
| `pl_ingest_zip_expand` | Called by dispatch | Decompresses archives, invokes validate foreach | 30-120 sec |
| `pl_ingest_error_handler` | Called on failure | Centralized quarantine + alerting | 3-8 sec |
| `pl_ingest_metadata_publish` | Called by validate | Writes metadata records | 5-10 sec |
| `pl_reprocess` | Manual/automation | Reprocesses quarantined files | 15-60 sec |

### ADF Configuration

**Resource Properties:**

```bicep
resource dataFactory 'Microsoft.DataFactory/factories@2018-06-01' = {
  name: 'adf-${namingPrefix}'
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    publicNetworkAccess: 'Enabled' // Restricted to VNet via Managed VNet
  }
}

// Managed Virtual Network (private network isolation)
resource managedVirtualNetwork 'Microsoft.DataFactory/factories/managedVirtualNetworks@2018-06-01' = {
  parent: dataFactory
  name: 'default'
  properties: {}
}

// Integration Runtime (compute for activities)
resource integrationRuntime 'Microsoft.DataFactory/factories/integrationRuntimes@2018-06-01' = {
  parent: dataFactory
  name: 'AutoResolveIntegrationRuntime'
  properties: {
    type: 'Managed'
    managedVirtualNetwork: {
      referenceName: 'default'
      type: 'ManagedVirtualNetworkReference'
    }
    typeProperties: {
      computeProperties: {
        location: 'AutoResolve'
        dataFlowProperties: {
          computeType: 'General'
          coreCount: 8
          timeToLive: 10 // minutes
        }
      }
    }
  }
}
```

**Linked Services:**

- **AzureDataLakeStorage**: Connection to ADLS Gen2 raw/staging zones
  - Authentication: System-assigned managed identity
  - Permissions: Storage Blob Data Contributor (RBAC)
- **AzureKeyVault**: Secret retrieval for partner credentials
  - Authentication: System-assigned managed identity
  - Permissions: Key Vault Secrets User
- **AzureSqlDatabase**: Control number store (for outbound)
  - Authentication: System-assigned managed identity
  - Permissions: db_datareader, db_datawriter
- **AzureFunction**: Custom validation activities
  - Authentication: Function key from Key Vault

### File Naming Convention

**Required Pattern:**

```regex
^[A-Za-z0-9]{2,10}_(\d{3}[A-Za-z]?)_\d{14}_[0-9]{3,6}\.[A-Za-z0-9]+$
```

**Example Valid Names:**

- `PARTNERA_270_20251006120000_001.edi`
- `HEALTHCO_837P_20251006143022_000125.txt`
- `PAYER01_834_20251006090000_999999.dat`

**Components:**

| Component | Description | Pattern | Example |
|-----------|-------------|---------|---------|
| **PartnerCode** | Unique partner identifier | `[A-Za-z0-9]{2,10}` | `PARTNERA` |
| **TransactionSet** | X12 transaction type | `\d{3}[A-Za-z]?` | `270`, `837P` |
| **Timestamp** | UTC generation time | `YYYYMMDDHHMMSS` | `20251006120000` |
| **Sequence** | Incremental counter | `\d{3,6}` | `001`, `000125` |
| **Extension** | File type | `.edi`, `.txt`, `.dat`, `.zip` | `.edi` |

### Storage Path Patterns

**Landing Zone (SFTP):**

```text
sftp-root/inbound/<partnerCode>/<originalFileName>

Example:
sftp-root/inbound/PARTNERA/PARTNERA_270_20251006120000_001.edi
```

**Raw Zone (Validated):**

```text
raw/partner=<partnerCode>/transaction=<transactionSet>/ingest_date=YYYY-MM-DD/<originalFileName>

Example:
raw/partner=PARTNERA/transaction=270/ingest_date=2025-10-06/PARTNERA_270_20251006120000_001.edi
```

**Quarantine Zone:**

```text
quarantine/partner=<partnerCode>/ingest_date=YYYY-MM-DD/<originalFileName>

Example:
quarantine/partner=PARTNERA/ingest_date=2025-10-06/PARTNERA_270_20251006120000_001.edi
```

**Metadata Zone:**

```text
metadata/ingestion/date=YYYY-MM-DD/part-<guid>.json

Example:
metadata/ingestion/date=2025-10-06/part-9f1c6d2e-3f3d-4b6c-9d4b-0e9e2a6c1a11.json
```

---

## Operations

### Monitoring Metrics

**Pipeline Metrics:**

| Metric | Description | Target | KQL Query |
|--------|-------------|--------|-----------|
| **Ingestion Latency** | processedUtc - receivedUtc | < 300 sec (p95) | See below |
| **Validation Failure Rate** | Failures / total per day | < 2% | See below |
| **Pipeline Success Rate** | Successful runs / total | > 99% | See below |
| **Throughput** | Files processed per hour | 5,000+ | See below |
| **Quarantine Rate** | Quarantined files / total | < 2% | See below |

**KQL Query Examples:**

```kql
// Ingestion Latency (p50, p95, p99)
EDIIngestion_CL
| where TimeGenerated > ago(24h)
| extend LatencySec = datetime_diff('second', processedUtc_t, receivedUtc_t)
| summarize 
    p50 = percentile(LatencySec, 50),
    p95 = percentile(LatencySec, 95),
    p99 = percentile(LatencySec, 99),
    count()
| render timechart

// Validation Failure Rate by Partner
EDIIngestion_CL
| where TimeGenerated > ago(7d)
| summarize 
    Total = count(),
    Failures = countif(validationStatus_s != "SUCCESS")
    by partnerCode_s
| extend FailureRate = round(100.0 * Failures / Total, 2)
| where FailureRate > 2.0
| order by FailureRate desc

// Pipeline Duration by Activity
ADFPipelineRun
| where PipelineName == "pl_ingest_validate_copy"
| where TimeGenerated > ago(24h)
| extend DurationSec = datetime_diff('second', End, Start)
| summarize 
    avg(DurationSec), 
    percentile(DurationSec, 95)
    by ActivityName
| order by avg_DurationSec desc
| render barchart

// Quarantine Trend by Reason
EDIIngestion_CL
| where TimeGenerated > ago(30d)
| where validationStatus_s == "QUARANTINED"
| summarize count() by quarantineReason_s, bin(TimeGenerated, 1d)
| render timechart

// Find failed pipeline runs
ADFPipelineRun
| where Status == "Failed"
| where TimeGenerated > ago(1h)
| project 
    TimeGenerated,
    PipelineName,
    RunId,
    ErrorCode,
    ErrorMessage,
    FailureType
| order by TimeGenerated desc

// Identify slow activities
ADFActivityRun
| where TimeGenerated > ago(1h)
| extend DurationSec = datetime_diff('second', End, Start)
| where DurationSec > 30
| summarize 
    avg(DurationSec),
    max(DurationSec),
    count()
    by ActivityName, ActivityType
| order by avg_DurationSec desc

// Quarantine reasons breakdown
EDIIngestion_CL
| where TimeGenerated > ago(24h)
| where validationStatus_s == "QUARANTINED"
| summarize count() by quarantineReason_s
| order by count_ desc
| render piechart

// Find duplicate ingestion IDs
EDIIngestion_CL
| where TimeGenerated > ago(24h)
| summarize count() by checksumSha256_s
| where count_ > 1
| order by count_ desc
```

### Troubleshooting

#### 1. Pipeline Execution Failures

**Symptom:** Pipeline status = Failed in ADF monitoring

**Common Causes:**

- **Timeout**: Integration Runtime busy (check TTL settings)
- **Permission denied**: Managed identity missing RBAC role
- **Network error**: Managed VNet not configured or firewall blocking
- **Invalid reference**: Linked service or dataset misconfigured

**Resolution:**

```powershell
# Check managed identity permissions
$MI_ObjectId = (Get-AzDataFactoryV2 -ResourceGroupName "rg-edi-prod" -Name "adf-edi-prod").Identity.PrincipalId
Get-AzRoleAssignment -ObjectId $MI_ObjectId

# Verify linked service connection
Test-AzDataFactoryV2LinkedService -ResourceGroupName "rg-edi-prod" -DataFactoryName "adf-edi-prod" -Name "AzureDataLakeStorage"

# Check Integration Runtime status
Get-AzDataFactoryV2IntegrationRuntime -ResourceGroupName "rg-edi-prod" -DataFactoryName "adf-edi-prod" -Name "AutoResolveIntegrationRuntime"
```

#### 2. Slow Pipeline Performance

**Symptom:** Ingestion latency > 300 seconds

**Common Causes:**

- **Large files**: > 50 MB files causing Copy Activity timeout
- **Cold start**: Integration Runtime warming up (increase TTL)
- **Azure Function latency**: Custom validation functions slow
- **Metadata lookup**: Inefficient partner config queries

**Resolution:**

- Enable parallel processing for large files
- Increase Integration Runtime TTL to 30 minutes
- Optimize Azure Function code (caching, connection pooling)
- Use indexed lookup datasets for partner config

#### 3. Quarantine Rate Spike

**Symptom:** > 5% of files quarantined

**Common Causes:**

- **Naming violations**: Partner changed naming convention
- **Unauthorized partner**: Partner config not updated
- **Virus scan failures**: False positives from AV engine
- **Structural errors**: Malformed X12 envelopes

**Resolution:**

- Review partner config for recent changes
- Validate naming convention with partner
- Tune AV engine sensitivity (if false positives)
- Contact partner for file format corrections

#### 4. Duplicate Detection Failures

**Symptom:** Same file processed multiple times

**Common Causes:**

- **Hash collision**: Extremely rare (SHA256 collision)
- **Metadata lookup failure**: Metadata store unavailable
- **Race condition**: Two pipelines processing same file

**Resolution:**

- Verify metadata store availability
- Check for concurrent pipeline runs (should not happen)
- Review Event Grid subscription filter to prevent duplicates

### Reprocessing

**Manual Reprocessing:**

1. Locate file in quarantine zone
2. Verify issue resolved
3. Trigger `pl_reprocess` pipeline with parameters:
   - `originalBlobPath`: Full path to quarantined file
   - `force`: true/false (skip re-validation if true)
4. Monitor pipeline execution
5. Verify success in `EDIIngestion_CL` log

**Automated Reprocessing:**

- Scheduled retry for transient errors (network, service unavailable)
- Max 3 retry attempts with exponential backoff (5min, 15min, 30min)
- After max retries: Dead-letter queue for manual intervention

---

## Security

### Access Control

**Azure Data Factory Managed Identity:**

| Resource | Role | Justification |
|----------|------|---------------|
| ADLS Gen2 Storage Account | Storage Blob Data Contributor | Read/write to landing, raw, quarantine zones |
| Azure Key Vault | Key Vault Secrets User | Retrieve partner credentials, connection strings |
| Azure SQL Database | db_datareader, db_datawriter | Control number queries (outbound) |
| Azure Functions | Function Key retrieval | Invoke custom validation functions |
| Log Analytics | Monitoring Metrics Publisher | Publish custom metrics and logs |

**Bicep Configuration:**

```bicep
// Storage RBAC assignment
resource storageBlobContributor 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: storageAccount
  name: guid(storageAccount.id, dataFactory.id, 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 
      'ba92f5b4-2d11-453d-a403-e96b0029c9fe') // Storage Blob Data Contributor
    principalId: dataFactory.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Key Vault RBAC assignment
resource keyVaultSecretsUser 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: keyVault
  name: guid(keyVault.id, dataFactory.id, '4633458b-17de-408a-b874-0445c86b69e6')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 
      '4633458b-17de-408a-b874-0445c86b69e6') // Key Vault Secrets User
    principalId: dataFactory.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

### Network Isolation

**Managed Virtual Network:**

- All Integration Runtime activities execute within ADF Managed VNet
- No public internet access for pipeline activities
- Private endpoints to ADLS Gen2, Key Vault, SQL Database
- Outbound traffic restricted to approved Azure services

**Configuration:**

```bicep
resource managedVirtualNetwork 'Microsoft.DataFactory/factories/managedVirtualNetworks@2018-06-01' = {
  parent: dataFactory
  name: 'default'
  properties: {}
}

// Private endpoint to storage
resource storagePrivateEndpoint 'Microsoft.DataFactory/factories/managedVirtualNetworks/managedPrivateEndpoints@2018-06-01' = {
  parent: managedVirtualNetwork
  name: 'storage-private-endpoint'
  properties: {
    privateLinkResourceId: storageAccount.id
    groupId: 'blob'
  }
}
```

### Data Protection

**In Transit:**

- HTTPS/TLS 1.2+ only for all data transfers
- Managed identity authentication (no credentials in pipelines)
- Private endpoints for Azure service communication

**At Rest:**

- Azure Storage Service Encryption (SSE) enabled by default
- Customer-managed keys (CMK) optional via Key Vault
- Blob versioning enabled for accidental deletion protection
- Soft delete enabled (30 days retention)

**PHI Handling:**

- Minimal data exposure in pipeline logs (no claim lines, no member IDs)
- Metadata extracts only envelope control numbers
- File content never logged to Application Insights or Log Analytics
- Blob tags mark PHI/PII sensitivity for governance

---

## Performance

### Capacity Planning

**Current Throughput:**

- **Files per hour**: 5,000+ (tested with 500 KB average file size)
- **Concurrent pipelines**: 50 (ADF default concurrency limit)
- **Integration Runtime**: 8 vCores, 10-minute TTL
- **Average latency**: 15-45 seconds (p95 < 300 sec)

**Scaling Strategies:**

1. **Horizontal Scaling**: Increase Integration Runtime core count (8 → 16 → 32 vCores)
2. **Parallel Processing**: Enable ForEach parallelism for batch files
3. **Partitioning**: Distribute files across multiple pipelines by partner
4. **Caching**: Cache partner config in memory (Azure Function static variable)

### Optimization Best Practices

1. **Minimize Data Movement:**
   - Use Get Metadata instead of full Copy for validation
   - Structural peek reads only first 10 KB of file
   - Checksum computed incrementally (streaming)

2. **Optimize Activity Chaining:**
   - Execute independent activities in parallel
   - Avoid unnecessary dependencies between activities
   - Use pipeline parameters to skip optional activities

3. **Integration Runtime Tuning:**

```bicep
resource integrationRuntime 'Microsoft.DataFactory/factories/integrationRuntimes@2018-06-01' = {
  properties: {
    typeProperties: {
      computeProperties: {
        dataFlowProperties: {
          computeType: 'General'
          coreCount: 16 // Increased from 8 for high throughput
          timeToLive: 30 // Increased from 10 to reduce cold starts
        }
      }
    }
  }
}
```

4. **Reduce External Calls:**
   - Batch Azure Function invocations where possible
   - Use ADF-native activities instead of Web Activity
   - Cache lookup results in pipeline variables

---

## Testing

### Integration Testing

**Test Scenarios:**

1. **Happy Path**: Valid file → raw zone → metadata published → routing triggered
2. **Invalid Naming**: Malformed filename → quarantine → alert
3. **Unauthorized Partner**: Unknown partner code → quarantine
4. **Duplicate File**: Same hash → skipped with SKIPPED_DUPLICATE status
5. **Large File**: 100 MB file → parallel copy → success
6. **Zip Archive**: Archive file → decompressed → each file validated
7. **Virus Detected**: Infected file → quarantine → high-severity alert
8. **Structural Error**: Malformed X12 → quarantine
9. **Network Timeout**: Storage unavailable → retry → eventual success
10. **Concurrent Uploads**: 100 files simultaneously → all processed

**Test Data Generation:**

```powershell
# Generate test file
$partnerCode = "TESTPART"
$transactionSet = "270"
$timestamp = (Get-Date).ToString("yyyyMMddHHmmss")
$sequence = "001"
$filename = "${partnerCode}_${transactionSet}_${timestamp}_${sequence}.edi"

# Upload to landing zone
az storage blob upload `
  --account-name "edistgdev01" `
  --container-name "sftp-root" `
  --name "inbound/${partnerCode}/${filename}" `
  --file "./testdata/${filename}"

# Monitor pipeline execution
az datafactory pipeline-run show `
  --factory-name "adf-edi-dev" `
  --resource-group "rg-edi-dev" `
  --run-id "<run-id>"
```

### Load Testing

**Test Configuration:**

- **Tool**: Azure Load Testing service
- **Scenario**: 5,000 files uploaded in 60 minutes (83 files/minute)
- **File sizes**: 100 KB to 5 MB (mixed distribution)
- **Partners**: 10 unique partner codes
- **Transaction types**: 270, 834, 837P (mixed)

**Success Criteria:**

- All files processed successfully
- p95 latency < 300 seconds
- No pipeline failures
- No duplicate processing
- Quarantine rate < 2%

---

## Related Documentation

- [01-data-ingestion-layer.md](./01-data-ingestion-layer.md) - SFTP landing and file reception
- [03-routing-messaging.md](./03-routing-messaging.md) - Service Bus routing (next stage)
- [06-storage-strategy.md](./06-storage-strategy.md) - Storage zones and lifecycle management

---

**Document Status:** Complete  
**Last Validation:** October 6, 2025  
**Next Review:** January 6, 2026
