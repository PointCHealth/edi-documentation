# Storage Strategy & Data Lake Architecture

## Table of Contents
- [Overview](#overview)
- [ADLS Gen2 Multi-Zone Architecture](#adls-gen2-multi-zone-architecture)
- [Container Specifications](#container-specifications)
- [Lifecycle Management](#lifecycle-management)
- [Hybrid Storage Model](#hybrid-storage-model)
- [Operations](#operations)
- [Monitoring](#monitoring)
- [Security](#security)
- [Disaster Recovery](#disaster-recovery)
- [Cost Optimization](#cost-optimization)

---

## Overview

### Purpose

The Storage Strategy defines how EDI files, metadata, and processing artifacts are organized, retained, and optimized across the Healthcare EDI Platform. This strategy implements an Azure Data Lake Storage Gen2 (ADLS Gen2) multi-zone architecture with automated lifecycle management to balance operational access, compliance requirements, and cost optimization.

### Business Requirements

1. **Event Sourcing**: Preserve original EDI files for replay and reprocessing
2. **Idempotency**: Prevent duplicate processing via file hash comparison
3. **Audit Trail**: Maintain complete chain of custody for compliance
4. **Retention**: HIPAA 7-10 year retention for healthcare transactions
5. **Performance**: Fast access (< 1 ms) for operational files (0-90 days)
6. **Cost Optimization**: Reduce storage costs by 85-90% through automated tiering

### Key Benefits

- **Cost Savings**: $60,682 over 5 years (90% reduction) vs. database-only storage
- **Scalability**: Petabyte-scale storage with hierarchical namespace for efficient querying
- **Compliance**: HIPAA retention policies with automated lifecycle enforcement
- **Performance**: Hot tier (< 1 ms access) for active files, Archive tier for long-term retention
- **Event Replay**: Original EDI files preserved for reprocessing scenarios

### Architecture Principles

1. **Separation of Concerns**: Database stores metadata/references, blob stores file content
2. **Zone-Based Organization**: Landing → Raw → Processed → Archive progression
3. **Automated Tiering**: Lifecycle policies move files between tiers without manual intervention
4. **Immutable Storage**: Versioning enabled for point-in-time recovery
5. **Defense in Depth**: Private endpoints, encryption at rest/transit, RBAC authorization

---

## ADLS Gen2 Multi-Zone Architecture

### Storage Account Configuration

**Account Type**: StorageV2 (General Purpose v2)  
**Performance Tier**: Standard  
**Replication**: GRS (Geo-Redundant Storage) for production, LRS (Locally Redundant) for dev/test  
**Hierarchical Namespace**: Enabled (ADLS Gen2 feature)  
**Versioning**: Enabled (30-day retention)  
**Soft Delete**: Enabled (30-day retention)  
**Change Feed**: Enabled (audit trail)

**Naming Convention**: `edistg{environment}{region}01`

**Examples**:
- `edistgprodeast01` - Production (East US)
- `edistgdeveast01` - Development (East US)
- `edistgtesteast01` - Test (East US)

### Container Overview

The platform uses 10 specialized containers organized by processing stage and purpose:

| Container | Purpose | Tier | Lifecycle | Retention |
|-----------|---------|------|-----------|-----------|
| **landing** | SFTP upload zone | Hot | Delete after 7 days | 7 days |
| **raw** | Validated original EDI | Hot → Cool → Archive | 90d → 2y → 10y | 10 years |
| **quarantine** | Failed validation | Hot | Delete after 90 days | 90 days |
| **metadata** | Ingestion JSON | Hot → Cool | 90d → 2y | 2 years |
| **outbound-staging** | Transformed outputs | Hot | Delete after 30 days | 30 days |
| **processed** | Successfully routed | Cool → Archive | Immediate → 180d | 7 years |
| **archive** | Long-term compliance | Archive | Permanent (10y) | 10 years |
| **config** | Partner configs, mapping rules | Hot | Permanent (versioned) | Permanent |
| **inbound-responses** | Partner responses | Hot → Cool | 30d → 90d | 90 days |
| **orphaned-responses** | Unmatched correlations | Hot | Delete after 90 days | 90 days |

### Data Flow Across Zones

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    EDI FILE LIFECYCLE FLOW                           │
└─────────────────────────────────────────────────────────────────────┘

INBOUND FLOW:
1. Partner SFTP → landing/ (Event Grid trigger)
2. Validation → raw/ (original preserved) OR quarantine/ (validation failed)
3. ADF Pipeline → processing → processed/ (successfully routed)
4. Lifecycle Policy → Cool (90d) → Archive (2y) → Delete (10y)

OUTBOUND FLOW:
5. Mapper → outbound-staging/pending/ (transformed response)
6. Outbound Orchestrator → outbound-staging/delivered/ (envelope generated)
7. SFTP Connector → partner SFTP (delivery complete)
8. Lifecycle Policy → Delete after 30 days

PARTNER RESPONSE FLOW:
9. Partner → SFTP → inbound-responses/raw/ (partner acknowledgment)
10. Inbound Mapper → inbound-responses/processed/ (canonical transformation)
11. Correlation Match → processed/ (correlated to routing ID)
12. Correlation Failed → orphaned-responses/ (investigate within 90 days)
13. Lifecycle Policy → Delete after 90 days
```

### Path Structure Standards

All containers use hierarchical date-based partitioning for efficient querying:

**Standard Pattern**: `{container}/{YYYY}/{MM}/{DD}/{filename}`

**Examples**:
```text
landing/PARTNERA/2025/01/06/PARTNERA_837P_20250106153045.edi
raw/2025/01/06/abc123-def456_PARTNERA_837P.edi
quarantine/2025/01/06/naming-violation/INVALID_FILE_20250106.txt
metadata/2025/01/06/abc123-def456.json
outbound-staging/PARTNERA/pending/PARTNERA_271_000123456.edi
processed/2025/01/06/abc123-def456_PARTNERA_837P.edi
archive/2025/archive-2025-01.tar.gz
config/partners/PARTNERA/partner-config-v2.json
inbound-responses/raw/PARTNERA_999_000789012_20250106.edi
orphaned-responses/2025/01/06/PARTNERB_271_000555555.edi
```

**Benefits**:
- **Partition Pruning**: Queries filter by date path for performance
- **Retention Policies**: Lifecycle rules apply to date-based prefixes
- **Troubleshooting**: Easy to locate files by date range
- **Cost Tracking**: Blob inventory reports costs by date partition

---

## Container Specifications

### 1. Landing Container

**Purpose**: Temporary staging area for partner SFTP uploads before validation.

**Access Pattern**: Write-only for SFTP service, read-only for ingestion validation function.

**Path Structure**:
```text
landing/{partner-code}/{YYYY}/{MM}/{DD}/{filename}
```

**Example**:
```text
landing/PARTNERA/2025/01/06/PARTNERA_837P_20250106153045.edi
landing/INTERNALCLAIMS/2025/01/06/CLAIMS_834_20250106140000.edi
```

**Blob Metadata**: None (transient storage)

**Lifecycle Policy**:
- **Tier**: Hot (write-optimized)
- **Retention**: 7 days
- **Action**: Delete after 7 days (files should be processed within minutes)

**Use Cases**:
- SFTP upload target
- Event Grid trigger source for ingestion validation
- Temporary buffer for file naming/authorization checks

**Error Scenarios**:
- Files older than 7 days indicate ingestion pipeline failure
- Alert triggers for files not processed within 1 hour

### 2. Raw Container

**Purpose**: Long-term storage of validated original EDI files for event sourcing and replay.

**Access Pattern**: Write-once, read-many (operational access 0-90 days, archive access > 2 years).

**Path Structure**:
```text
raw/{YYYY}/{MM}/{DD}/{ingestion-id}_{partner-code}_{transaction-type}.edi
```

**Example**:
```text
raw/2025/01/06/abc123-def456_PARTNERA_837P.edi
raw/2025/01/06/xyz789-ghi012_INTERNALCLAIMS_834.edi
```

**Blob Metadata**:
```json
{
  "ingestionId": "abc123-def456",
  "partnerCode": "PARTNERA",
  "transactionSet": "837P",
  "originalFileName": "PARTNERA_837P_20250106153045.edi",
  "receivedUtc": "2025-01-06T15:30:45Z",
  "checksumSha256": "a1b2c3d4e5f6...",
  "fileSize": "102400",
  "interchangeControl": "000123456",
  "validationStatus": "VALID"
}
```

**Lifecycle Policy**:
- **0-90 days**: Hot tier (< 1 ms access for operational replay)
- **90 days - 2 years**: Cool tier (occasional access for investigations)
- **2-10 years**: Archive tier (compliance retention, 1-15 hrs rehydration)
- **> 10 years**: Delete (HIPAA retention complete)

**Use Cases**:
- Event replay (reprocess failed transactions)
- Audit investigations (retrieve original file for dispute resolution)
- Partner reconciliation (compare sent vs. received files)
- Compliance inquiries (legal hold, regulatory audits)

**Integrity Checks**:
- SHA256 hash stored in metadata and database for idempotency
- Blob versioning enabled (30-day history)
- Soft delete enabled (30-day recovery window)

### 3. Quarantine Container

**Purpose**: Isolates files that failed validation for investigation and remediation.

**Access Pattern**: Write for failed validations, read for operations team investigation.

**Path Structure**:
```text
quarantine/{YYYY}/{MM}/{DD}/{reason}/{filename}
```

**Quarantine Reasons** (subfolder organization):
- `naming-violation/` - File name doesn't match pattern
- `unauthorized-partner/` - Partner not in authorization list
- `checksum-mismatch/` - File integrity verification failed
- `virus-detected/` - Malware scan flagged file
- `structural-invalid/` - EDI syntax errors (ISA/IEA mismatch)

**Example**:
```text
quarantine/2025/01/06/naming-violation/INVALID_FILE_20250106.txt
quarantine/2025/01/06/unauthorized-partner/UNKNOWNPARTNER_837_20250106.edi
```

**Blob Metadata**:
```json
{
  "quarantineReason": "NAMING_VIOLATION",
  "originalFileName": "INVALID_FILE_20250106.txt",
  "receivedUtc": "2025-01-06T15:30:45Z",
  "validationErrors": "File name must match pattern: {partner}_{transaction}_{timestamp}.edi",
  "remediationStatus": "PENDING",
  "assignedTo": ""
}
```

**Lifecycle Policy**:
- **Tier**: Hot (investigation required)
- **Retention**: 90 days
- **Action**: Delete after 90 days (assume resolved or abandoned)

**Operations Workflow**:
1. Alert fires for new quarantine file
2. Operations team investigates via Azure Portal or Storage Explorer
3. Remediation actions:
   - **Rename & Resubmit**: Correct file name, move to landing
   - **Partner Onboarding**: Add partner to authorization list, reprocess
   - **Reject**: Document reason, notify partner, delete file
4. Update `remediationStatus` metadata to `RESOLVED` or `REJECTED`

### 4. Metadata Container

**Purpose**: Stores ingestion metadata JSON for each processed file (correlation, routing, audit trail).

**Access Pattern**: Write during ingestion, read during investigations and reporting.

**Path Structure**:
```text
metadata/{YYYY}/{MM}/{DD}/{ingestion-id}.json
```

**Example**:
```text
metadata/2025/01/06/abc123-def456.json
```

**Metadata JSON Schema**:
```json
{
  "ingestionId": "abc123-def456",
  "partnerCode": "PARTNERA",
  "transactionSet": "837P",
  "originalFileName": "PARTNERA_837P_20250106153045.edi",
  "blobUri": "https://edistgprodeast01.blob.core.windows.net/raw/2025/01/06/abc123-def456_PARTNERA_837P.edi",
  "receivedUtc": "2025-01-06T15:30:45Z",
  "validatedUtc": "2025-01-06T15:30:47Z",
  "fileSize": 102400,
  "checksumSha256": "a1b2c3d4e5f6...",
  "interchangeControl": "000123456",
  "groupControl": "123",
  "transactionControl": "0001",
  "validationStatus": "VALID",
  "routingDecision": {
    "routingId": "route-001",
    "targetSystem": "INTERNAL-CLAIMS",
    "routingReason": "ClaimType=Professional, PartnerCode=PARTNERA"
  },
  "processingEvents": [
    {
      "eventType": "INGESTION_STARTED",
      "timestamp": "2025-01-06T15:30:45Z",
      "component": "func-edi-ingestion"
    },
    {
      "eventType": "VALIDATION_COMPLETED",
      "timestamp": "2025-01-06T15:30:47Z",
      "component": "func-edi-validation"
    },
    {
      "eventType": "ROUTING_COMPLETED",
      "timestamp": "2025-01-06T15:30:50Z",
      "component": "func-edi-router"
    }
  ]
}
```

**Lifecycle Policy**:
- **0-90 days**: Hot tier (operational queries)
- **90 days - 2 years**: Cool tier (historical analysis)
- **> 2 years**: Delete (detailed events in Application Insights)

**Use Cases**:
- Correlation lookups (find routing ID for acknowledgment)
- Processing timeline reconstruction (troubleshoot latency)
- Audit reports (compliance queries for date range)

### 5. Outbound-Staging Container

**Purpose**: Temporary storage for transformed mapper outputs and assembled outbound files awaiting delivery.

**Access Pattern**: Write by mappers, read by outbound orchestrator, read/update by SFTP connector.

**Path Structure**:
```text
outbound-staging/{partner-id}/pending/{filename}
outbound-staging/{partner-id}/delivered/{filename}
outbound-staging/{partner-id}/errors/{filename}
```

**Example**:
```text
outbound-staging/PARTNERA/pending/PARTNERA_271_000123456_20250106153045.edi
outbound-staging/PARTNERA/delivered/PARTNERA_271_000123456_20250106153045.edi
outbound-staging/PARTNERA/errors/PARTNERA_271_FAILED_20250106153045.edi
```

**Blob Metadata**:
```json
{
  "outboundFileId": "out-abc123-def456",
  "partnerCode": "PARTNERA",
  "transactionType": "271",
  "isa13": "000123456",
  "gs06": "123",
  "st02": "0001",
  "createdUtc": "2025-01-06T15:30:45Z",
  "deliveryStatus": "PENDING",
  "deliveryAttempts": "0",
  "lastDeliveryAttemptUtc": "",
  "errorMessage": ""
}
```

**Lifecycle Policy**:
- **Tier**: Hot (active delivery)
- **Retention**: 30 days
- **Action**: Delete after 30 days (delivered files archived to outbound container)

**Delivery Workflow**:
1. Mapper writes transformed file to `pending/`
2. Outbound orchestrator reads `pending/`, generates envelope, updates metadata
3. SFTP Connector:
   - Success: Move to `delivered/`, update `deliveryStatus = DELIVERED`
   - Failure: Increment `deliveryAttempts`, update `errorMessage`
   - Max Retries (3): Move to `errors/`, alert operations team

### 6. Processed Container

**Purpose**: Long-term storage of successfully routed files (after ADF pipeline completes).

**Access Pattern**: Write-once after routing, read-rarely (historical queries only).

**Path Structure**:
```text
processed/{YYYY}/{MM}/{DD}/{ingestion-id}_{partner-code}_{transaction-type}.edi
```

**Example**:
```text
processed/2025/01/06/abc123-def456_PARTNERA_837P.edi
```

**Blob Metadata**:
```json
{
  "ingestionId": "abc123-def456",
  "partnerCode": "PARTNERA",
  "transactionSet": "837P",
  "routingId": "route-001",
  "targetSystem": "INTERNAL-CLAIMS",
  "processedUtc": "2025-01-06T15:31:00Z",
  "pipelineRunId": "pipeline-run-abc123"
}
```

**Lifecycle Policy**:
- **0 days**: Cool tier (immediate transition after write)
- **180 days**: Archive tier (compliance storage, 1-15 hrs rehydration)
- **7 years**: Delete (HIPAA retention complete)

**Use Cases**:
- Historical reporting (transaction volume by partner/date)
- Compliance audits (retrieve processed files for specific date range)

**Note**: This container has aggressive tiering to minimize costs. Operational access should use `raw` container instead.

### 7. Archive Container

**Purpose**: Long-term cold storage for compliance retention (10-year HIPAA requirement).

**Access Pattern**: Write-rarely (monthly tar.gz creation), read-rarely (legal hold, regulatory audit).

**Path Structure**:
```text
archive/{YYYY}/archive-{YYYY-MM}.tar.gz
```

**Example**:
```text
archive/2025/archive-2025-01.tar.gz
archive/2024/archive-2024-12.tar.gz
```

**Archive Creation Process** (monthly job):
1. Query blobs in `raw` and `processed` containers older than 2 years
2. Download blobs, create tar.gz archive
3. Upload archive to `archive/` container (Archive tier)
4. Verify archive integrity (checksum)
5. Delete source blobs (moved to archive)

**Blob Metadata**:
```json
{
  "archiveMonth": "2025-01",
  "fileCount": "45678",
  "totalSize": "10737418240",
  "checksumSha256": "archive-checksum...",
  "createdUtc": "2025-02-01T02:00:00Z"
}
```

**Lifecycle Policy**:
- **Tier**: Archive (default)
- **Retention**: 10 years
- **Action**: Delete after 10 years (HIPAA retention complete)

**Rehydration Process** (legal hold scenario):
1. Identify required archive file by date range
2. Initiate rehydration: `az storage blob set-tier --name archive-2025-01.tar.gz --tier Hot --rehydrate-priority High`
3. Wait 1-15 hours (High priority < 1 hr)
4. Download archive, extract files, provide to legal team

### 8. Config Container

**Purpose**: Stores partner configurations, mapping rules, validation schemas (centralized repository).

**Access Pattern**: Read-frequently (mappers, validators), write-rarely (configuration updates).

**Path Structure**:
```text
config/partners/{partner-code}/partner-config-v{version}.json
config/mappers/claim-systems/{system-id}/{mapping-rule}.json
config/mappers/trading-partners/{partner-id}/{mapping-rule}.json
config/schemas/x12/{transaction-type}-schema-v{version}.xsd
config/routing/routing-rules-v{version}.json
```

**Example**:
```text
config/partners/PARTNERA/partner-config-v2.json
config/mappers/claim-systems/INTERNAL-CLAIMS/837-to-xml-v1.json
config/mappers/trading-partners/PARTNERA/837-outbound-customization.json
config/schemas/x12/837P-schema-v5010.xsd
config/routing/routing-rules-v3.json
```

**Blob Metadata**:
```json
{
  "configType": "PARTNER_CONFIG",
  "partnerCode": "PARTNERA",
  "version": "2",
  "createdBy": "admin@example.com",
  "createdUtc": "2025-01-06T10:00:00Z",
  "approvedBy": "manager@example.com",
  "approvedUtc": "2025-01-06T11:00:00Z",
  "effectiveDate": "2025-01-07T00:00:00Z",
  "status": "ACTIVE"
}
```

**Lifecycle Policy**:
- **Tier**: Hot (frequent reads by mappers/validators)
- **Retention**: Permanent (versioned, historical configs preserved)
- **Versioning**: Enabled (30-day version history for rollback)

**Version Management**:
- New versions appended with `-v{N}` suffix
- Previous versions retained for rollback scenarios
- `status` metadata indicates `ACTIVE`, `DEPRECATED`, or `RETIRED`

### 9. Inbound-Responses Container

**Purpose**: Stores partner responses (271, 277, 835) received via SFTP before correlation and processing.

**Access Pattern**: Write by SFTP connector, read by inbound mapper, delete after processing.

**Path Structure**:
```text
inbound-responses/raw/{partner-code}/{filename}
inbound-responses/processed/{YYYY}/{MM}/{DD}/{filename}
inbound-responses/errors/{partner-code}/{filename}
```

**Example**:
```text
inbound-responses/raw/PARTNERA/PARTNERA_271_000789012_20250106.edi
inbound-responses/processed/2025/01/06/PARTNERA_271_000789012_20250106.edi
inbound-responses/errors/PARTNERA/PARTNERA_271_PARSE_FAILED_20250106.edi
```

**Blob Metadata**:
```json
{
  "partnerCode": "PARTNERA",
  "transactionType": "271",
  "isa13": "000789012",
  "receivedUtc": "2025-01-06T15:35:00Z",
  "processingStatus": "PENDING",
  "correlationId": "",
  "errorMessage": ""
}
```

**Lifecycle Policy**:
- **0-30 days**: Hot tier (active processing)
- **30-90 days**: Cool tier (investigation window)
- **> 90 days**: Delete (correlated responses moved to processed)

**Processing Workflow**:
1. SFTP Connector downloads partner response, writes to `raw/`
2. Inbound Mapper:
   - Success: Parse, correlate, move to `processed/`, update correlation store
   - Correlation Failed: Move to `orphaned-responses/` for investigation
   - Parse Failed: Move to `errors/`, alert operations team

### 10. Orphaned-Responses Container

**Purpose**: Isolates partner responses that failed correlation lookup (no matching outbound routing ID).

**Access Pattern**: Write by inbound mapper on correlation failure, read by operations team.

**Path Structure**:
```text
orphaned-responses/{YYYY}/{MM}/{DD}/{partner-code}_{transaction-type}_{isa13}.edi
```

**Example**:
```text
orphaned-responses/2025/01/06/PARTNERA_271_000999999.edi
```

**Blob Metadata**:
```json
{
  "partnerCode": "PARTNERA",
  "transactionType": "271",
  "isa13": "000999999",
  "receivedUtc": "2025-01-06T16:00:00Z",
  "correlationAttempts": "3",
  "lastCorrelationAttemptUtc": "2025-01-06T16:05:00Z",
  "orphanReason": "NO_MATCHING_ROUTING_ID",
  "investigationStatus": "PENDING",
  "resolvedBy": "",
  "resolution": ""
}
```

**Lifecycle Policy**:
- **Tier**: Hot (investigation required)
- **Retention**: 90 days
- **Action**: Delete after 90 days (assume unrecoverable)

**Investigation Workflow**:
1. Alert fires for new orphaned response
2. Operations team queries correlation store for potential matches
3. Remediation actions:
   - **Manual Correlation**: Update correlation store with correct mapping, reprocess
   - **Partner Error**: Partner sent response for transaction we never sent (notify partner)
   - **System Gap**: Outbound file failed delivery (regenerate and resend)
4. Update `investigationStatus` metadata to `RESOLVED` or `ABANDONED`

---

## Lifecycle Management

### Cost Optimization Through Automated Tiering

Azure Blob Storage offers three access tiers with different cost profiles:

| Tier | Access Latency | Cost per GB/month | Cost per 10K Transactions | Use Case |
|------|----------------|-------------------|---------------------------|----------|
| **Hot** | < 1 ms | $0.018 | $0.05 (write), $0.004 (read) | Operational files (0-90 days) |
| **Cool** | < 1 ms | $0.010 | $0.10 (write), $0.01 (read) | Infrequent access (90d-2y) |
| **Archive** | 1-15 hours | $0.002 | $0.11 (write), $5.50 (read + rehydrate) | Compliance storage (2-10y) |

**Key Savings**:
- **Cool vs. Hot**: 44% reduction ($0.010 vs. $0.018 per GB)
- **Archive vs. Hot**: 89% reduction ($0.002 vs. $0.018 per GB)
- **Archive vs. Cool**: 80% reduction ($0.002 vs. $0.010 per GB)

### Lifecycle Policy Rules (Bicep)

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' existing = {
  name: 'edistgprodeast01'
}

resource lifecyclePolicy 'Microsoft.Storage/storageAccounts/managementPolicies@2023-01-01' = {
  name: 'default'
  parent: storageAccount
  properties: {
    policy: {
      rules: [
        // Rule 1: Tier raw files (Hot → Cool → Archive → Delete)
        {
          name: 'TierRawFiles'
          enabled: true
          type: 'Lifecycle'
          definition: {
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['raw/']
            }
            actions: {
              baseBlob: {
                tierToCool: {
                  daysAfterModificationGreaterThan: 90
                }
                tierToArchive: {
                  daysAfterModificationGreaterThan: 730  // 2 years
                }
                delete: {
                  daysAfterModificationGreaterThan: 3650  // 10 years
                }
              }
              snapshot: {
                delete: {
                  daysAfterCreationGreaterThan: 90
                }
              }
            }
          }
        }

        // Rule 2: Delete landing files after 7 days
        {
          name: 'DeleteLandingAfter7Days'
          enabled: true
          type: 'Lifecycle'
          definition: {
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['landing/']
            }
            actions: {
              baseBlob: {
                delete: {
                  daysAfterModificationGreaterThan: 7
                }
              }
            }
          }
        }

        // Rule 3: Delete quarantine files after 90 days
        {
          name: 'DeleteQuarantineAfter90Days'
          enabled: true
          type: 'Lifecycle'
          definition: {
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['quarantine/']
            }
            actions: {
              baseBlob: {
                delete: {
                  daysAfterModificationGreaterThan: 90
                }
              }
            }
          }
        }

        // Rule 4: Tier metadata files (Hot → Cool → Delete)
        {
          name: 'TierMetadata'
          enabled: true
          type: 'Lifecycle'
          definition: {
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['metadata/']
            }
            actions: {
              baseBlob: {
                tierToCool: {
                  daysAfterModificationGreaterThan: 90
                }
                delete: {
                  daysAfterModificationGreaterThan: 730  // 2 years
                }
              }
            }
          }
        }

        // Rule 5: Delete outbound-staging after 30 days
        {
          name: 'DeleteOutboundStagingAfter30Days'
          enabled: true
          type: 'Lifecycle'
          definition: {
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['outbound-staging/']
            }
            actions: {
              baseBlob: {
                delete: {
                  daysAfterModificationGreaterThan: 30
                }
              }
            }
          }
        }

        // Rule 6: Tier processed files (Cool immediate → Archive → Delete)
        {
          name: 'TierProcessedFiles'
          enabled: true
          type: 'Lifecycle'
          definition: {
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['processed/']
            }
            actions: {
              baseBlob: {
                tierToCool: {
                  daysAfterModificationGreaterThan: 0  // Immediate Cool tier
                }
                tierToArchive: {
                  daysAfterModificationGreaterThan: 180
                }
                delete: {
                  daysAfterModificationGreaterThan: 2555  // 7 years (HIPAA retention)
                }
              }
            }
          }
        }

        // Rule 7: Tier inbound responses (Hot → Cool → Delete)
        {
          name: 'TierInboundResponses'
          enabled: true
          type: 'Lifecycle'
          definition: {
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['inbound-responses/']
            }
            actions: {
              baseBlob: {
                tierToCool: {
                  daysAfterModificationGreaterThan: 30
                }
                delete: {
                  daysAfterModificationGreaterThan: 90
                }
              }
            }
          }
        }

        // Rule 8: Delete orphaned responses after 90 days
        {
          name: 'DeleteOrphanedResponsesAfter90Days'
          enabled: true
          type: 'Lifecycle'
          definition: {
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['orphaned-responses/']
            }
            actions: {
              baseBlob: {
                delete: {
                  daysAfterModificationGreaterThan: 90
                }
              }
            }
          }
        }
      ]
    }
  }
}
```

### Lifecycle Policy Effectiveness Monitoring

**KQL Query - Verify Files Tier Correctly**:

```kql
AzureStorageBlobLogs
| where OperationName == "GetBlobProperties"
| where TimeGenerated > ago(24h)
| extend Container = extract(@"/([^/]+)/", 1, Uri)
| extend FileAge = datetime_diff('day', now(), todatetime(properties_d.CreationTime))
| extend CurrentTier = tostring(properties_d.AccessTier)
| extend ExpectedTier = case(
    Container == "raw" and FileAge < 90, "Hot",
    Container == "raw" and FileAge >= 90 and FileAge < 730, "Cool",
    Container == "raw" and FileAge >= 730, "Archive",
    Container == "metadata" and FileAge < 90, "Hot",
    Container == "metadata" and FileAge >= 90, "Cool",
    Container == "processed", "Cool",  // Immediate Cool tier
    "Hot"  // Default
)
| where CurrentTier != ExpectedTier
| summarize MisalignedFiles = count() by Container, CurrentTier, ExpectedTier, FileAge
| order by MisalignedFiles desc
```

**Expected Result**: 0 misaligned files (lifecycle policies enforced correctly)

### Manual Tier Management (Ad-Hoc Scenarios)

**Bulk Tier Change** (e.g., move specific partner files to Archive):

```powershell
# Scenario: Partner requested early archival of files from 2023
$storageAccount = "edistgprodeast01"
$container = "raw"
$prefix = "2023/"

# List blobs matching prefix
$blobs = az storage blob list `
    --account-name $storageAccount `
    --container-name $container `
    --prefix $prefix `
    --query "[].name" `
    --output tsv

Write-Host "Found $($blobs.Count) blobs to archive"

# Set tier to Archive for each blob
foreach ($blob in $blobs) {
    az storage blob set-tier `
        --account-name $storageAccount `
        --container-name $container `
        --name $blob `
        --tier Archive
    
    Write-Host "Archived: $blob"
}

Write-Host "✓ Archival complete. Files will be available after 1-15 hour rehydration."
```

**Rehydrate Archive Blob** (legal hold scenario):

```powershell
# Scenario: Legal team needs files from January 2022
$blobName = "2022/01/15/abc123-def456_PARTNERA_837P.edi"

# Check current tier
$properties = az storage blob show `
    --account-name $storageAccount `
    --container-name $container `
    --name $blobName `
    --query "{tier:properties.blobTier,archiveStatus:properties.archiveStatus}" `
    --output json | ConvertFrom-Json

if ($properties.tier -eq "Archive") {
    Write-Host "⏳ Blob is in Archive tier. Initiating rehydration..."
    
    # Rehydrate to Hot tier with High priority
    az storage blob set-tier `
        --account-name $storageAccount `
        --container-name $container `
        --name $blobName `
        --tier Hot `
        --rehydrate-priority High
    
    Write-Host "✓ Rehydration initiated (High priority: < 1 hour)"
    Write-Host "Monitor status with: az storage blob show --name $blobName --query properties.archiveStatus"
} else {
    Write-Host "✓ Blob is already in $($properties.tier) tier (available immediately)"
}
```

---

## Hybrid Storage Model

### Architecture Decision: Database + Blob Storage

The platform uses a **hybrid storage model** where:
- **Azure SQL Database**: Stores metadata, references, and transactional data
- **Azure Blob Storage**: Stores file content (EDI files, JSON, XML)

**Decision Rationale**:

| Factor | Database-Only | Blob-Only | Hybrid (Selected) |
|--------|---------------|-----------|-------------------|
| **Cost** | ❌ Expensive ($0.12/GB) | ✅ Cheap ($0.002-$0.018/GB) | ✅ Optimized (90% savings) |
| **Query Performance** | ✅ Fast (indexed queries) | ❌ Slow (blob metadata queries) | ✅ Fast (SQL metadata + blob content) |
| **ACID Transactions** | ✅ Full ACID support | ❌ Eventual consistency | ✅ Metadata ACID + blob idempotency |
| **Scalability** | ⚠️ Limited (TB-scale) | ✅ Unlimited (PB-scale) | ✅ Best of both |
| **Compliance** | ✅ Built-in retention | ⚠️ Manual lifecycle | ✅ Automated lifecycle |
| **Backup/Recovery** | ✅ Point-in-time restore | ⚠️ Soft delete only | ✅ Both |

### Modified Database Schema

**Before** (Database-Only - Not Recommended):

```sql
CREATE TABLE [dbo].[TransactionBatch]
(
    [TransactionBatchID]    INT IDENTITY(1,1)        NOT NULL,
    [IngestionID]           UNIQUEIDENTIFIER         NOT NULL,
    [PartnerCode]           NVARCHAR(15)             NOT NULL,
    [TransactionType]       NVARCHAR(10)             NOT NULL,
    [OriginalFileName]      NVARCHAR(255)            NOT NULL,
    [RawFileContent]        VARBINARY(MAX)           NOT NULL,  -- ❌ PROBLEM: Stores file in database
    [FileHash]              CHAR(64)                 NOT NULL,
    [FileSize]              BIGINT                   NOT NULL,
    [ReceivedUtc]           DATETIME2(3)             NOT NULL,
    CONSTRAINT [PK_TransactionBatch] PRIMARY KEY CLUSTERED ([TransactionBatchID])
);
```

**After** (Hybrid Model - Recommended):

```sql
CREATE TABLE [dbo].[TransactionBatch]
(
    [TransactionBatchID]    INT IDENTITY(1,1)        NOT NULL,
    [IngestionID]           UNIQUEIDENTIFIER         NOT NULL,
    [PartnerCode]           NVARCHAR(15)             NOT NULL,
    [TransactionType]       NVARCHAR(10)             NOT NULL,
    [OriginalFileName]      NVARCHAR(255)            NOT NULL,
    
    -- ✅ Blob Storage References (instead of VARBINARY(MAX))
    [BlobStorageAccount]    NVARCHAR(100)            NOT NULL,  -- 'edistgprodeast01'
    [BlobContainerName]     NVARCHAR(100)            NOT NULL,  -- 'raw'
    [BlobFileName]          NVARCHAR(500)            NOT NULL,  -- '2025/01/06/abc123-def456_PARTNERA_837P.edi'
    [BlobFullUri]           NVARCHAR(1000)           NOT NULL,  -- Full blob URI for direct access
    [BlobStorageTier]       NVARCHAR(20)             NOT NULL DEFAULT ('Hot'),  -- 'Hot', 'Cool', 'Archive', 'Rehydrating'
    [BlobETag]              NVARCHAR(100)            NULL,      -- ETag for optimistic concurrency
    [BlobLastModified]      DATETIME2(3)             NULL,
    
    -- File Metadata
    [FileHash]              CHAR(64)                 NOT NULL,  -- SHA256 hash for idempotency
    [FileSize]              BIGINT                   NOT NULL,
    [ReceivedUtc]           DATETIME2(3)             NOT NULL,
    
    -- Lifecycle Tracking
    [RetentionStatus]       NVARCHAR(20)             NOT NULL DEFAULT ('Active'),  -- 'Active', 'Archived', 'PurgeEligible', 'Purged'
    [TierTransitionDate]    DATETIME2(3)             NULL,      -- Last tier change date
    [PurgeEligibleDate]     DATE                     NULL,      -- Date when file becomes eligible for purge
    [ArchivedDate]          DATETIME2(3)             NULL,
    [PurgedDate]            DATETIME2(3)             NULL,
    
    CONSTRAINT [PK_TransactionBatch] PRIMARY KEY CLUSTERED ([TransactionBatchID])
);

-- Index for idempotency checks
CREATE UNIQUE INDEX [IX_TransactionBatch_Hash]
    ON [dbo].[TransactionBatch] ([FileHash]);

-- Index for blob URI lookups
CREATE INDEX [IX_TransactionBatch_BlobUri]
    ON [dbo].[TransactionBatch] ([BlobFullUri]);

-- Index for retention queries
CREATE INDEX [IX_TransactionBatch_RetentionStatus]
    ON [dbo].[TransactionBatch] ([RetentionStatus], [PurgeEligibleDate])
    INCLUDE ([BlobFullUri], [BlobStorageTier]);

-- Index for storage tier reporting
CREATE INDEX [IX_TransactionBatch_StorageTier]
    ON [dbo].[TransactionBatch] ([BlobStorageTier], [TierTransitionDate])
    INCLUDE ([FileSize], [PartnerCode], [TransactionType]);
```

**Benefits of Hybrid Model**:
1. **90% Cost Savings**: $7,200 vs. $72,000 over 5 years (100TB scenario)
2. **Fast Metadata Queries**: SQL indexes for partner, date, transaction type queries
3. **Idempotency**: SHA256 hash unique index prevents duplicate processing
4. **Lifecycle Visibility**: `RetentionStatus` and `BlobStorageTier` tracked in database
5. **Direct Blob Access**: `BlobFullUri` for SAS token generation or direct download

### C# Implementation - BlobFileStorageService

```csharp
public class BlobFileStorageService
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly ILogger<BlobFileStorageService> _logger;

    public async Task<BlobStorageReference> UploadTransactionFile(
        Guid ingestionId,
        string partnerCode,
        string transactionType,
        string originalFileName,
        byte[] fileContent,
        string interchangeControl)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient("raw");

        // Generate blob path: raw/{YYYY}/{MM}/{DD}/{ingestionId}_{partner}_{transaction}.edi
        var blobPath = GenerateBlobPath(ingestionId, partnerCode, transactionType);
        var blobClient = containerClient.GetBlobClient(blobPath);

        // Calculate SHA256 hash for idempotency
        var fileHash = ComputeSha256(fileContent);

        // Prepare metadata
        var metadata = new Dictionary<string, string>
        {
            { "OriginalFileName", originalFileName },
            { "IngestionId", ingestionId.ToString() },
            { "PartnerCode", partnerCode },
            { "TransactionType", transactionType },
            { "InterchangeControlNumber", interchangeControl },
            { "ReceivedDate", DateTime.UtcNow.ToString("O") },
            { "FileHash", fileHash },
            { "UploadedBy", "func-edi-ingestion" }
        };

        // Upload with Hot tier and metadata
        using var stream = new MemoryStream(fileContent);
        await blobClient.UploadAsync(
            stream,
            new BlobUploadOptions
            {
                AccessTier = AccessTier.Hot,
                Metadata = metadata,
                HttpHeaders = new BlobHttpHeaders
                {
                    ContentType = "application/edi-x12"
                }
            });

        var properties = await blobClient.GetPropertiesAsync();

        _logger.LogInformation(
            "Uploaded file to blob storage: {BlobPath} ({FileSize} bytes, ETag: {ETag})",
            blobPath, fileContent.Length, properties.Value.ETag);

        return new BlobStorageReference
        {
            StorageAccount = _blobServiceClient.AccountName,
            ContainerName = "raw",
            BlobFileName = blobPath,
            BlobFullUri = blobClient.Uri.ToString(),
            BlobETag = properties.Value.ETag.ToString(),
            FileHash = fileHash,
            FileSize = fileContent.Length,
            StorageTier = "Hot"
        };
    }

    public async Task<byte[]> DownloadTransactionFile(string blobFullUri)
    {
        var blobClient = new BlobClient(new Uri(blobFullUri));

        // Check if blob is in Archive tier (requires rehydration)
        var properties = await blobClient.GetPropertiesAsync();
        if (properties.Value.AccessTier == AccessTier.Archive)
        {
            if (properties.Value.ArchiveStatus == "rehydrate-pending-to-hot" ||
                properties.Value.ArchiveStatus == "rehydrate-pending-to-cool")
            {
                throw new BlobArchivedException(
                    $"Blob is currently rehydrating. Status: {properties.Value.ArchiveStatus}. Retry in 1-15 hours.");
            }

            // Initiate rehydration
            _logger.LogWarning(
                "Blob {BlobUri} is in Archive tier. Initiating rehydration to Hot tier.",
                blobFullUri);

            await blobClient.SetAccessTierAsync(
                AccessTier.Hot,
                rehydratePriority: RehydratePriority.Standard);

            throw new BlobArchivedException(
                "Blob rehydration initiated (Standard priority: 1-15 hours). Retry after rehydration completes.");
        }

        // Download blob content
        var downloadResponse = await blobClient.DownloadContentAsync();
        
        _logger.LogInformation(
            "Downloaded file from blob storage: {BlobUri} ({ContentLength} bytes)",
            blobFullUri, downloadResponse.Value.Details.ContentLength);

        return downloadResponse.Value.Content.ToArray();
    }

    private string GenerateBlobPath(Guid ingestionId, string partnerCode, string transactionType)
    {
        var now = DateTime.UtcNow;
        var fileName = $"{ingestionId}_{partnerCode}_{transactionType}.edi";
        return $"{now:yyyy}/{now:MM}/{now:dd}/{fileName}";
    }

    private string ComputeSha256(byte[] data)
    {
        using var sha256 = SHA256.Create();
        var hashBytes = sha256.ComputeHash(data);
        return BitConverter.ToString(hashBytes).Replace("-", "").ToLowerInvariant();
    }
}

public class BlobStorageReference
{
    public string StorageAccount { get; set; }
    public string ContainerName { get; set; }
    public string BlobFileName { get; set; }
    public string BlobFullUri { get; set; }
    public string BlobETag { get; set; }
    public string FileHash { get; set; }
    public long FileSize { get; set; }
    public string StorageTier { get; set; }
}

public class BlobArchivedException : Exception
{
    public BlobArchivedException(string message) : base(message) { }
}
```

### Idempotency Check - Stored Procedure

```sql
CREATE PROCEDURE [dbo].[usp_CheckFileIdempotency]
    @FileHash           CHAR(64),
    @AlreadyProcessed   BIT OUTPUT,
    @ExistingBatchID    INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        @AlreadyProcessed = CASE WHEN COUNT(*) > 0 THEN 1 ELSE 0 END,
        @ExistingBatchID = MAX(TransactionBatchID)
    FROM dbo.TransactionBatch
    WHERE FileHash = @FileHash;

    IF @AlreadyProcessed = 1
    BEGIN
        -- Return existing batch details
        SELECT
            TransactionBatchID,
            IngestionID,
            PartnerCode,
            TransactionType,
            OriginalFileName,
            BlobFullUri,
            FileSize,
            ReceivedUtc,
            RetentionStatus,
            BlobStorageTier
        FROM dbo.TransactionBatch
        WHERE TransactionBatchID = @ExistingBatchID;
    END;

    RETURN 0;
END;
GO
```

**Usage in Ingestion Function**:

```csharp
// Check idempotency before processing
var fileHash = ComputeSha256(fileContent);

using var connection = new SqlConnection(_connectionString);
await connection.OpenAsync();

using var command = new SqlCommand("dbo.usp_CheckFileIdempotency", connection)
{
    CommandType = CommandType.StoredProcedure
};

command.Parameters.AddWithValue("@FileHash", fileHash);
var alreadyProcessed = new SqlParameter("@AlreadyProcessed", SqlDbType.Bit)
{
    Direction = ParameterDirection.Output
};
var existingBatchId = new SqlParameter("@ExistingBatchID", SqlDbType.Int)
{
    Direction = ParameterDirection.Output
};
command.Parameters.Add(alreadyProcessed);
command.Parameters.Add(existingBatchId);

using var reader = await command.ExecuteReaderAsync();

if ((bool)alreadyProcessed.Value)
{
    // Read existing batch details
    await reader.ReadAsync();
    var existingBlobUri = reader.GetString(reader.GetOrdinal("BlobFullUri"));
    
    _logger.LogWarning(
        "Duplicate file detected: FileHash={FileHash}, ExistingBatchID={ExistingBatchID}, BlobUri={BlobUri}",
        fileHash, existingBatchId.Value, existingBlobUri);

    return new IngestionResult
    {
        Status = "DUPLICATE",
        Message = $"File already processed (Batch ID: {existingBatchId.Value})",
        ExistingBlobUri = existingBlobUri
    };
}

// Continue with file upload and processing
```

### File Retrieval - Stored Procedure

```sql
CREATE PROCEDURE [dbo].[usp_GetTransactionFileReference]
    @TransactionBatchID INT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        TransactionBatchID,
        IngestionID,
        PartnerCode,
        TransactionType,
        OriginalFileName,
        BlobFullUri,
        BlobStorageTier,
        BlobETag,
        FileHash,
        FileSize,
        ReceivedUtc,
        RetentionStatus,
        TierTransitionDate,
        -- Calculate access latency info
        CASE BlobStorageTier
            WHEN 'Archive' THEN 'Rehydration required (1-15 hours)'
            WHEN 'Cool' THEN 'Available (few seconds)'
            WHEN 'Hot' THEN 'Immediate access'
            ELSE 'Unknown tier'
        END AS AccessLatency
    FROM dbo.TransactionBatch
    WHERE TransactionBatchID = @TransactionBatchID;

    RETURN 0;
END;
GO
```

### Lifecycle Management Job - Stored Procedure

```sql
CREATE PROCEDURE [dbo].[usp_ManageFileLifecycle]
    @DryRun BIT = 1  -- Preview changes without updating
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @ArchiveThresholdDays INT = 730;  -- 2 years
    DECLARE @PurgeThresholdDays INT = 3650;   -- 10 years
    DECLARE @ArchiveDate DATETIME2 = DATEADD(DAY, -@ArchiveThresholdDays, GETUTCDATE());
    DECLARE @PurgeDate DATE = CAST(DATEADD(DAY, -@PurgeThresholdDays, GETUTCDATE()) AS DATE);

    -- Step 1: Mark files older than 2 years for Archive tier
    -- (Actual tiering performed by Azure lifecycle policy, we just track status)
    IF @DryRun = 0
    BEGIN
        UPDATE dbo.TransactionBatch
        SET
            RetentionStatus = 'Archived',
            ArchivedDate = GETUTCDATE(),
            TierTransitionDate = GETUTCDATE()
        WHERE ReceivedUtc < @ArchiveDate
          AND RetentionStatus = 'Active'
          AND BlobStorageTier != 'Archive';
    END;

    -- Step 2: Calculate files eligible for purge (10+ years old)
    IF @DryRun = 0
    BEGIN
        UPDATE dbo.TransactionBatch
        SET
            RetentionStatus = 'PurgeEligible',
            PurgeEligibleDate = @PurgeDate
        WHERE ReceivedUtc < CAST(@PurgeDate AS DATETIME2)
          AND RetentionStatus IN ('Active', 'Archived')
          AND PurgeEligibleDate IS NULL;
    END;

    -- Step 3: Report lifecycle changes
    SELECT
        LifecycleAction = CASE
            WHEN ReceivedUtc < @ArchiveDate AND RetentionStatus = 'Active' THEN 'ARCHIVE'
            WHEN ReceivedUtc < CAST(@PurgeDate AS DATETIME2) AND RetentionStatus IN ('Active', 'Archived') THEN 'PURGE_ELIGIBLE'
            ELSE 'NO_ACTION'
        END,
        FileCount = COUNT(*),
        AvgAge = AVG(DATEDIFF(DAY, ReceivedUtc, GETUTCDATE())),
        MinAge = MIN(DATEDIFF(DAY, ReceivedUtc, GETUTCDATE())),
        MaxAge = MAX(DATEDIFF(DAY, ReceivedUtc, GETUTCDATE()))
    FROM dbo.TransactionBatch
    WHERE (@DryRun = 1 AND ReceivedUtc < @ArchiveDate)
       OR (@DryRun = 1 AND ReceivedUtc < CAST(@PurgeDate AS DATETIME2))
    GROUP BY
        CASE
            WHEN ReceivedUtc < @ArchiveDate AND RetentionStatus = 'Active' THEN 'ARCHIVE'
            WHEN ReceivedUtc < CAST(@PurgeDate AS DATETIME2) AND RetentionStatus IN ('Active', 'Archived') THEN 'PURGE_ELIGIBLE'
            ELSE 'NO_ACTION'
        END
    ORDER BY LifecycleAction;

    IF @DryRun = 1
    BEGIN
        PRINT 'DRY RUN MODE: No records updated. Set @DryRun = 0 to apply changes.';
    END;

    RETURN 0;
END;
GO
```

**Scheduled Job** (Azure Function with Timer Trigger):

```csharp
[FunctionName("LifecycleManagementJob")]
public async Task Run(
    [TimerTrigger("0 0 2 * * *")] TimerInfo timer,  // Daily at 2:00 AM UTC
    ILogger log)
{
    log.LogInformation("Starting lifecycle management job at {Timestamp}", DateTime.UtcNow);

    using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    // Run with DryRun = 0 to apply changes
    using var command = new SqlCommand("dbo.usp_ManageFileLifecycle", connection)
    {
        CommandType = CommandType.StoredProcedure
    };
    command.Parameters.AddWithValue("@DryRun", 0);

    using var reader = await command.ExecuteReaderAsync();

    // Log lifecycle changes
    while (await reader.ReadAsync())
    {
        var action = reader.GetString(0);
        var fileCount = reader.GetInt32(1);
        var avgAge = reader.GetInt32(2);

        log.LogInformation(
            "Lifecycle action: {Action}, Files: {FileCount}, Avg Age: {AvgAge} days",
            action, fileCount, avgAge);
    }

    log.LogInformation("Lifecycle management job completed successfully");
}
```

---

## Operations

### Blob Storage Health Checks

**PowerShell Script - Check Container Availability**:

```powershell
$storageAccount = "edistgprodeast01"
$containers = @("landing", "raw", "quarantine", "metadata", "outbound-staging", "processed", "archive", "config", "inbound-responses", "orphaned-responses")

Write-Host "Checking storage account health: $storageAccount" -ForegroundColor Cyan

foreach ($container in $containers) {
    try {
        $blobCount = az storage blob list `
            --account-name $storageAccount `
            --container-name $container `
            --query "length(@)" `
            --output tsv
        
        Write-Host "✓ Container '$container': $blobCount blobs" -ForegroundColor Green
    }
    catch {
        Write-Host "✗ Container '$container': ERROR - $($_.Exception.Message)" -ForegroundColor Red
    }
}
```

### Blob Metadata Bulk Update

**Scenario**: Update `deliveryStatus` metadata for outbound files after manual delivery.

```powershell
$container = "outbound-staging"
$prefix = "PARTNERA/delivered/"

# List blobs with PENDING delivery status
$blobs = az storage blob list `
    --account-name $storageAccount `
    --container-name $container `
    --prefix $prefix `
    --query "[?metadata.deliveryStatus=='PENDING'].name" `
    --output tsv

foreach ($blob in $blobs) {
    # Update metadata
    az storage blob metadata update `
        --account-name $storageAccount `
        --container-name $container `
        --name $blob `
        --metadata "deliveryStatus=DELIVERED" "lastDeliveryAttemptUtc=$(Get-Date -Format 'o')"
    
    Write-Host "Updated: $blob"
}
```

### Archive Blob Rehydration (Batch)

**Scenario**: Legal team needs all files from Q1 2022 for audit.

```powershell
$container = "raw"
$startDate = "2022/01"
$endDate = "2022/03"

# List Archive tier blobs in date range
$blobs = az storage blob list `
    --account-name $storageAccount `
    --container-name $container `
    --prefix $startDate `
    --query "[?properties.blobTier=='Archive'].name" `
    --output tsv

Write-Host "Found $($blobs.Count) archived blobs. Initiating rehydration..." -ForegroundColor Yellow

foreach ($blob in $blobs) {
    # Check if blob is in target date range
    if ($blob -match "^202[2-3]/(0[1-3])/") {
        az storage blob set-tier `
            --account-name $storageAccount `
            --container-name $container `
            --name $blob `
            --tier Hot `
            --rehydrate-priority High
        
        Write-Host "Rehydrating: $blob (High priority: < 1 hour)"
    }
}

Write-Host "✓ Rehydration initiated for $($blobs.Count) blobs. Monitor status with:" -ForegroundColor Green
Write-Host "az storage blob show --name <blob-name> --query properties.archiveStatus"
```

### Blob Inventory Report

**Azure Blob Inventory** (automated daily report):

```bicep
resource blobInventoryPolicy 'Microsoft.Storage/storageAccounts/inventoryPolicies@2023-01-01' = {
  name: 'default'
  parent: storageAccount
  properties: {
    policy: {
      enabled: true
      type: 'Inventory'
      rules: [
        {
          enabled: true
          name: 'DailyInventory'
          destination: 'inventory-reports'
          definition: {
            format: 'CSV'
            schedule: 'Daily'
            objectType: 'Blob'
            schemaFields: [
              'Name'
              'Creation-Time'
              'Last-Modified'
              'Content-Length'
              'Content-MD5'
              'BlobType'
              'AccessTier'
              'AccessTierChangeTime'
              'Metadata'
            ]
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['raw/', 'processed/', 'archive/']
            }
          }
        }
      ]
    }
  }
}
```

**Benefits**:
- Daily CSV report with all blob details
- Track storage tier transitions
- Cost analysis by container/partner
- Compliance reporting (file counts, retention status)

---

## Monitoring

### KQL Queries

#### Storage Tier Distribution

```kql
AzureStorageBlobLogs
| where OperationName == "GetBlobProperties"
| where TimeGenerated > ago(24h)
| extend Container = extract(@"/([^/]+)/", 1, Uri)
| extend Tier = tostring(properties_d.AccessTier)
| summarize BlobCount = dcount(Uri) by Container, Tier
| render columnchart
```

#### Storage Cost Trend (Estimated)

```kql
AzureStorageBlobLogs
| where TimeGenerated > ago(30d)
| extend Container = extract(@"/([^/]+)/", 1, Uri)
| extend Tier = tostring(properties_d.AccessTier)
| extend SizeGB = todouble(properties_d.ContentLength) / (1024.0 * 1024.0 * 1024.0)
| summarize TotalSizeGB = sum(SizeGB) by bin(TimeGenerated, 1d), Tier
| extend CostPerGB = case(
    Tier == "Hot", 0.018,
    Tier == "Cool", 0.010,
    Tier == "Archive", 0.002,
    0.0
)
| extend EstimatedMonthlyCost = TotalSizeGB * CostPerGB
| project TimeGenerated, Tier, TotalSizeGB, EstimatedMonthlyCost
| render timechart
```

#### Quarantine Rate by Reason

```kql
AzureStorageBlobLogs
| where OperationName == "PutBlob"
| where Uri contains "quarantine/"
| where TimeGenerated > ago(7d)
| extend QuarantineReason = extract(@"quarantine/\d{4}/\d{2}/\d{2}/([^/]+)/", 1, Uri)
| summarize FileCount = count() by QuarantineReason
| render piechart
```

#### Archive Rehydration Requests

```kql
AzureStorageBlobLogs
| where OperationName == "SetBlobTier"
| where properties_s contains "rehydrate"
| where TimeGenerated > ago(30d)
| extend Container = extract(@"/([^/]+)/", 1, Uri)
| extend TargetTier = tostring(properties_d.AccessTier)
| extend Priority = tostring(properties_d.RehydratePriority)
| summarize RehydrationCount = count() by bin(TimeGenerated, 1d), Priority
| render timechart
```

#### Storage Health Dashboard (Single Query)

```kql
let TotalFiles = AzureStorageBlobLogs
    | where TimeGenerated > ago(1h)
    | where OperationName == "GetBlobProperties"
    | summarize FileCount = dcount(Uri);
let TotalSizeGB = AzureStorageBlobLogs
    | where TimeGenerated > ago(1h)
    | where OperationName == "GetBlobProperties"
    | extend SizeGB = todouble(properties_d.ContentLength) / (1024.0 * 1024.0 * 1024.0)
    | summarize TotalGB = sum(SizeGB);
let TierDistribution = AzureStorageBlobLogs
    | where TimeGenerated > ago(1h)
    | where OperationName == "GetBlobProperties"
    | extend Tier = tostring(properties_d.AccessTier)
    | summarize FileCount = dcount(Uri) by Tier
    | extend Percentage = (FileCount * 100.0) / toscalar(TotalFiles);
let AvgFileSize = AzureStorageBlobLogs
    | where TimeGenerated > ago(1h)
    | where OperationName == "GetBlobProperties"
    | extend SizeMB = todouble(properties_d.ContentLength) / (1024.0 * 1024.0)
    | summarize AvgSizeMB = avg(SizeMB);
union TotalFiles, TotalSizeGB, TierDistribution, AvgFileSize
| project
    TotalFiles = toint(FileCount),
    TotalSizeGB = round(TotalGB, 2),
    Tier,
    Percentage = round(Percentage, 2),
    AvgFileSizeMB = round(AvgSizeMB, 2)
```

#### Blob Access Patterns (Hot vs. Cool)

```kql
AzureStorageBlobLogs
| where OperationName in ("GetBlob", "GetBlobProperties")
| where TimeGenerated > ago(7d)
| extend Tier = tostring(properties_d.AccessTier)
| extend Latency = todouble(DurationMs)
| summarize
    AccessCount = count(),
    AvgLatency = avg(Latency),
    p95Latency = percentile(Latency, 95)
    by bin(TimeGenerated, 1h), Tier
| render timechart
```

#### Missing Blob References (Integrity Check)

```kql
// SQL query to identify database records without valid blob references
let MissingBlobs = SqlDatabaseQuery
    | where Query == "SELECT * FROM TransactionBatch WHERE BlobFullUri IS NULL OR BlobFullUri = ''"
    | summarize MissingCount = count();
MissingBlobs
| extend AlertMessage = strcat("Found ", MissingCount, " database records with missing blob references")
| project AlertMessage, MissingCount
```

---

## Security

### RBAC (Role-Based Access Control)

**Function App Assignments**:

```bicep
// Ingestion Function: Write to landing/raw/quarantine/metadata
resource ingestionFunctionStorageRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storageAccount.id, ingestionFunctionApp.id, 'StorageBlobDataContributor')
  scope: storageAccount
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')  // Storage Blob Data Contributor
    principalId: ingestionFunctionApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Mapper Function: Read raw, Write outbound-staging
resource mapperFunctionStorageRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storageAccount.id, mapperFunctionApp.id, 'StorageBlobDataContributor')
  scope: storageAccount
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
    principalId: mapperFunctionApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// ADF: Read landing, Write raw/quarantine/metadata/processed
resource adfStorageRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storageAccount.id, dataFactory.id, 'StorageBlobDataContributor')
  scope: storageAccount
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
    principalId: dataFactory.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Operations Team: Read-only access (investigations)
resource opsTeamStorageRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storageAccount.id, opsTeamGroup.id, 'StorageBlobDataReader')
  scope: storageAccount
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '2a2b9908-6ea1-4ae2-8e65-a410df84e7d1')  // Storage Blob Data Reader
    principalId: opsTeamGroup.id
    principalType: 'Group'
  }
}
```

### Private Endpoints

```bicep
resource blobPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-05-01' = {
  name: 'pe-${storageAccount.name}-blob'
  location: location
  properties: {
    subnet: {
      id: subnet.id
    }
    privateLinkServiceConnections: [
      {
        name: 'blob-connection'
        properties: {
          privateLinkServiceId: storageAccount.id
          groupIds: ['blob']
        }
      }
    ]
  }
}

resource privateDnsZoneGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-05-01' = {
  name: 'default'
  parent: blobPrivateEndpoint
  properties: {
    privateDnsZoneConfigs: [
      {
        name: 'blob-privatelink-config'
        properties: {
          privateDnsZoneId: privateDnsZone.id
        }
      }
    ]
  }
}

// Disable public network access
resource storageAccountNetworkRules 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccount.name
  properties: {
    publicNetworkAccess: 'Disabled'
    networkAcls: {
      defaultAction: 'Deny'
      bypass: 'AzureServices'
      ipRules: []
      virtualNetworkRules: []
    }
  }
}
```

### Encryption

**At-Rest Encryption** (automatic):
- **Algorithm**: AES-256
- **Key Management**: Microsoft-managed keys (default) or customer-managed keys (Key Vault)
- **Scope**: All blobs, metadata, and queue messages encrypted automatically

**Customer-Managed Keys** (optional):

```bicep
resource customerManagedKey 'Microsoft.KeyVault/vaults/keys@2023-02-01' = {
  name: 'storage-encryption-key'
  parent: keyVault
  properties: {
    keySize: 4096
    kty: 'RSA'
    keyOps: ['encrypt', 'decrypt', 'wrapKey', 'unwrapKey']
  }
}

resource storageAccountEncryption 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccount.name
  properties: {
    encryption: {
      services: {
        blob: {
          enabled: true
          keyType: 'Account'
        }
      }
      keySource: 'Microsoft.Keyvault'
      keyvaultproperties: {
        keyname: customerManagedKey.name
        keyvaulturi: keyVault.properties.vaultUri
      }
    }
  }
  identity: {
    type: 'SystemAssigned'
  }
}
```

**In-Transit Encryption**:
- **TLS 1.2+**: Enforced via storage account policy
- **HTTPS-Only**: All blob operations require HTTPS

```bicep
resource storageAccountHttpsOnly 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccount.name
  properties: {
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
  }
}
```

### Blob Versioning (Point-in-Time Recovery)

```bicep
resource blobVersioning 'Microsoft.Storage/storageAccounts/blobServices@2023-01-01' = {
  name: 'default'
  parent: storageAccount
  properties: {
    isVersioningEnabled: true
    changeFeed: {
      enabled: true
      retentionInDays: 90
    }
    deleteRetentionPolicy: {
      enabled: true
      days: 30
    }
    containerDeleteRetentionPolicy: {
      enabled: true
      days: 30
    }
  }
}
```

**Benefits**:
- **Accidental Deletion Recovery**: Soft delete retains deleted blobs for 30 days
- **Version History**: All blob modifications tracked for 90 days
- **Point-in-Time Restore**: Restore blobs to any point in the last 90 days

### Audit Logging

```bicep
resource diagnosticSettings 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'blob-diagnostics'
  scope: storageAccount::blobServices
  properties: {
    workspaceId: logAnalyticsWorkspace.id
    logs: [
      {
        category: 'StorageRead'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: 90
        }
      }
      {
        category: 'StorageWrite'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: 90
        }
      }
      {
        category: 'StorageDelete'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: 90
        }
      }
    ]
    metrics: [
      {
        category: 'Transaction'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: 90
        }
      }
      {
        category: 'Capacity'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: 90
        }
      }
    ]
  }
}
```

**Audit Trail Queries**:

```kql
// All blob deletions in last 7 days
AzureStorageBlobLogs
| where OperationName == "DeleteBlob"
| where TimeGenerated > ago(7d)
| project TimeGenerated, Uri, CallerIpAddress, AuthenticationType, RequesterUpn
| order by TimeGenerated desc

// Unauthorized access attempts
AzureStorageBlobLogs
| where StatusCode == 403  // Forbidden
| where TimeGenerated > ago(24h)
| summarize FailedAttempts = count() by CallerIpAddress, Uri
| order by FailedAttempts desc
```

---

## Disaster Recovery

### Geo-Redundant Storage (GRS)

**Production Configuration**:
- **Replication**: GRS (Geo-Redundant Storage)
- **Secondary Region**: Paired region (e.g., West US if primary is East US)
- **RPO**: < 15 minutes (Azure-managed replication)
- **RTO**: < 1 hour (failover initiation + DNS propagation)

**Failover Scenarios**:

1. **Read-Access GRS (RA-GRS)**: Read from secondary region without failover (degraded mode)
2. **Customer-Initiated Failover**: Promote secondary region to primary (manual trigger)
3. **Automatic Failover**: Not supported (requires customer initiation)

### Recovery Scenarios

#### Scenario 1: Accidental Blob Deletion

**Recovery Method**: Soft Delete (30-day retention)

```powershell
# List deleted blobs
az storage blob list `
    --account-name $storageAccount `
    --container-name raw `
    --include d `
    --query "[?properties.deletedTime != null].{name:name,deleted:properties.deletedTime}" `
    --output table

# Undelete specific blob
az storage blob undelete `
    --account-name $storageAccount `
    --container-name raw `
    --name "2025/01/06/abc123-def456_PARTNERA_837P.edi"

Write-Host "✓ Blob undeleted successfully"
```

**RPO**: 0 (immediate recovery)  
**RTO**: < 5 minutes (undelete operation)

#### Scenario 2: Storage Account Corruption/Unavailability

**Recovery Method**: GRS Failover

```powershell
# Check secondary region availability
az storage account show `
    --name $storageAccount `
    --query "secondaryEndpoints.blob" `
    --output tsv

# Initiate failover to secondary region
az storage account failover `
    --name $storageAccount `
    --resource-group $resourceGroup

Write-Host "⏳ Failover initiated. Primary region will switch to secondary. ETA: 1 hour"

# Update application endpoints to use secondary region
$newEndpoint = "https://${storageAccount}-secondary.blob.core.windows.net/"
Write-Host "Update application connection strings to: $newEndpoint"
```

**RPO**: < 15 minutes (data replicated to secondary)  
**RTO**: < 1 hour (failover duration + application endpoint updates)

**Post-Failover Actions**:
1. Update Azure Function app settings with new storage endpoint
2. Update ADF linked services with new connection string
3. Verify blob access from all components
4. Monitor for data consistency issues

#### Scenario 3: Event Replay from Archive

**Recovery Method**: Rehydrate Archive Blobs + Reprocess

```powershell
# Scenario: Reprocess all 837 claims from January 2024 after system bug fix
$container = "raw"
$prefix = "2024/01"
$transactionType = "837P"

# List Archive blobs matching criteria
$blobs = az storage blob list `
    --account-name $storageAccount `
    --container-name $container `
    --prefix $prefix `
    --query "[?properties.blobTier=='Archive' && contains(name,'$transactionType')].name" `
    --output tsv

Write-Host "Found $($blobs.Count) archived 837P files. Initiating rehydration..."

foreach ($blob in $blobs) {
    # Rehydrate to Hot tier
    az storage blob set-tier `
        --account-name $storageAccount `
        --container-name $container `
        --name $blob `
        --tier Hot `
        --rehydrate-priority High
}

Write-Host "✓ Rehydration initiated. Files will be available in < 1 hour (High priority)."
Write-Host "After rehydration, trigger reprocessing via ADF pipeline."
```

**RPO**: 0 (original files preserved)  
**RTO**: 1-15 hours (rehydration) + reprocessing time

### Backup Strategy

**Database Metadata Backup**:
- **Azure SQL Automated Backups**: Point-in-time restore (35 days)
- **Frequency**: Continuous (transaction log backups every 5-10 minutes)
- **Retention**: 7 days (default), 35 days (configured)

**Blob Content Backup**:
- **GRS Replication**: Automatic replication to secondary region
- **Blob Versioning**: 90-day version history for accidental modifications
- **Soft Delete**: 30-day retention for deleted blobs

**Critical Configuration Backup**:
- **Config Container**: Partner configs, mapping rules, schemas
- **Version Control**: Git repository for all JSON/XSD files
- **Deployment Pipeline**: Infrastructure as Code (Bicep) in Git

---

## Cost Optimization

### 5-Year Total Cost of Ownership (TCO)

**Scenario**: Healthcare EDI platform processing 500 files/day average (182,500 files/year), 500 KB average file size.

**Assumptions**:
- Growth rate: 10% per year
- Database storage: $0.12/GB/month
- Hot tier: $0.018/GB/month
- Cool tier: $0.010/GB/month
- Archive tier: $0.002/GB/month

#### Option A: Database-Only Storage (Not Recommended)

| Year | Files/Day | Total Files | Total Size (GB) | Monthly Cost | Annual Cost |
|------|-----------|-------------|-----------------|--------------|-------------|
| 1 | 500 | 182,500 | 91 | $10.95 | $131.40 |
| 2 | 550 | 382,750 | 191 | $22.92 | $275.04 |
| 3 | 605 | 603,578 | 302 | $36.24 | $434.88 |
| 4 | 666 | 846,594 | 423 | $50.76 | $609.12 |
| 5 | 732 | 1,113,738 | 557 | $66.84 | $802.08 |
| **5-Year Total** | | **1,113,738** | **557 GB** | | **$2,252.52** |

**Cost per GB**: $0.12/month × 557 GB × 60 months = **$4,051.20**

#### Option B: Hybrid Storage (Recommended)

| Year | Files/Day | Total Files | Hot (GB) | Cool (GB) | Archive (GB) | Monthly Cost | Annual Cost |
|------|-----------|-------------|----------|-----------|--------------|--------------|-------------|
| 1 | 500 | 182,500 | 91 (0-90d) | 0 | 0 | $1.64 | $19.68 |
| 2 | 550 | 382,750 | 100 | 91 (90d-2y) | 0 | $2.71 | $32.52 |
| 3 | 605 | 603,578 | 110 | 100 | 91 (2y+) | $3.20 | $38.40 |
| 4 | 666 | 846,594 | 121 | 110 | 192 | $3.56 | $42.72 |
| 5 | 732 | 1,113,738 | 133 | 121 | 303 | $3.94 | $47.28 |
| **5-Year Total** | | **1,113,738** | **557 GB** | | | | **$180.60** |

**Cost Breakdown** (Year 5):
- Hot tier: 133 GB × $0.018 = $2.39/month
- Cool tier: 121 GB × $0.010 = $1.21/month
- Archive tier: 303 GB × $0.002 = $0.61/month
- **Total**: $4.21/month = $50.52/year

#### Cost Comparison

| Metric | Database-Only | Hybrid Storage | Savings |
|--------|---------------|----------------|---------|
| **5-Year Total Cost** | $2,252.52 | $180.60 | $2,071.92 (92%) |
| **Year 5 Monthly Cost** | $66.84 | $3.94 | $62.90 (94%) |
| **Cost per GB (Year 5)** | $0.12 | $0.0076 | 94% reduction |
| **Total Storage (Year 5)** | 557 GB | 557 GB | Same capacity |

**ROI**: 1,146% (savings ÷ implementation cost)  
**Break-Even**: Month 2 (hybrid storage pays for itself immediately)

### Cost Optimization Strategies

1. **Aggressive Lifecycle Policies**: Move files to Archive tier at 2 years (vs. 5 years default)
2. **Delete Temporary Containers Early**: Landing (7 days), outbound-staging (30 days), quarantine (90 days)
3. **Archive Compression**: Tar.gz monthly archives (60-70% compression ratio)
4. **Reserved Capacity**: Purchase 1-year reserved capacity for predictable storage (5-25% discount)
5. **Regional Selection**: Choose regions with lower storage costs (e.g., Central US vs. East US)

### Monthly Cost Monitoring

```kql
// Estimated monthly cost by container
AzureStorageBlobLogs
| where TimeGenerated > startofmonth(now())
| extend Container = extract(@"/([^/]+)/", 1, Uri)
| extend Tier = tostring(properties_d.AccessTier)
| extend SizeGB = todouble(properties_d.ContentLength) / (1024.0 * 1024.0 * 1024.0)
| summarize TotalSizeGB = sum(SizeGB) by Container, Tier
| extend CostPerGB = case(
    Tier == "Hot", 0.018,
    Tier == "Cool", 0.010,
    Tier == "Archive", 0.002,
    0.0
)
| extend EstimatedMonthlyCost = TotalSizeGB * CostPerGB
| project Container, Tier, TotalSizeGB = round(TotalSizeGB, 2), EstimatedMonthlyCost = round(EstimatedMonthlyCost, 2)
| order by EstimatedMonthlyCost desc
```

**Cost Alert** (Azure Monitor):

```bicep
resource costAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'alert-storage-cost-threshold'
  location: 'global'
  properties: {
    severity: 2
    enabled: true
    scopes: [storageAccount.id]
    evaluationFrequency: 'PT1H'
    windowSize: 'PT24H'
    criteria: {
      allOf: [
        {
          name: 'UsedCapacity'
          metricName: 'UsedCapacity'
          metricNamespace: 'Microsoft.Storage/storageAccounts'
          operator: 'GreaterThan'
          threshold: 1099511627776  // 1 TB in bytes
          timeAggregation: 'Average'
        }
      ]
    }
    actions: [
      {
        actionGroupId: actionGroup.id
      }
    ]
  }
}
```

---

## Infrastructure Deployment (Bicep)

### Complete Storage Account Module

```bicep
@description('Environment name (dev, test, prod)')
param environment string

@description('Azure region')
param location string = resourceGroup().location

@description('SKU for storage account')
@allowed(['Standard_LRS', 'Standard_GRS'])
param storageSku string = environment == 'prod' ? 'Standard_GRS' : 'Standard_LRS'

@description('Log Analytics workspace ID for diagnostics')
param workspaceId string

// Storage account name
var storageAccountName = 'edistg${environment}${location}01'

// Container names
var containerNames = [
  'landing'
  'raw'
  'quarantine'
  'metadata'
  'outbound-staging'
  'processed'
  'archive'
  'config'
  'inbound-responses'
  'orphaned-responses'
]

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  kind: 'StorageV2'
  sku: {
    name: storageSku
  }
  properties: {
    accessTier: 'Hot'
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    isHnsEnabled: true  // Hierarchical namespace (ADLS Gen2)
    publicNetworkAccess: 'Disabled'
    networkAcls: {
      defaultAction: 'Deny'
      bypass: 'AzureServices'
    }
  }
}

resource blobServices 'Microsoft.Storage/storageAccounts/blobServices@2023-01-01' = {
  name: 'default'
  parent: storageAccount
  properties: {
    isVersioningEnabled: true
    changeFeed: {
      enabled: true
      retentionInDays: 90
    }
    deleteRetentionPolicy: {
      enabled: true
      days: 30
    }
    containerDeleteRetentionPolicy: {
      enabled: true
      days: 30
    }
  }
}

resource containers 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = [for containerName in containerNames: {
  name: containerName
  parent: blobServices
  properties: {
    publicAccess: 'None'
  }
}]

resource lifecyclePolicy 'Microsoft.Storage/storageAccounts/managementPolicies@2023-01-01' = {
  name: 'default'
  parent: storageAccount
  properties: {
    policy: {
      rules: [
        {
          name: 'TierRawFiles'
          enabled: true
          type: 'Lifecycle'
          definition: {
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['raw/']
            }
            actions: {
              baseBlob: {
                tierToCool: { daysAfterModificationGreaterThan: 90 }
                tierToArchive: { daysAfterModificationGreaterThan: 730 }
                delete: { daysAfterModificationGreaterThan: 3650 }
              }
              snapshot: {
                delete: { daysAfterCreationGreaterThan: 90 }
              }
            }
          }
        }
        {
          name: 'DeleteLandingAfter7Days'
          enabled: true
          type: 'Lifecycle'
          definition: {
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['landing/']
            }
            actions: {
              baseBlob: {
                delete: { daysAfterModificationGreaterThan: 7 }
              }
            }
          }
        }
        // Additional rules omitted for brevity (see Lifecycle Management section)
      ]
    }
  }
}

resource diagnosticSettings 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'blob-diagnostics'
  scope: blobServices
  properties: {
    workspaceId: workspaceId
    logs: [
      { category: 'StorageRead', enabled: true, retentionPolicy: { enabled: true, days: 90 } }
      { category: 'StorageWrite', enabled: true, retentionPolicy: { enabled: true, days: 90 } }
      { category: 'StorageDelete', enabled: true, retentionPolicy: { enabled: true, days: 90 } }
    ]
    metrics: [
      { category: 'Transaction', enabled: true, retentionPolicy: { enabled: true, days: 90 } }
      { category: 'Capacity', enabled: true, retentionPolicy: { enabled: true, days: 90 } }
    ]
  }
}

output storageAccountName string = storageAccount.name
output storageAccountId string = storageAccount.id
output blobEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

---

**Document Complete**: 3 of 3 Parts  
**Total Sections**: 10 (Overview, ADLS Gen2 Multi-Zone Architecture, Container Specifications, Lifecycle Management, Hybrid Storage Model, Operations, Monitoring, Security, Disaster Recovery, Cost Optimization)  
**Last Updated**: 2025-01-06
