# Healthcare EDI Platform - Routing & Messaging

**Document Version:** 1.0  
**Date:** October 6, 2025  
**Status:** Production  
**Owner:** EDI Platform Team

---

## Overview

### Purpose

The Routing & Messaging subsystem decouples ingestion from domain processing using Azure Service Bus as a durable message broker. After files are validated and persisted to raw storage, the router publishes granular messages per EDI transaction set (ST segment), enabling parallel processing by specialized trading partner adapters while maintaining loose coupling and high availability.

**Key Responsibilities:**

- Envelope parsing and transaction set extraction
- Routing message generation and publication
- Service Bus topic/subscription management
- Message filtering and delivery to trading partners
- Dead-letter queue management
- Correlation tracking across ingestion → routing → processing

### Key Principles

1. **Decoupled Architecture**: Ingestion completes before routing (no blocking)
2. **Message Granularity**: One message per ST transaction segment
3. **Durable Messaging**: Service Bus guarantees at-least-once delivery
4. **Filtered Subscriptions**: SQL filter expressions route to appropriate partners
5. **Correlation Chain**: ingestionId → routingId linkage for traceability
6. **Minimal Data Exposure**: No PHI/PII in message payload (references only)

---

## Architecture

### Component Diagram

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                   ROUTING & MESSAGING ARCHITECTURE                       │
└─────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────┐
  │  Processing Pipeline (Completed)                                      │
  │  • File validated and persisted to raw zone                          │
  │  • Metadata published to EDIIngestion_CL                             │
  └──────────────┬───────────────────────────────────────────────────────┘
                 │
                 │ Invoke Azure Function (HTTP trigger)
                 │ Parameters: ingestionId, rawBlobPath
                 ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Azure Function: func_router_dispatch                                 │
  │  ──────────────────────────────────────                               │
  │  1. Retrieve file metadata from ingestion record                     │
  │  2. Download blob (streaming, bounded peek: first 50 KB)             │
  │  3. Parse X12 envelope headers (ISA, GS, ST segments)                │
  │  4. Extract control numbers: ISA13, GS06, ST02                       │
  │  5. Enumerate ST segments (may be multiple per file)                 │
  │  6. For each ST:                                                      │
  │     a. Generate routingId (GUID)                                      │
  │     b. Build routing message JSON                                     │
  │     c. Publish to Service Bus Topic: edi-routing                     │
  │  7. Write routing audit to RoutingEvent_CL (Log Analytics)           │
  │  8. Handle publish failures with retry + DLQ                         │
  └──────────────┬───────────────────────────────────────────────────────┘
                 │
                 │ Publish messages
                 ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Azure Service Bus Namespace: sb-edi-{env}-{region}                  │
  │  ───────────────────────────────────────────                         │
  │                                                                        │
  │  ┌────────────────────────────────────────────────────────────┐     │
  │  │  Topic: edi-routing                                         │     │
  │  │  • Max size: 5 GB (Standard) / 80 GB (Premium)             │     │
  │  │  • TTL: 14 days (default)                                   │     │
  │  │  • Duplicate detection: 10 minutes                          │     │
  │  │  • Partitioning: Disabled (ordered delivery)                │     │
  │  │  • Message properties: transactionSet, partnerCode,         │     │
  │  │    direction, priority, correlationKey                      │     │
  │  └────────────────────────────────────────────────────────────┘     │
  │                                                                        │
  │  Subscriptions (filtered):                                            │
  │  ┌──────────────────────────────┬─────────────────────────────────┐ │
  │  │ sub-enrollment-partner       │ transactionSet = '834'          │ │
  │  │                              │ AND direction IN ('INBOUND',    │ │
  │  │                              │                   'INTERNAL')   │ │
  │  ├──────────────────────────────┼─────────────────────────────────┤ │
  │  │ sub-claims-partner           │ transactionSet LIKE '837%'      │ │
  │  │                              │ AND direction IN ('INBOUND',    │ │
  │  │                              │                   'INTERNAL')   │ │
  │  ├──────────────────────────────┼─────────────────────────────────┤ │
  │  │ sub-eligibility-partner      │ transactionSet IN ('270','271') │ │
  │  │                              │ AND direction IN ('INBOUND',    │ │
  │  │                              │                   'INTERNAL')   │ │
  │  ├──────────────────────────────┼─────────────────────────────────┤ │
  │  │ sub-remittance-partner       │ transactionSet = '835'          │ │
  │  │                              │ AND direction = 'INBOUND'       │ │
  │  ├──────────────────────────────┼─────────────────────────────────┤ │
  │  │ sub-correlation-analytics    │ ALL (no filter)                 │ │
  │  │                              │ For correlation tracking only   │ │
  │  └──────────────────────────────┴─────────────────────────────────┘ │
  │                                                                        │
  │  ┌────────────────────────────────────────────────────────────┐     │
  │  │  Topic: edi-deadletter                                      │     │
  │  │  • Poison messages from all subscriptions                   │     │
  │  │  • Manual review required before resubmission              │     │
  │  └────────────────────────────────────────────────────────────┘     │
  │                                                                        │
  │  ┌────────────────────────────────────────────────────────────┐     │
  │  │  Topic: edi-outbound-ready                                  │     │
  │  │  • Signals from trading partners: outcomes ready            │     │
  │  │  • Triggers outbound orchestrator assembly                  │     │
  │  └────────────────────────────────────────────────────────────┘     │
  └────────────────────────────────────────────────────────────────────────┘
                 │
                 │ Receive messages (each subscription)
                 ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Trading Partner Integration Adapters                                 │
  │  ─────────────────────────────────────                                │
  │  • Enrollment Management Partner: Processes 834 with event sourcing  │
  │  • Claims Processing Partner: Handles 837x with CQRS pattern         │
  │  • Eligibility Service Partner: 270/271 real-time processing         │
  │  • Remittance Processing Partner: 835 financial reconciliation       │
  │                                                                        │
  │  Each adapter:                                                         │
  │  1. Receives routing message from subscription                       │
  │  2. Idempotency check (routingId in state store)                     │
  │  3. Fetches raw file content (or relevant slice)                     │
  │  4. Parses domain-specific segments                                   │
  │  5. Executes partner business logic                                   │
  │  6. Persists outcome to partner storage                              │
  │  7. Emits acknowledgment signal to edi-outbound-ready               │
  └──────────────────────────────────────────────────────────────────────┘
```

### Message Flow Sequence

```text
1. Processing Pipeline: ADF invokes func_router_dispatch(ingestionId, rawBlobPath)
2. Router Function: Retrieves metadata from EDIIngestion_CL custom table
3. Router Function: Downloads blob (streaming, first 50 KB peek for envelope)
4. Router Function: Parses ISA segment → extract ISA13 control number
5. Router Function: Parses GS segment → extract GS06 functional group
6. Router Function: Enumerates ST segments → extract ST01 (transaction set), ST02 (control)
7. For each ST segment:
   a. Generate routingId = GUID
   b. Build JSON message body
   c. Set application properties: transactionSet, partnerCode, direction, priority
   d. Publish to Service Bus topic 'edi-routing'
   e. Await confirmation (at-least-once delivery guarantee)
8. Router Function: Write audit record to RoutingEvent_CL
9. Service Bus: Evaluates subscription filters for each message
10. Service Bus: Delivers message copy to matching subscriptions
11. Trading Partner Adapters: Receive messages via subscription listeners
12. Adapters: Process messages → persist outcomes → emit edi-outbound-ready signals
```

---

## Configuration

### Service Bus Infrastructure

**SKU Selection:**

| SKU | Max Topics | Max Subscriptions | Max Message Size | Throughput | Use Case |
|-----|------------|-------------------|------------------|------------|----------|
| **Standard** | 10,000 | 2,000/topic | 256 KB | 100 ops/sec | Dev, Test |
| **Premium** | Unlimited | Unlimited | 1 MB | 1,000+ ops/sec | Production |

**Recommendation:** Use Standard for dev/test; Premium for production with zone redundancy enabled.

**Bicep Configuration:**

```bicep
resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'sb-edi-${environment}-${region}'
  location: location
  sku: {
    name: 'Standard' // Use 'Premium' for production high-throughput
    tier: 'Standard'
    capacity: null
  }
  properties: {
    zoneRedundant: false // Set true for Premium SKU in production
    publicNetworkAccess: 'Enabled' // Restrict via VNet integration
    minimumTlsVersion: '1.2'
    disableLocalAuth: false // Enable managed identity auth
  }
}
```

### Topic Configuration

**Primary Topic: edi-routing**

```bicep
resource ediRoutingTopic 'Microsoft.ServiceBus/namespaces/topics@2022-10-01-preview' = {
  parent: serviceBusNamespace
  name: 'edi-routing'
  properties: {
    maxSizeInMegabytes: 5120 // 5 GB (Standard max)
    requiresDuplicateDetection: true
    duplicateDetectionHistoryTimeWindow: 'PT10M' // 10 minutes
    defaultMessageTimeToLive: 'P14D' // 14 days
    enableBatchedOperations: true
    supportOrdering: true // Maintains FIFO per partition key
    enablePartitioning: false // Disabled for strict ordering
    autoDeleteOnIdle: 'P10675199DT2H48M5.4775807S' // Never auto-delete
  }
}
```

**Key Properties:**

- **Duplicate Detection**: Prevents reprocessing if router retries within 10 minutes
- **Message TTL**: 14 days ensures messages available for delayed processing
- **Ordering**: Supports FIFO delivery when using session ID or partition key
- **Partitioning**: Disabled to maintain strict ordering per partner

### Subscription Examples

**Enrollment Management Partner:**

```bicep
resource subEnrollmentPartner 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2022-10-01-preview' = {
  parent: ediRoutingTopic
  name: 'sub-enrollment-partner'
  properties: {
    lockDuration: 'PT5M' // 5 minutes processing lock
    requiresSession: false
    defaultMessageTimeToLive: 'P14D'
    deadLetteringOnMessageExpiration: true
    maxDeliveryCount: 5 // Move to DLQ after 5 failures
    enableBatchedOperations: true
    forwardDeadLetteredMessagesTo: 'edi-deadletter' // Auto-forward to DLQ topic
  }
}

// SQL Filter for 834 transactions
resource subEnrollmentPartnerFilter 'Microsoft.ServiceBus/namespaces/topics/subscriptions/rules@2022-10-01-preview' = {
  parent: subEnrollmentPartner
  name: 'Filter834Transactions'
  properties: {
    filterType: 'SqlFilter'
    sqlFilter: {
      sqlExpression: "transactionSet = '834' AND direction IN ('INBOUND', 'INTERNAL')"
      requiresPreprocessing: true
    }
    action: {
      sqlExpression: "SET sys.Label = 'ENROLLMENT'"
    }
  }
}
```

**Eligibility Service Partner:**

```bicep
resource subEligibilityPartner 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2022-10-01-preview' = {
  parent: ediRoutingTopic
  name: 'sub-eligibility-partner'
  properties: {
    lockDuration: 'PT2M' // Shorter lock for real-time processing
    requiresSession: false
    defaultMessageTimeToLive: 'P14D'
    deadLetteringOnMessageExpiration: true
    maxDeliveryCount: 3 // Lower threshold for real-time
    enableBatchedOperations: true
    forwardDeadLetteredMessagesTo: 'edi-deadletter'
  }
}

// SQL Filter for 270/271 eligibility transactions
resource subEligibilityPartnerFilter 'Microsoft.ServiceBus/namespaces/topics/subscriptions/rules@2022-10-01-preview' = {
  parent: subEligibilityPartner
  name: 'FilterEligibilityTransactions'
  properties: {
    filterType: 'SqlFilter'
    sqlFilter: {
      sqlExpression: "transactionSet IN ('270', '271') AND direction IN ('INBOUND', 'INTERNAL')"
      requiresPreprocessing: true
    }
    action: {
      sqlExpression: "SET sys.Label = 'ELIGIBILITY'"
    }
  }
}
```

### Routing Message Schema

**JSON Structure:**

```json
{
  "routingId": "9f1c6d2e-3f3d-4b6c-9d4b-0e9e2a6c1a11",
  "ingestionId": "af82c4d7-5b3e-4c2d-9e8f-1a2b3c4d5e6f",
  "partnerCode": "PARTNERA",
  "transactionSet": "270",
  "direction": "INBOUND",
  "interchangeControl": "123456789",
  "functionalGroup": "987654321",
  "transactionControl": "0001",
  "stPosition": 1,
  "fileBlobPath": "raw/partner=PARTNERA/transaction=270/ingest_date=2025-10-06/PARTNERA_270_20251006120000_001.edi",
  "receivedUtc": "2025-10-06T12:00:10Z",
  "checksumSha256": "a3f5b8c2d1e9f0a7b6c4d3e2f1a0b9c8d7e6f5a4b3c2d1e0f9a8b7c6d5e4f3a2",
  "priority": "standard",
  "correlationKey": "PARTNERA-123456789-987654321"
}
```

**Application Properties (for filtering):**

```json
{
  "transactionSet": "270",
  "partnerCode": "PARTNERA",
  "direction": "INBOUND",
  "priority": "standard",
  "correlationKey": "PARTNERA-123456789-987654321",
  "MessageId": "9f1c6d2e-3f3d-4b6c-9d4b-0e9e2a6c1a11",
  "ContentType": "application/json",
  "Label": "EDI-ROUTING"
}
```

**Field Descriptions:**

| Field | Type | Description | Used For |
|-------|------|-------------|----------|
| routingId | GUID | Unique message identifier | Idempotency check |
| ingestionId | GUID | Links back to ingestion metadata | Traceability |
| partnerCode | String | Trading partner identifier | Filtering, correlation |
| transactionSet | String | X12 ST01 value (270, 834, 837P, etc.) | Subscription filtering |
| direction | String | INBOUND, OUTBOUND, INTERNAL | Subscription filtering |
| interchangeControl | String | ISA13 control number | Correlation, acknowledgments |
| functionalGroup | String | GS06 control number | Correlation, acknowledgments |
| transactionControl | String | ST02 control number | Transaction-level tracking |
| stPosition | Int | ST segment index in file (1-based) | File slice retrieval |
| fileBlobPath | String | Raw zone blob URI | Content retrieval |
| receivedUtc | Timestamp | Original ingestion time | SLA tracking |
| checksumSha256 | String | File integrity hash | Validation |
| priority | String | standard, high, urgent | Future prioritization |
| correlationKey | String | Composite key for grouping | Correlation queries |

---

## Operations

### Monitoring Metrics

**Key Metrics:**

| Metric | Description | Target | Alert Threshold |
|--------|-------------|--------|-----------------|
| **Routing Latency** | Time from ingestion complete to messages published | p95 < 2 sec | p95 > 5 sec |
| **Publish Success Rate** | Successful publishes / total attempts | > 99.9% | < 99.5% |
| **Message Count** | Messages published per hour | 5,000+ | N/A (capacity planning) |
| **DLQ Count** | Messages in dead-letter queue | 0 | > 10 |
| **Subscription Backlog** | Active messages per subscription | < 100 | > 1,000 |
| **Lock Duration Exceeded** | Messages exceeding processing lock | 0 | > 5 |

**KQL Queries:**

```kql
// Routing latency percentiles
RoutingEvent_CL
| where TimeGenerated > ago(24h)
| extend LatencyMs = datetime_diff('millisecond', publishedUtc_t, validationCompleteUtc_t)
| summarize 
    p50 = percentile(LatencyMs, 50),
    p95 = percentile(LatencyMs, 95),
    p99 = percentile(LatencyMs, 99),
    count()
    by bin(TimeGenerated, 1h)
| render timechart

// Publish failures by partner
RoutingEvent_CL
| where TimeGenerated > ago(24h)
| where publishStatus_s == "FAILED"
| summarize FailureCount = count() by partnerCode_s, transactionSet_s
| order by FailureCount desc
| render barchart

// Service Bus subscription metrics
ServiceBusMetrics
| where ResourceId contains "edi-routing"
| where TimeGenerated > ago(1h)
| summarize 
    ActiveMessages = max(ActiveMessages),
    DeadLetterMessages = max(DeadLetterMessages),
    ScheduledMessages = max(ScheduledMessages)
    by SubscriptionName, bin(TimeGenerated, 5m)
| render timechart

// Dead-letter queue analysis
ServiceBusDeadLetterMessages
| where TimeGenerated > ago(7d)
| extend DeadLetterReason = tostring(Properties.DeadLetterReason)
| summarize Count = count() by DeadLetterReason, SubscriptionName
| order by Count desc
| render piechart

// Find failed router invocations
FunctionAppLogs
| where FunctionName == "func_router_dispatch"
| where severityLevel >= 3 // Error or Critical
| where timestamp > ago(1h)
| project 
    timestamp,
    message,
    severityLevel,
    exception,
    ingestionId = tostring(customDimensions.ingestionId)
| order by timestamp desc

// Track subscription backlog trend
ServiceBusMetrics
| where ResourceId contains "edi-routing"
| where MetricName == "ActiveMessages"
| where TimeGenerated > ago(24h)
| summarize AvgActive = avg(Total), MaxActive = max(Total) by SubscriptionName, bin(TimeGenerated, 30m)
| render timechart

// Analyze DLQ messages by reason
ServiceBusDeadLetterMessages
| where TimeGenerated > ago(24h)
| extend 
    Reason = tostring(Properties.DeadLetterReason),
    Description = tostring(Properties.DeadLetterErrorDescription)
| summarize Count = count() by Reason, Description
| order by Count desc

// Find duplicate routingIds
RoutingEvent_CL
| where TimeGenerated > ago(24h)
| summarize ProcessCount = count() by routingId_g
| where ProcessCount > 1
| order by ProcessCount desc
```

### Troubleshooting

#### 1. Router Function Failures

**Symptom:** func_router_dispatch HTTP 500 errors

**Common Causes:**

- **Envelope parse failure**: Malformed X12 file (missing ISA/GS segments)
- **Service Bus connection timeout**: Network or auth issue
- **Blob download failure**: Storage account inaccessible or blob deleted
- **Memory exhaustion**: Large file exceeds Function memory limit

**Resolution:**

```powershell
# Check Function App configuration
az functionapp config show --name "func-edi-router-prod" --resource-group "rg-edi-prod"

# Verify Service Bus connection string in Key Vault
az keyvault secret show --vault-name "kv-edi-prod" --name "ServiceBusConnectionString"

# Check managed identity permissions
$FunctionAppId = (Get-AzFunctionApp -Name "func-edi-router-prod" -ResourceGroupName "rg-edi-prod").Identity.PrincipalId
Get-AzRoleAssignment -ObjectId $FunctionAppId

# Test Service Bus connectivity
Test-NetConnection -ComputerName "sb-edi-prod.servicebus.windows.net" -Port 5671
```

#### 2. Subscription Backlog Growing

**Symptom:** Active message count increasing, processing lag

**Common Causes:**

- **Processing function down**: Trading partner adapter not running
- **Processing too slow**: Business logic taking > lock duration
- **Poison messages**: Specific messages causing crashes
- **Scaling insufficient**: Too few worker instances

**Resolution:**

```powershell
# Scale out Function App
az functionapp config appsettings set --name "func-enrollment-partner" --resource-group "rg-edi-prod" --settings "FUNCTIONS_WORKER_PROCESS_COUNT=10"

# Check for poison messages in DLQ
az servicebus topic subscription show --namespace-name "sb-edi-prod" --topic-name "edi-routing" --name "sub-enrollment-partner" --query "countDetails.deadLetterMessageCount"

# Manually drain subscription (emergency)
az servicebus topic subscription update --namespace-name "sb-edi-prod" --topic-name "edi-routing" --name "sub-enrollment-partner" --status "ReceiveDisabled"
```

#### 3. Dead-Letter Queue Messages

**Symptom:** Messages appearing in edi-deadletter topic

**Common Reasons:**

| Reason | Description | Action |
|--------|-------------|--------|
| **MaxDeliveryCountExceeded** | Message failed 5 times | Investigate processing logic bug |
| **TTLExpiredException** | Message expired before processing | Increase subscription backlog capacity |
| **SessionLockLost** | Session-enabled subscription timeout | Increase lock duration |
| **MessageSizeExceeded** | Message > 256 KB (Standard) | Use Premium tier or reduce payload |

#### 4. Duplicate Message Processing

**Symptom:** Same routingId processed multiple times

**Common Causes:**

- **Duplicate detection disabled**: Topic/subscription configuration issue
- **Idempotency check missing**: Trading partner adapter not checking routingId
- **Router retry**: Function retried after transient failure

**Resolution:**

- Ensure duplicate detection enabled on topic (10-minute window)
- Implement idempotency check in all trading partner adapters
- Use routingId as unique key in partner state stores

---

## Security

### Access Control

**Service Bus RBAC Roles:**

| Identity | Role | Scope | Justification |
|----------|------|-------|---------------|
| func_router_dispatch | Azure Service Bus Data Sender | Topic: edi-routing | Publish routing messages |
| func_enrollment_partner | Azure Service Bus Data Receiver | Subscription: sub-enrollment-partner | Receive 834 messages |
| func_claims_partner | Azure Service Bus Data Receiver | Subscription: sub-claims-partner | Receive 837x messages |
| func_eligibility_partner | Azure Service Bus Data Receiver | Subscription: sub-eligibility-partner | Receive 270/271 messages |
| func_outbound_orchestrator | Azure Service Bus Data Sender | Topic: edi-outbound-ready | Signal outcomes ready |

**Bicep Configuration:**

```bicep
// Grant router function Send permissions
resource routerSendRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: ediRoutingTopic
  name: guid(ediRoutingTopic.id, routerFunction.id, '69a216fc-b8fb-44d8-bc22-1f3c2cd27a39')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 
      '69a216fc-b8fb-44d8-bc22-1f3c2cd27a39') // Azure Service Bus Data Sender
    principalId: routerFunction.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Grant enrollment partner Receive permissions
resource enrollmentReceiveRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: subEnrollmentPartner
  name: guid(subEnrollmentPartner.id, enrollmentFunction.id, '4f6d3b9b-027b-4f4c-9142-0e5a2a2247e0')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 
      '4f6d3b9b-027b-4f4c-9142-0e5a2a2247e0') // Azure Service Bus Data Receiver
    principalId: enrollmentFunction.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

### Data Protection

**Message Encryption:**

- **In Transit**: TLS 1.2+ for all Service Bus connections
- **At Rest**: Azure Service Bus automatic encryption with Microsoft-managed keys
- **Optional**: Customer-managed keys via Azure Key Vault

**PHI Minimization:**

- Routing messages contain NO patient identifiers (names, SSNs, member IDs)
- Only envelope control numbers and transaction set codes
- Full file content retrieved on-demand from secure blob storage

### Network Isolation

**Service Bus Private Endpoint:**

```bicep
resource serviceBusPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-04-01' = {
  name: 'pe-servicebus-${environment}'
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: 'servicebus-connection'
        properties: {
          privateLinkServiceId: serviceBusNamespace.id
          groupIds: [
            'namespace'
          ]
        }
      }
    ]
  }
}

// Disable public network access after private endpoint configured
resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  properties: {
    publicNetworkAccess: 'Disabled'
  }
}
```

---

## Performance

### Capacity Planning

**Current Throughput:**

- **Messages per hour**: 10,000+ (tested with average 1 ST per file)
- **Concurrent publishers**: 10 (router function instances)
- **Subscription processing**: 5,000+ messages/hour per partner
- **Average latency**: 500-2000 ms (p95 < 2 sec)

**Scaling Strategies:**

1. **Service Bus Premium**: Increase messaging units (1 → 2 → 4 → 8)
2. **Router Function**: Scale out App Service Plan (EP1 → EP2 → EP3)
3. **Batching**: Publish messages in batches of 100
4. **Partitioning**: Enable topic partitioning for higher throughput (trades ordering)

### Optimization Best Practices

**1. Batch Message Publishing:**

```csharp
// Publish in batches of 100 instead of individually
var messageBatch = await sender.CreateMessageBatchAsync();
foreach (var message in routingMessages)
{
    if (!messageBatch.TryAddMessage(message))
    {
        await sender.SendMessagesAsync(messageBatch);
        messageBatch = await sender.CreateMessageBatchAsync();
        messageBatch.TryAddMessage(message);
    }
}
if (messageBatch.Count > 0)
    await sender.SendMessagesAsync(messageBatch);
```

**2. Prefetch Messages:**

```csharp
// Enable prefetch for subscribers
var processor = serviceBusClient.CreateProcessor("edi-routing", "sub-enrollment-partner", new ServiceBusProcessorOptions
{
    MaxConcurrentCalls = 10,
    PrefetchCount = 20, // Fetch 20 messages ahead
    AutoCompleteMessages = false
});
```

**3. Connection Pooling:**

```csharp
// Reuse Service Bus client across function invocations (static)
private static ServiceBusClient _serviceBusClient;

public static async Task Run(...)
{
    if (_serviceBusClient == null)
    {
        _serviceBusClient = new ServiceBusClient(connectionString);
    }
    // Use _serviceBusClient for all operations
}
```

---

## Testing

### Integration Testing

**Test Scenarios:**

1. **Single ST File**: File with one transaction → one routing message
2. **Multiple ST File**: Batch file with 10 transactions → 10 routing messages
3. **Mixed Transaction Types**: File with 834 + 837P → routed to different subscriptions
4. **Subscription Filtering**: Verify 270 only goes to eligibility subscription
5. **Duplicate Detection**: Publish same message twice → deduplicated
6. **Dead-Letter Handling**: Force message failure → verify DLQ delivery
7. **Correlation Tracking**: Verify routingId → ingestionId linkage
8. **High Volume**: 1,000 files simultaneously → all routed successfully

**Test Data Generation:**

```powershell
# Generate test routing request
$routerRequest = @{
    ingestionId = "af82c4d7-5b3e-4c2d-9e8f-1a2b3c4d5e6f"
    rawBlobPath = "raw/partner=TESTPART/transaction=270/ingest_date=2025-10-06/TESTPART_270_20251006120000_001.edi"
} | ConvertTo-Json

# Invoke router function
Invoke-RestMethod -Method Post -Uri "https://func-edi-router-dev.azurewebsites.net/api/func_router_dispatch" -Body $routerRequest -ContentType "application/json" -Headers @{
    "x-functions-key" = $functionKey
}

# Verify message in subscription
az servicebus topic subscription show --namespace-name "sb-edi-dev" --topic-name "edi-routing" --name "sub-eligibility-partner" --query "countDetails.activeMessageCount"
```

### Load Testing

**Test Configuration:**

- **Tool**: Azure Load Testing service
- **Scenario**: 10,000 routing messages published in 60 minutes
- **Message distribution**: 40% 270, 30% 834, 20% 837P, 10% 835
- **Success criteria**: 
  - Publish success rate > 99.9%
  - p95 latency < 2 seconds
  - No DLQ messages
  - All subscriptions process within 5 minutes

---

## Related Documentation

- [02-processing-pipeline.md](./02-processing-pipeline.md) - Upstream validation and raw persistence
- [04-mapper-transformation.md](./04-mapper-transformation.md) - Downstream transformation (next)
- [10-trading-partner-config.md](./10-trading-partner-config.md) - Partner configuration

---

**Document Status:** Complete  
**Last Validation:** October 6, 2025  
**Next Review:** January 6, 2026
