# Outbound Delivery & Acknowledgment Generation

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Control Number Store](#control-number-store)
- [Envelope Generation](#envelope-generation)
- [Acknowledgment Types](#acknowledgment-types)
- [Outbound Assembly Orchestration](#outbound-assembly-orchestration)
- [Response File Management](#response-file-management)
- [Error Handling](#error-handling)
- [Security & Compliance](#security--compliance)
- [Performance & Scalability](#performance--scalability)
- [Monitoring & Observability](#monitoring--observability)
- [Troubleshooting](#troubleshooting)

---

## Overview

### Purpose
The Outbound Delivery subsystem generates and delivers EDI acknowledgments and business responses back to trading partners. This includes technical acknowledgments (TA1, 999), business responses (271 eligibility, 277 claim status, 835 remittance), and ensures proper EDI envelope structure with monotonically increasing control numbers.

### Key Responsibilities
- **Control Number Management**: Maintains ISA13 (interchange), GS06 (functional group), and ST02 (transaction set) control numbers per partner
- **Envelope Generation**: Constructs valid X12 EDI envelopes (ISA/GS/ST...SE/GE/IEA) with proper segment counts
- **Acknowledgment Assembly**: Generates TA1 (interchange), 999 (functional), and business responses based on processing outcomes
- **Response Delivery**: Persists outbound files to storage and signals delivery systems
- **Audit Trail**: Maintains comprehensive audit logs for all control numbers issued

### Business Context
Healthcare trading partners require timely acknowledgments to confirm receipt and processing status:
- **TA1 (Interchange Acknowledgment)**: < 5 min SLA - structural validation
- **999 (Functional Acknowledgment)**: < 15 min SLA - syntax validation
- **271 (Eligibility Response)**: < 5 min SLA - real-time eligibility
- **277CA (Claim Acknowledgment)**: < 4 hrs SLA - claim receipt confirmation
- **835 (Remittance Advice)**: Payer SLA - payment processing (weekly cycle)

### Integration Points

**Upstream Dependencies**:
- Service Bus `edi-routing` topic: Receives routing messages with inbound control numbers
- Partner Adapters: Emit processing outcome signals (success, error, business response data)
- Processing Pipeline: Generates staging data for response assembly

**Downstream Consumers**:
- Service Bus `edi-outbound-ready` topic: Signals outbound files ready for delivery
- SFTP Connector: Delivers files to partner SFTP endpoints
- Partner Portal API: Provides self-service file download
- Storage Accounts: Archives outbound files for compliance

### Architecture Principles

1. **Monotonic Control Numbers**: ISA/GS/ST numbers must always increment, never reuse (except documented gaps)
2. **Optimistic Concurrency**: ROWVERSION-based locking prevents duplicate control numbers under concurrent load
3. **Idempotent Assembly**: Same input outcomes produce same output file (deterministic)
4. **Atomic Persistence**: File write + control number audit + Service Bus publish in logical transaction
5. **Gap Detection**: Continuous monitoring detects skipped control numbers for investigation

---

## Architecture

### Component Diagram

The outbound delivery layer consists of:

- **Outbound Orchestrator** (Azure Durable Function): Stateful workflow coordinating response assembly
- **Control Number Store** (Azure SQL): ACID-compliant counter management with ROWVERSION concurrency
- **Envelope Template Library**: C# segment builder for ISA/GS/ST/SE/GE/IEA segments
- **Outbound Storage**: Azure Blob Storage for persisting EDI files
- **Service Bus**: Event notification for delivery systems
- **Application Insights**: Latency tracking, retry monitoring, gap detection alerts

### Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Orchestrator | Azure Durable Functions (.NET 9) | Stateful outbound assembly workflow |
| Control Number Store | Azure SQL Database (S1 tier) | ACID-compliant counter management with ROWVERSION |
| Concurrency Model | Optimistic Locking (ROWVERSION) | Prevents duplicate control numbers under concurrent load |
| Message Broker | Service Bus Standard | Signals outbound files ready for delivery |
| File Storage | Azure Blob Storage (Hot tier) | Persists outbound EDI files |
| Template Engine | Custom C# Segment Builder | Generates EDI envelope segments (ISA/GS/ST/SE/GE/IEA) |
| Monitoring | Application Insights + Azure Monitor | Tracks latency, retry rate, gap detection |

### Data Flow - Outbound Response Assembly

```text
┌────────────────────────────────────────────────────────────────────┐
│                  OUTBOUND RESPONSE ASSEMBLY FLOW                    │
└────────────────────────────────────────────────────────────────────┘

1. Partner Adapter → Processing Complete Signal
2. Outbound Orchestrator → Query Staging Tables (5-min batch window)
3. Orchestrator → Group by Partner + Transaction + Response Type

For each outbound file:
  4. Orchestrator → Control Number Store: Request ISA13
     - Store reads CurrentValue + RowVersion
     - Store updates WHERE RowVersion = @CurrentRowVersion
     - If successful: Insert audit record, return control number
     - If collision: Retry with exponential backoff (max 5 attempts)
  
  5. Orchestrator → Control Number Store: Request GS06
  6. Orchestrator → Control Number Store: Request ST02
  
  7. Orchestrator → Build Segment List (ISA/GS/ST...SE/GE/IEA)
  8. Orchestrator → Validate Segment Counts (ST03, SE01, GS01, GE01, IEA01)
  9. Orchestrator → Persist file to Outbound Storage
  10. Orchestrator → Update Control Number Audit (PERSISTED status)
  11. Orchestrator → Publish to Service Bus (edi-outbound-ready)
```

### Deployment Architecture

```text
Resource Group: rg-edi-outbound-prod
├── Azure SQL Database: sql-edi-controlnumbers-prod
│   ├── Database: edi-controlnumbers
│   ├── Tier: Standard S1 (20 DTUs)
│   ├── Backup: Point-in-time restore (35 days)
│   └── Security: Private Endpoint, TDE enabled
├── Function App: func-edi-outbound-prod
│   ├── Runtime: .NET 9 Isolated
│   ├── Plan: Premium EP1 (3.5 GB RAM, always-on)
│   ├── Identity: Managed Identity (db_datawriter + db_datareader)
│   └── Connection: Key Vault reference for SQL connection string
├── Storage Account: stedioutboundprod
│   ├── Container: outbound (Hot tier, versioning enabled)
│   ├── Container: outbound-staging (Hot tier, 7-day lifecycle)
│   └── Lifecycle: Move to Cool after 90 days, Archive after 1 year
├── Service Bus Namespace: sb-edi-routing-prod
│   ├── Topic: edi-outbound-ready
│   └── Subscriptions: sub-sftp-delivery, sub-portal-api
└── Key Vault: kv-edi-prod
    └── Secret: sql-controlnumbers-connection-string
```

---

## Control Number Store

### Purpose & Design Decision

The Control Number Store maintains monotonically increasing ISA13 (interchange), GS06 (functional group), and ST02 (transaction set) control numbers for all outbound EDI responses. These numbers are critical for:

- **Partner Reconciliation**: Trading partners use control numbers to match acknowledgments to original transactions
- **Duplicate Detection**: Prevents reprocessing of already-received files
- **Audit Compliance**: Provides complete chain of custody for all EDI exchanges
- **Gap Detection**: Identifies missing or skipped control numbers for investigation

**Technology Decision**: Azure SQL Database was selected over Azure Table Storage or Durable Function Entities for:

1. **ACID Guarantees**: Transactions ensure atomic counter increment + audit insert
2. **Optimistic Concurrency**: ROWVERSION column provides built-in collision detection
3. **Query Flexibility**: SQL queries for gap detection, partner reporting, rollover alerts
4. **Backup/Recovery**: Point-in-time restore for disaster recovery scenarios
5. **Azure Integration**: Native Azure Monitor integration, Private Endpoints, Managed Identity

### Database Schema

**Repository**: `edi-database-controlnumbers` (DACPAC deployed to Azure SQL)

#### ControlNumberCounters Table

```sql
CREATE TABLE [dbo].[ControlNumberCounters]
(
    [CounterId]           INT IDENTITY(1,1)            NOT NULL,
    [PartnerCode]         NVARCHAR(15)                 NOT NULL,
    [TransactionType]     NVARCHAR(10)                 NOT NULL,    -- '270', '271', '837', '835', '999'
    [CounterType]         NVARCHAR(20)                 NOT NULL,    -- 'ISA', 'GS', 'ST'
    [CurrentValue]        BIGINT                       NOT NULL DEFAULT (1),
    [MaxValue]            BIGINT                       NOT NULL DEFAULT (999999999), -- 9-digit ISA13
    [LastIncrementUtc]    DATETIME2(3)                 NOT NULL DEFAULT (SYSUTCDATETIME()),
    [LastFileGenerated]   NVARCHAR(255)                NULL,        -- Last outbound file name
    [CreatedUtc]          DATETIME2(3)                 NOT NULL DEFAULT (SYSUTCDATETIME()),
    [ModifiedUtc]         DATETIME2(3)                 NOT NULL DEFAULT (SYSUTCDATETIME()),
    [RowVersion]          ROWVERSION                   NOT NULL,    -- Optimistic concurrency
    CONSTRAINT [PK_ControlNumberCounters] PRIMARY KEY CLUSTERED ([CounterId] ASC)
);
GO

-- Enforce unique constraint on partner + transaction + counter type
CREATE UNIQUE INDEX [UQ_ControlNumberCounters_Key]
    ON [dbo].[ControlNumberCounters] ([PartnerCode], [TransactionType], [CounterType]);
GO

-- Performance index for counter lookups
CREATE INDEX [IX_ControlNumberCounters_PartnerType]
    ON [dbo].[ControlNumberCounters] ([PartnerCode], [TransactionType])
    INCLUDE ([CurrentValue], [LastIncrementUtc]);
GO
```

**Key Fields**:
- `PartnerCode`: Trading partner identifier (e.g., 'PARTNERA', 'TEST001')
- `TransactionType`: X12 transaction set code (e.g., '270', '837', '999')
- `CounterType`: Envelope level ('ISA', 'GS', 'ST')
- `CurrentValue`: Current control number (1 to 999,999,999)
- `RowVersion`: Automatic timestamp for optimistic concurrency (changes on every UPDATE)

#### ControlNumberAudit Table

```sql
CREATE TABLE [dbo].[ControlNumberAudit]
(
    [AuditId]             BIGINT IDENTITY(1,1)         NOT NULL,
    [CounterId]           INT                          NOT NULL,
    [ControlNumberIssued] BIGINT                       NOT NULL,    -- Actual control number issued
    [OutboundFileId]      UNIQUEIDENTIFIER             NOT NULL,    -- Correlation to outbound file
    [IssuedUtc]           DATETIME2(3)                 NOT NULL DEFAULT (SYSUTCDATETIME()),
    [RetryCount]          INT                          NOT NULL DEFAULT (0),
    [Status]              NVARCHAR(20)                 NOT NULL DEFAULT ('ISSUED'),
    [Notes]               NVARCHAR(500)                NULL,        -- Reset reason, gap explanation
    CONSTRAINT [PK_ControlNumberAudit] PRIMARY KEY CLUSTERED ([AuditId] ASC),
    CONSTRAINT [FK_ControlNumberAudit_CounterId] FOREIGN KEY ([CounterId]) 
        REFERENCES [dbo].[ControlNumberCounters]([CounterId])
);
GO

CREATE INDEX [IX_ControlNumberAudit_CounterId]
    ON [dbo].[ControlNumberAudit] ([CounterId], [IssuedUtc]);
GO

CREATE INDEX [IX_ControlNumberAudit_OutboundFileId]
    ON [dbo].[ControlNumberAudit] ([OutboundFileId]);
GO
```

**Status Values**:
- `ISSUED`: Control number acquired, outbound file assembly in progress
- `PERSISTED`: Outbound file successfully written to storage
- `FAILED`: Outbound file assembly failed (gap documented)
- `RESET`: Counter manually reset (with reason in Notes)
- `REISSUED`: Control number reused after failed assembly (gap closed)

### Optimistic Concurrency Model

The control number store uses ROWVERSION-based optimistic concurrency to prevent duplicate control numbers under concurrent load.

#### Stored Procedure: usp_GetNextControlNumber

```sql
CREATE PROCEDURE [dbo].[usp_GetNextControlNumber]
    @PartnerCode        NVARCHAR(15),
    @TransactionType    NVARCHAR(10),
    @CounterType        NVARCHAR(20),
    @OutboundFileId     UNIQUEIDENTIFIER,
    @NextControlNumber  BIGINT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    DECLARE @MaxRetries INT = 5;
    DECLARE @RetryCount INT = 0;
    DECLARE @CurrentRowVersion BINARY(8);
    DECLARE @CurrentValue BIGINT;
    DECLARE @MaxValue BIGINT;
    DECLARE @CounterId INT;

    WHILE @RetryCount < @MaxRetries
    BEGIN
        BEGIN TRY
            -- Read current value and row version with UPDLOCK
            SELECT
                @CounterId = CounterId,
                @CurrentValue = CurrentValue,
                @MaxValue = MaxValue,
                @CurrentRowVersion = RowVersion
            FROM dbo.ControlNumberCounters WITH (UPDLOCK, READPAST)
            WHERE PartnerCode = @PartnerCode
              AND TransactionType = @TransactionType
              AND CounterType = @CounterType;

            -- Initialize counter if not exists
            IF @CounterId IS NULL
            BEGIN
                INSERT INTO dbo.ControlNumberCounters 
                    (PartnerCode, TransactionType, CounterType)
                VALUES (@PartnerCode, @TransactionType, @CounterType);

                SELECT
                    @CounterId = CounterId,
                    @CurrentValue = CurrentValue,
                    @MaxValue = MaxValue,
                    @CurrentRowVersion = RowVersion
                FROM dbo.ControlNumberCounters
                WHERE PartnerCode = @PartnerCode
                  AND TransactionType = @TransactionType
                  AND CounterType = @CounterType;
            END;

            -- Calculate next value
            SET @NextControlNumber = @CurrentValue + 1;

            -- Check for rollover
            IF @NextControlNumber > @MaxValue
            BEGIN
                THROW 50001, 'Control number max value exceeded. Rollover required.', 1;
            END;

            -- Attempt update with concurrency check
            UPDATE dbo.ControlNumberCounters
            SET
                CurrentValue = @NextControlNumber,
                LastIncrementUtc = SYSUTCDATETIME(),
                ModifiedUtc = SYSUTCDATETIME()
            WHERE CounterId = @CounterId
              AND RowVersion = @CurrentRowVersion;  -- Optimistic concurrency guard

            -- Check if update succeeded
            IF @@ROWCOUNT = 1
            BEGIN
                -- Success - insert audit record
                INSERT INTO dbo.ControlNumberAudit
                    (CounterId, ControlNumberIssued, OutboundFileId, RetryCount, Status)
                VALUES
                    (@CounterId, @NextControlNumber, @OutboundFileId, @RetryCount, 'ISSUED');

                -- Return success
                RETURN 0;
            END
            ELSE
            BEGIN
                -- Concurrency collision - retry with exponential backoff
                SET @RetryCount = @RetryCount + 1;
                WAITFOR DELAY '00:00:00.050';  -- 50ms base delay
            END;

        END TRY
        BEGIN CATCH
            -- Log error and retry
            SET @RetryCount = @RetryCount + 1;

            IF @RetryCount >= @MaxRetries
            BEGIN
                THROW;
            END;

            WAITFOR DELAY '00:00:00.100';  -- 100ms on error
        END CATCH;
    END;

    -- Max retries exceeded
    THROW 50002, 'Control number acquisition failed after maximum retries', 1;
END;
GO
```

**Concurrency Workflow**:
1. Read `CurrentValue` + `RowVersion` with `UPDLOCK` (prevents dirty reads)
2. Calculate `@NextControlNumber = @CurrentValue + 1`
3. Attempt UPDATE WHERE `RowVersion = @CurrentRowVersion`
4. If `@@ROWCOUNT = 1`: Success, insert audit record, return control number
5. If `@@ROWCOUNT = 0`: Collision detected (another process updated), retry with exponential backoff
6. Max 5 retries with 50ms base delay (50ms, 100ms, 150ms, 200ms, 250ms)

**Performance Characteristics**:
- **Read Lock Duration**: < 10ms (UPDLOCK held only during SELECT)
- **Update Duration**: < 5ms (single row update with index seek)
- **Retry Probability**: < 5% under normal load (100 TPS)
- **p95 Latency**: < 50ms including retries

### Gap Detection

#### View: ControlNumberGaps

```sql
CREATE VIEW [dbo].[ControlNumberGaps]
AS
    WITH NumberedAudit AS (
        SELECT
            a.CounterId,
            c.PartnerCode,
            c.TransactionType,
            c.CounterType,
            a.ControlNumberIssued,
            LAG(a.ControlNumberIssued) OVER (
                PARTITION BY a.CounterId 
                ORDER BY a.ControlNumberIssued
            ) AS PreviousNumber
        FROM dbo.ControlNumberAudit a
        INNER JOIN dbo.ControlNumberCounters c ON a.CounterId = c.CounterId
        WHERE a.Status IN ('ISSUED', 'PERSISTED')
    )
    SELECT
        PartnerCode,
        TransactionType,
        CounterType,
        PreviousNumber AS GapStart,
        ControlNumberIssued AS GapEnd,
        (ControlNumberIssued - PreviousNumber - 1) AS GapSize
    FROM NumberedAudit
    WHERE PreviousNumber IS NOT NULL
      AND ControlNumberIssued - PreviousNumber > 1;
GO
```

**Gap Detection Logic**:
- Uses `LAG()` window function to compare consecutive control numbers
- Identifies gaps where `GapEnd - GapStart > 1`
- Excludes `FAILED` status records (expected gaps)

#### Stored Procedure: usp_DetectControlNumberGaps

```sql
CREATE PROCEDURE [dbo].[usp_DetectControlNumberGaps]
    @PartnerCode        NVARCHAR(15) = NULL,
    @TransactionType    NVARCHAR(10) = NULL,
    @DaysToCheck        INT = 30
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @StartDate DATETIME2 = DATEADD(DAY, -@DaysToCheck, SYSUTCDATETIME());

    SELECT
        PartnerCode,
        TransactionType,
        CounterType,
        GapStart,
        GapEnd,
        GapSize,
        CASE
            WHEN GapSize = 1 THEN 'MINOR'
            WHEN GapSize <= 5 THEN 'MODERATE'
            ELSE 'CRITICAL'
        END AS Severity
    FROM dbo.ControlNumberGaps
    WHERE (@PartnerCode IS NULL OR PartnerCode = @PartnerCode)
      AND (@TransactionType IS NULL OR TransactionType = @TransactionType)
    ORDER BY GapSize DESC, PartnerCode, TransactionType, CounterType;
END;
GO
```

**Severity Classification**:
- `MINOR`: Gap of 1 (single missed control number, acceptable for failed assembly)
- `MODERATE`: Gap of 2-5 (investigate, may indicate multiple failures)
- `CRITICAL`: Gap > 5 (requires immediate investigation, potential data loss)

### Rollover & Reset Handling

#### Stored Procedure: usp_ResetControlNumber

```sql
CREATE PROCEDURE [dbo].[usp_ResetControlNumber]
    @PartnerCode        NVARCHAR(15),
    @TransactionType    NVARCHAR(10),
    @CounterType        NVARCHAR(20),
    @NewValue           BIGINT = 1,
    @Reason             NVARCHAR(500)
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    DECLARE @CounterId INT;

    BEGIN TRANSACTION;

    -- Get counter ID
    SELECT @CounterId = CounterId
    FROM dbo.ControlNumberCounters
    WHERE PartnerCode = @PartnerCode
      AND TransactionType = @TransactionType
      AND CounterType = @CounterType;

    IF @CounterId IS NULL
    BEGIN
        ROLLBACK TRANSACTION;
        THROW 50003, 'Control number counter not found', 1;
    END;

    -- Log reset in audit table
    INSERT INTO dbo.ControlNumberAudit
        (CounterId, ControlNumberIssued, OutboundFileId, RetryCount, Status, Notes)
    VALUES
        (@CounterId, @NewValue, NEWID(), 0, 'RESET', @Reason);

    -- Reset counter
    UPDATE dbo.ControlNumberCounters
    SET
        CurrentValue = @NewValue,
        LastIncrementUtc = SYSUTCDATETIME(),
        ModifiedUtc = SYSUTCDATETIME()
    WHERE CounterId = @CounterId;

    COMMIT TRANSACTION;

    RETURN 0;
END;
GO
```

**Rollover Workflow**:
1. **Alert at 90% Threshold**: Azure Monitor alert fires when `CurrentValue > 0.9 * MaxValue` (899,999,999)
2. **Coordination Window**: Notify trading partner of upcoming reset (7-day notice)
3. **Execute Reset**: Call `usp_ResetControlNumber` with reason (e.g., "Planned rollover to 1 after reaching 900M")
4. **Document Gap**: Audit trail shows RESET status with explanation
5. **Partner Notification**: Send notification confirming reset and new starting number

### Seed Data & Initial Setup

```sql
-- Partner A - Full transaction set (270/271 eligibility, 837 claims, 835 remittance)
MERGE INTO dbo.ControlNumberCounters AS target
USING (
    SELECT 'PARTNERA' AS PartnerCode, '270' AS TransactionType, 'ISA' AS CounterType UNION ALL
    SELECT 'PARTNERA', '270', 'GS' UNION ALL
    SELECT 'PARTNERA', '270', 'ST' UNION ALL
    SELECT 'PARTNERA', '271', 'ISA' UNION ALL
    SELECT 'PARTNERA', '271', 'GS' UNION ALL
    SELECT 'PARTNERA', '271', 'ST' UNION ALL
    SELECT 'PARTNERA', '837', 'ISA' UNION ALL
    SELECT 'PARTNERA', '837', 'GS' UNION ALL
    SELECT 'PARTNERA', '837', 'ST' UNION ALL
    SELECT 'PARTNERA', '835', 'ISA' UNION ALL
    SELECT 'PARTNERA', '835', 'GS' UNION ALL
    SELECT 'PARTNERA', '835', 'ST'
) AS source (PartnerCode, TransactionType, CounterType)
ON target.PartnerCode = source.PartnerCode
   AND target.TransactionType = source.TransactionType
   AND target.CounterType = source.CounterType
WHEN NOT MATCHED THEN
    INSERT (PartnerCode, TransactionType, CounterType, CurrentValue)
    VALUES (source.PartnerCode, source.TransactionType, source.CounterType, 1);

-- Internal Claims - Outbound acknowledgments only (999, 277)
MERGE INTO dbo.ControlNumberCounters AS target
USING (
    SELECT 'INTERNAL-CLAIMS' AS PartnerCode, '277' AS TransactionType, 'ISA' AS CounterType UNION ALL
    SELECT 'INTERNAL-CLAIMS', '277', 'GS' UNION ALL
    SELECT 'INTERNAL-CLAIMS', '277', 'ST' UNION ALL
    SELECT 'INTERNAL-CLAIMS', '999', 'ISA' UNION ALL
    SELECT 'INTERNAL-CLAIMS', '999', 'GS' UNION ALL
    SELECT 'INTERNAL-CLAIMS', '999', 'ST'
) AS source (PartnerCode, TransactionType, CounterType)
ON target.PartnerCode = source.PartnerCode
   AND target.TransactionType = source.TransactionType
   AND target.CounterType = source.CounterType
WHEN NOT MATCHED THEN
    INSERT (PartnerCode, TransactionType, CounterType, CurrentValue)
    VALUES (source.PartnerCode, source.TransactionType, source.CounterType, 1);
```

**Onboarding New Partner**:

1. Partner metadata created in `edi-partner-configs` repository
2. Seed script adds 3 counters per transaction type (ISA, GS, ST)
3. Initial value = 1 for all counters
4. MaxValue = 999,999,999 (9-digit ISA13 standard)

---

## Envelope Generation

### X12 EDI Envelope Structure

EDI files use a hierarchical envelope structure to organize transactions:

```text
ISA (Interchange)                           ← ISA13 control number
├── GS (Functional Group)                   ← GS06 control number
│   ├── ST (Transaction Set)                ← ST02 control number
│   │   └── [Transaction segments]
│   ├── SE (Transaction Set Trailer)        ← SE01 = segment count
│   ├── ST (Transaction Set)
│   │   └── [Transaction segments]
│   └── SE (Transaction Set Trailer)
├── GE (Functional Group Trailer)           ← GE01 = ST count, GE02 = GS06
└── IEA (Interchange Trailer)               ← IEA01 = GS count, IEA02 = ISA13
```

**Segment Count Validation**:
- `SE01`: Number of segments in transaction set (including ST and SE)
- `GE01`: Number of ST segments in functional group
- `IEA01`: Number of GS segments in interchange

### Envelope Template Example - 999 Acknowledgment

```text
ISA*00*          *00*          *ZZ*SENDER-ID      *ZZ*RECEIVER-ID    *250106*1530*^*00501*000123456*0*P*:~
GS*FA*SENDER-APP*RECEIVER-APP*20250106*1530*123*X*005010X231A1~
ST*999*0001*005010X231A1~
AK1*HC*123456*005010X834A1~
AK9*A*1*1*1~
SE*4*0001~
GE*1*123~
IEA*1*000123456~
```

**Control Number Mapping**:
- `ISA13 = 000123456` (Interchange Control Number from Control Number Store ISA counter)
- `ISA16 = :` (Component element separator)
- `GS06 = 123` (Group Control Number from Control Number Store GS counter)
- `ST02 = 0001` (Transaction Set Control Number from Control Number Store ST counter)
- `SE02 = 0001` (Must match ST02)
- `GE02 = 123` (Must match GS06)
- `IEA02 = 000123456` (Must match ISA13)

### C# Envelope Builder Example

```csharp
public class EdiEnvelopeBuilder
{
    private readonly IControlNumberService _controlNumberService;
    private readonly ILogger<EdiEnvelopeBuilder> _logger;

    public async Task<string> BuildOutboundFile(
        string partnerCode,
        string transactionType,
        List<string> transactionSegments,
        Guid outboundFileId)
    {
        // Acquire control numbers
        var isaNumber = await _controlNumberService.GetNextControlNumber(
            partnerCode, transactionType, "ISA", outboundFileId);
        var gsNumber = await _controlNumberService.GetNextControlNumber(
            partnerCode, transactionType, "GS", outboundFileId);
        var stNumber = await _controlNumberService.GetNextControlNumber(
            partnerCode, transactionType, "ST", outboundFileId);

        _logger.LogInformation(
            "Acquired control numbers: ISA13={Isa13}, GS06={Gs06}, ST02={St02}",
            isaNumber, gsNumber, stNumber);

        // Build envelope
        var segments = new List<string>();

        // ISA segment
        segments.Add(BuildIsaSegment(partnerCode, isaNumber));

        // GS segment
        segments.Add(BuildGsSegment(partnerCode, transactionType, gsNumber));

        // ST segment
        segments.Add($"ST*{transactionType}*{stNumber:D4}*005010X231A1~");

        // Add transaction segments
        segments.AddRange(transactionSegments);

        // SE segment (segment count including ST and SE)
        var segmentCount = transactionSegments.Count + 2;
        segments.Add($"SE*{segmentCount}*{stNumber:D4}~");

        // GE segment (ST count = 1 for single transaction)
        segments.Add($"GE*1*{gsNumber}~");

        // IEA segment (GS count = 1 for single functional group)
        segments.Add($"IEA*1*{isaNumber:D9}~");

        // Validate segment counts
        ValidateEnvelope(segments, isaNumber, gsNumber, stNumber);

        return string.Join("", segments);
    }

    private string BuildIsaSegment(string partnerCode, long isaNumber)
    {
        var now = DateTime.UtcNow;
        var partner = GetPartnerConfig(partnerCode);

        return $"ISA" +
            $"*00*          " +              // Authorization info (no security)
            $"*00*          " +              // Security info (no security)
            $"*ZZ*{partner.SenderId.PadRight(15)}" +
            $"*ZZ*{partner.ReceiverId.PadRight(15)}" +
            $"*{now:yyMMdd}" +               // Date (YYMMDD)
            $"*{now:HHmm}" +                 // Time (HHMM)
            $"*^" +                          // Repetition separator
            $"*00501" +                      // Version (5010)
            $"*{isaNumber:D9}" +             // Interchange Control Number (9 digits)
            $"*0" +                          // Acknowledgment requested (0 = no)
            $"*P" +                          // Usage (P = production)
            $"*:~";                          // Component separator + segment terminator
    }

    private string BuildGsSegment(string partnerCode, string transactionType, long gsNumber)
    {
        var now = DateTime.UtcNow;
        var partner = GetPartnerConfig(partnerCode);
        var functionalCode = GetFunctionalIdentifierCode(transactionType);

        return $"GS" +
            $"*{functionalCode}" +           // Functional identifier (FA, HC, etc.)
            $"*{partner.ApplicationSenderId}" +
            $"*{partner.ApplicationReceiverId}" +
            $"*{now:yyyyMMdd}" +             // Date (YYYYMMDD)
            $"*{now:HHmm}" +                 // Time (HHMM)
            $"*{gsNumber}" +                 // Group Control Number
            $"*X" +                          // Responsible agency (X = X12)
            $"*{GetVersionCode(transactionType)}~";
    }

    private void ValidateEnvelope(
        List<string> segments,
        long isaNumber,
        long gsNumber,
        long stNumber)
    {
        // Extract and validate control numbers
        var isaSegment = segments[0];
        var isaMatch = Regex.Match(isaSegment, @"\*(\d{9})\*");
        if (!isaMatch.Success || long.Parse(isaMatch.Groups[1].Value) != isaNumber)
        {
            throw new InvalidOperationException(
                $"ISA control number mismatch: expected {isaNumber}");
        }

        var geSegment = segments[^2];  // Second to last
        var geMatch = Regex.Match(geSegment, @"GE\*\d+\*(\d+)~");
        if (!geMatch.Success || long.Parse(geMatch.Groups[1].Value) != gsNumber)
        {
            throw new InvalidOperationException(
                $"GE control number mismatch: expected {gsNumber}");
        }

        var ieaSegment = segments[^1];  // Last
        var ieaMatch = Regex.Match(ieaSegment, @"IEA\*\d+\*(\d{9})~");
        if (!ieaMatch.Success || long.Parse(ieaMatch.Groups[1].Value) != isaNumber)
        {
            throw new InvalidOperationException(
                $"IEA control number mismatch: expected {isaNumber}");
        }

        _logger.LogInformation(
            "Envelope validation passed: {SegmentCount} segments, ISA13={Isa13}",
            segments.Count, isaNumber);
    }
}
```

### Functional Identifier Codes

| Transaction Type | GS01 Code | Description |
|------------------|-----------|-------------|
| 270 | HS | Health Care Eligibility/Benefit Inquiry |
| 271 | HB | Health Care Eligibility/Benefit Response |
| 834 | BE | Benefit Enrollment and Maintenance |
| 837 (P/I/D) | HC | Health Care Claim |
| 835 | HP | Health Care Claim Payment/Advice |
| 999 | FA | Functional Acknowledgment |
| TA1 | N/A | Interchange Acknowledgment (no GS envelope) |

---

## Acknowledgment Types

The platform generates five types of EDI acknowledgments and business responses:

### TA1 - Interchange Acknowledgment (< 5 min SLA)

**Purpose**: Validates structural integrity of the ISA/IEA envelope.

**Generated When**: Immediately upon receipt of inbound EDI file during ingestion validation.

**TA1 Segments**:
```text
ISA*00*...*000123456*0*P*:~
TA1*987654321*250106*1530*A*000~
IEA*1*000123456~
```

**TA1 Fields**:
- **TA101**: Interchange Control Number being acknowledged (matches inbound ISA13)
- **TA102**: Interchange Date (YYMMDD from ISA09)
- **TA103**: Interchange Time (HHMM from ISA10)
- **TA104**: Acknowledgment Code (A/E/R)
- **TA105**: Error Code (000 = No error, 001-999 = specific errors)

**TA104 Acknowledgment Codes**:
- `A`: Accepted - Interchange accepted, no errors
- `E`: Accepted with Errors - Interchange accepted but contains errors
- `R`: Rejected - Interchange rejected due to structural errors

**Common TA1 Error Codes**:
- `000`: No Error
- `001`: Interchange Control Number (ISA13) not matching IEA02
- `005`: Invalid Date in ISA09
- `006`: Invalid Time in ISA10
- `010`: Invalid Interchange ID Qualifier (ISA05/ISA07)
- `012`: Invalid Interchange Sender/Receiver ID
- `023`: Invalid Transaction Set Count in IEA01
- `027`: Group Control Number in GE does not match GS
- `030`: Invalid Transaction Set Control Number in SE02

**Generation Pattern**: TA1 is generated synchronously during SFTP ingestion validation (within 5 minutes of file receipt). No database transaction association required.

### 999 - Functional Acknowledgment (< 15 min SLA)

**Purpose**: Validates syntax and structure of GS/GE functional groups and ST/SE transaction sets.

**Generated When**: After parsing inbound EDI file and validating X12 syntax rules.

**999 Structure**:
```text
ISA*...~
GS*FA*...~
ST*999*0001*005010X231A1~
AK1*HC*123456*005010X837P~              -- Functional group being acknowledged
AK2*837*0001*005010X837P~               -- Transaction set being acknowledged
AK5*A*5~                                -- Transaction set response (A = accepted)
AK9*A*1*1*1~                            -- Functional group response (A = accepted)
SE*5*0001~
GE*1*789~
IEA*1*000123456~
```

**999 Segments**:

**AK1 - Functional Group Response Header**:
- **AK101**: Functional Identifier Code (e.g., 'HC' for 837, 'BE' for 834)
- **AK102**: Group Control Number (from inbound GS06)
- **AK103**: Version/Release/Industry Identifier (e.g., '005010X837P')

**AK2 - Transaction Set Response Header**:
- **AK201**: Transaction Set Identifier Code (e.g., '837', '270')
- **AK202**: Transaction Set Control Number (from inbound ST02)
- **AK203**: Implementation Convention Reference (e.g., '005010X837P')

**AK5 - Transaction Set Response Trailer**:
- **AK501**: Transaction Set Acknowledgment Code (A/E/M/R/W)
- **AK502**: Syntax Error Code (1-5 for specific syntax errors)

**AK9 - Functional Group Response Trailer**:
- **AK901**: Functional Group Acknowledge Code (A/E/P/R)
- **AK902**: Number of Transaction Sets Included
- **AK903**: Number of Received Transaction Sets
- **AK904**: Number of Accepted Transaction Sets

**999 Acceptance Codes**:

**AK501 (Transaction Set Level)**:
- `A`: Accepted - No errors
- `E`: Accepted with Errors - Contains syntax errors but can be processed
- `M`: Rejected, Message Authentication Code Failed
- `R`: Rejected - Contains errors preventing processing
- `W`: Rejected, Assurance Failed Validity Tests

**AK901 (Functional Group Level)**:
- `A`: Accepted - All transaction sets accepted
- `E`: Accepted with Errors - Some transaction sets have errors
- `P`: Partially Accepted - Some transaction sets rejected
- `R`: Rejected - All transaction sets rejected

**Generation Pattern**: 999 is generated after ADF pipeline completes X12 syntax validation (< 15 min SLA). Includes detailed error reporting for rejected segments.

### 271 - Eligibility Response (< 5 min SLA)

**Purpose**: Provides eligibility and benefit information in response to 270 inquiry.

**Generated When**: After real-time eligibility lookup completes (internal claims system or partner payer API).

**271 Key Segments**:
```text
ST*271*0001*005010X279A1~
BHT*0022*11*REF12345*20250106*1530*RT~
HL*1**20*1~                                  -- Information Source (Payer)
NM1*PR*2*INSURANCE COMPANY*****PI*12345~
HL*2*1*21*1~                                 -- Information Receiver (Provider)
NM1*1P*2*PROVIDER CLINIC*****XX*1234567890~
HL*3*2*22*0~                                 -- Subscriber
NM1*IL*1*DOE*JOHN****MI*123456789~
DMG*D8*19800101*M~                           -- Demographics
DTP*307*RD8*20250101-20251231~               -- Eligibility dates
EB*1*FAM*30**MEDICAL~                        -- Eligibility or benefit info
EB*C*FAM*30**DENTAL~                         -- Co-payment info
SE*12*0001~
```

**271 Response Codes**:

**EB01 (Eligibility/Benefit Information)**:
- `1`: Active Coverage
- `2`: Active - Full Risk Capitation
- `3`: Active - Services Capitated
- `4`: Active - Services Capitated to Primary Care Physician
- `5`: Active - Pending Investigation
- `6`: Inactive
- `7`: Inactive - Pending Eligibility Update
- `8`: Inactive - Pending Investigation
- `C`: Co-payment
- `D`: Deductible

**BHT02 (Transaction Set Purpose)**:
- `11`: Response

**Generation Pattern**: 271 is generated by Inbound Mapper after partner eligibility API responds. Transformation from partner JSON/XML to canonical 271 structure. No control number coordination required (outbound orchestrator acquires ISA/GS/ST).

### 277CA - Claim Acknowledgment (< 4 hrs SLA)

**Purpose**: Confirms receipt of 837 claim and provides initial status (accepted, rejected, pended).

**Generated When**: After 837 claim is validated and routed to claims system.

**277CA Key Segments**:
```text
ST*277*0001*005010X214~
BHT*0010*08*REF12345*20250106*1530~
HL*1**20*1~                                  -- Information Source (Payer)
NM1*PR*2*INSURANCE COMPANY*****PI*12345~
HL*2*1*21*1~                                 -- Information Receiver (Provider)
NM1*41*2*PROVIDER CLINIC*****XX*1234567890~
HL*3*2*19*1~                                 -- Subscriber
NM1*IL*1*DOE*JOHN****MI*123456789~
HL*4*3*PT*0~                                 -- Patient (if different)
NM1*QC*1*SMITH*JANE****MI*987654321~
STC*A1:42:20:Claim Received~                 -- Status information
REF*D9*CLM123456~                            -- Claim identifier
DTP*472*D8*20250106~                         -- Service date
SE*14*0001~
```

**277CA Status Codes (STC Segment)**:

**Category Code (STC01-1)**:
- `A1`: Acknowledgment - Receipt confirmed
- `A2`: Acknowledgment - Accepted for processing
- `A4`: Acknowledgment - Not accepted for processing
- `A5`: Acknowledgment - Split claim
- `P1`: Pending - Information required
- `P3`: Pending - Under review

**Status Code (STC01-2)**:
- `42`: Claim received
- `43`: Claim accepted
- `44`: Claim rejected
- `18`: Duplicate claim

**BHT02 (Transaction Set Purpose)**:
- `08`: Original

**Generation Pattern**: 277CA is generated by Claims System after 837 routing completes. Batched daily or per-claim depending on partner configuration. Includes claim identifiers for reconciliation.

### 835 - Remittance Advice (Weekly SLA)

**Purpose**: Notifies provider of payment/denial decisions for submitted claims.

**Generated When**: After payer adjudication completes (weekly batch or per-payment cycle).

**835 Key Segments**:
```text
ST*835*0001*005010X221A1~
BPR*I*5000*C*ACH*CCP*01*999999999*DA*12345***01*999999999*DA*54321*20250110~
TRN*1*12345678*1234567890~
REF*EV*PROVIDER123~
DTM*405*20250110~                            -- Production date
N1*PR*INSURANCE COMPANY*PI*12345~            -- Payer
N1*PE*PROVIDER CLINIC*XX*1234567890~         -- Payee
CLP*CLM123456*1*1000*800*200*MC*REF987*11*1~ -- Claim payment info
NM1*QC*1*DOE*JOHN****MI*123456789~
DTM*232*20250101~                            -- Service date
SVC*HC:99213*1000*800**1~                    -- Service payment info
CAS*CO*45*200~                               -- Claim adjustment
SE*15*0001~
```

**835 Key Fields**:

**BPR01 (Transaction Handling Code)**:
- `I`: Payment with remittance
- `C`: Payment only
- `H`: Notification only

**BPR02**: Total actual provider payment amount
**BPR16**: Payment effective date (CCYYMMDD)

**CLP02 (Claim Status Code)**:
- `1`: Processed as primary
- `2`: Processed as secondary
- `3`: Processed as tertiary
- `4`: Denied
- `19`: Processed as primary with forwarding
- `20`: Processed as secondary with forwarding
- `21`: Processed as tertiary with forwarding
- `22`: Reversal of previous payment

**CLP03**: Total submitted charge amount
**CLP04**: Total payment amount
**CLP05**: Patient responsibility amount

**CAS (Claim Adjustment Segment)**:
- **CAS01**: Claim adjustment group code (CO = Contractual, CR = Correction, OA = Other, PI = Payer Initiated)
- **CAS02**: Claim adjustment reason code (1-999, e.g., 45 = Charge exceeds fee schedule)
- **CAS03**: Claim adjustment amount

**Generation Pattern**: 835 is generated by Payer after adjudication cycle completes. Received as inbound partner response, parsed by Inbound Mapper, stored in outbound staging for delivery to provider.

---

## Outbound Assembly Orchestration

### Azure Durable Functions Workflow

The Outbound Orchestrator uses Durable Functions to coordinate multi-step response assembly with retry logic and stateful checkpoints.

**Orchestrator Function**: `OutboundAssemblyOrchestrator`

```csharp
[FunctionName("OutboundAssemblyOrchestrator")]
public async Task RunOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext context,
    ILogger log)
{
    var batchWindow = context.GetInput<OutboundBatchWindow>();

    try
    {
        // Step 1: Collect pending outcomes from staging tables
        var outcomes = await context.CallActivityAsync<List<ProcessingOutcome>>(
            "CollectPendingOutcomes",
            batchWindow);

        log.LogInformation(
            "Collected {OutcomeCount} pending outcomes for batch window {WindowStart} to {WindowEnd}",
            outcomes.Count, batchWindow.WindowStart, batchWindow.WindowEnd);

        // Step 2: Group outcomes by partner + transaction + response type
        var outboundBatches = outcomes
            .GroupBy(o => new
            {
                o.PartnerCode,
                o.TransactionType,
                o.ResponseType
            })
            .Select(g => new OutboundBatch
            {
                PartnerCode = g.Key.PartnerCode,
                TransactionType = g.Key.TransactionType,
                ResponseType = g.Key.ResponseType,
                Outcomes = g.ToList(),
                OutboundFileId = Guid.NewGuid()
            })
            .ToList();

        log.LogInformation(
            "Grouped into {BatchCount} outbound batches",
            outboundBatches.Count);

        // Step 3: Process each outbound batch
        var tasks = new List<Task<OutboundFileResult>>();

        foreach (var batch in outboundBatches)
        {
            // Create child orchestration for parallel processing
            var task = context.CallSubOrchestratorAsync<OutboundFileResult>(
                "ProcessOutboundBatch",
                batch);
            tasks.Add(task);
        }

        // Wait for all batches to complete
        var results = await Task.WhenAll(tasks);

        // Step 4: Aggregate results
        var successCount = results.Count(r => r.Status == "SUCCESS");
        var failureCount = results.Count(r => r.Status == "FAILED");

        log.LogInformation(
            "Outbound assembly completed: {SuccessCount} success, {FailureCount} failed",
            successCount, failureCount);

        return new OrchestrationResult
        {
            SuccessCount = successCount,
            FailureCount = failureCount,
            Results = results
        };
    }
    catch (Exception ex)
    {
        log.LogError(ex,
            "Outbound assembly orchestration failed for batch window {WindowStart} to {WindowEnd}",
            batchWindow.WindowStart, batchWindow.WindowEnd);
        throw;
    }
}
```

### Sub-Orchestrator: ProcessOutboundBatch

```csharp
[FunctionName("ProcessOutboundBatch")]
public async Task<OutboundFileResult> ProcessBatch(
    [OrchestrationTrigger] IDurableOrchestrationContext context,
    ILogger log)
{
    var batch = context.GetInput<OutboundBatch>();

    try
    {
        // Step 1: Acquire control numbers (with retry)
        var controlNumbers = await context.CallActivityWithRetryAsync<ControlNumbers>(
            "AcquireControlNumbers",
            new RetryOptions(TimeSpan.FromSeconds(5), 3)
            {
                Handle = ex => ex is SqlException || ex is TimeoutException
            },
            batch);

        log.LogInformation(
            "Acquired control numbers: ISA13={Isa13}, GS06={Gs06}, ST02={St02}",
            controlNumbers.Isa13, controlNumbers.Gs06, controlNumbers.St02);

        // Step 2: Build EDI envelope
        var ediContent = await context.CallActivityAsync<string>(
            "BuildEdiEnvelope",
            new EnvelopeRequest
            {
                Batch = batch,
                ControlNumbers = controlNumbers
            });

        // Step 3: Validate envelope structure
        var validation = await context.CallActivityAsync<ValidationResult>(
            "ValidateEnvelope",
            ediContent);

        if (!validation.IsValid)
        {
            log.LogError(
                "Envelope validation failed: {ErrorMessage}",
                validation.ErrorMessage);

            // Mark control numbers as FAILED in audit
            await context.CallActivityAsync(
                "UpdateControlNumberAudit",
                new AuditUpdate
                {
                    OutboundFileId = batch.OutboundFileId,
                    Status = "FAILED",
                    Notes = validation.ErrorMessage
                });

            return new OutboundFileResult
            {
                OutboundFileId = batch.OutboundFileId,
                Status = "FAILED",
                ErrorMessage = validation.ErrorMessage
            };
        }

        // Step 4: Persist outbound file to blob storage
        var blobUri = await context.CallActivityWithRetryAsync<string>(
            "PersistOutboundFile",
            new RetryOptions(TimeSpan.FromSeconds(2), 3)
            {
                Handle = ex => ex is RequestFailedException
            },
            new FileUploadRequest
            {
                OutboundFileId = batch.OutboundFileId,
                PartnerCode = batch.PartnerCode,
                TransactionType = batch.TransactionType,
                Content = ediContent,
                ControlNumbers = controlNumbers
            });

        log.LogInformation(
            "Persisted outbound file to {BlobUri}",
            blobUri);

        // Step 5: Update control number audit to PERSISTED
        await context.CallActivityAsync(
            "UpdateControlNumberAudit",
            new AuditUpdate
            {
                OutboundFileId = batch.OutboundFileId,
                Status = "PERSISTED",
                Notes = $"File uploaded to {blobUri}"
            });

        // Step 6: Publish outbound-ready event to Service Bus
        await context.CallActivityWithRetryAsync(
            "PublishOutboundReadyEvent",
            new RetryOptions(TimeSpan.FromSeconds(1), 5)
            {
                BackoffCoefficient = 2.0,
                Handle = ex => ex is ServiceBusException
            },
            new OutboundReadyEvent
            {
                OutboundFileId = batch.OutboundFileId,
                PartnerCode = batch.PartnerCode,
                TransactionType = batch.TransactionType,
                BlobUri = blobUri,
                ISA13 = controlNumbers.Isa13,
                GS06 = controlNumbers.Gs06,
                ST02 = controlNumbers.St02,
                FileSize = ediContent.Length,
                GeneratedUtc = context.CurrentUtcDateTime
            });

        return new OutboundFileResult
        {
            OutboundFileId = batch.OutboundFileId,
            Status = "SUCCESS",
            BlobUri = blobUri,
            ControlNumbers = controlNumbers
        };
    }
    catch (Exception ex)
    {
        log.LogError(ex,
            "Failed to process outbound batch for partner {PartnerCode}, transaction {TransactionType}",
            batch.PartnerCode, batch.TransactionType);

        return new OutboundFileResult
        {
            OutboundFileId = batch.OutboundFileId,
            Status = "FAILED",
            ErrorMessage = ex.Message
        };
    }
}
```

### Activity Functions

#### AcquireControlNumbers

```csharp
[FunctionName("AcquireControlNumbers")]
public async Task<ControlNumbers> AcquireControlNumbers(
    [ActivityTrigger] OutboundBatch batch,
    ILogger log)
{
    using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    var isaNumber = await GetNextControlNumber(
        connection, batch.PartnerCode, batch.TransactionType, "ISA", batch.OutboundFileId);
    var gsNumber = await GetNextControlNumber(
        connection, batch.PartnerCode, batch.TransactionType, "GS", batch.OutboundFileId);
    var stNumber = await GetNextControlNumber(
        connection, batch.PartnerCode, batch.TransactionType, "ST", batch.OutboundFileId);

    log.LogInformation(
        "Acquired control numbers for {PartnerCode}/{TransactionType}: ISA13={Isa13}, GS06={Gs06}, ST02={St02}",
        batch.PartnerCode, batch.TransactionType, isaNumber, gsNumber, stNumber);

    return new ControlNumbers
    {
        Isa13 = isaNumber,
        Gs06 = gsNumber,
        St02 = stNumber
    };
}

private async Task<long> GetNextControlNumber(
    SqlConnection connection,
    string partnerCode,
    string transactionType,
    string counterType,
    Guid outboundFileId)
{
    using var command = new SqlCommand("dbo.usp_GetNextControlNumber", connection)
    {
        CommandType = CommandType.StoredProcedure
    };

    command.Parameters.AddWithValue("@PartnerCode", partnerCode);
    command.Parameters.AddWithValue("@TransactionType", transactionType);
    command.Parameters.AddWithValue("@CounterType", counterType);
    command.Parameters.AddWithValue("@OutboundFileId", outboundFileId);

    var outputParam = new SqlParameter("@NextControlNumber", SqlDbType.BigInt)
    {
        Direction = ParameterDirection.Output
    };
    command.Parameters.Add(outputParam);

    await command.ExecuteNonQueryAsync();

    return (long)outputParam.Value;
}
```

#### BuildEdiEnvelope

```csharp
[FunctionName("BuildEdiEnvelope")]
public async Task<string> BuildEdiEnvelope(
    [ActivityTrigger] EnvelopeRequest request,
    ILogger log)
{
    var batch = request.Batch;
    var controlNumbers = request.ControlNumbers;

    // Fetch partner configuration
    var partnerConfig = await _partnerConfigService.GetPartnerConfig(batch.PartnerCode);

    // Build transaction segments based on response type
    var transactionSegments = batch.ResponseType switch
    {
        "TA1" => BuildTa1Segments(batch.Outcomes),
        "999" => Build999Segments(batch.Outcomes),
        "271" => Build271Segments(batch.Outcomes, partnerConfig),
        "277" => Build277Segments(batch.Outcomes, partnerConfig),
        "835" => Build835Segments(batch.Outcomes, partnerConfig),
        _ => throw new InvalidOperationException($"Unknown response type: {batch.ResponseType}")
    };

    // Use envelope builder to construct full EDI file
    var ediContent = await _envelopeBuilder.BuildOutboundFile(
        batch.PartnerCode,
        batch.TransactionType,
        transactionSegments,
        controlNumbers.Isa13,
        controlNumbers.Gs06,
        controlNumbers.St02);

    log.LogInformation(
        "Built EDI envelope: {SegmentCount} segments, {TotalLength} characters",
        transactionSegments.Count, ediContent.Length);

    return ediContent;
}
```

#### PersistOutboundFile

```csharp
[FunctionName("PersistOutboundFile")]
public async Task<string> PersistOutboundFile(
    [ActivityTrigger] FileUploadRequest request,
    ILogger log)
{
    var blobName = GenerateBlobName(
        request.PartnerCode,
        request.TransactionType,
        request.ControlNumbers.Isa13,
        DateTime.UtcNow);

    var containerClient = _blobServiceClient.GetBlobContainerClient("outbound");
    var blobClient = containerClient.GetBlobClient(blobName);

    // Upload with metadata
    var metadata = new Dictionary<string, string>
    {
        { "OutboundFileId", request.OutboundFileId.ToString() },
        { "PartnerCode", request.PartnerCode },
        { "TransactionType", request.TransactionType },
        { "ISA13", request.ControlNumbers.Isa13.ToString() },
        { "GS06", request.ControlNumbers.Gs06.ToString() },
        { "ST02", request.ControlNumbers.St02.ToString() },
        { "CreatedUtc", DateTime.UtcNow.ToString("O") }
    };

    using var stream = new MemoryStream(Encoding.UTF8.GetBytes(request.Content));

    await blobClient.UploadAsync(
        stream,
        new BlobUploadOptions
        {
            Metadata = metadata,
            HttpHeaders = new BlobHttpHeaders
            {
                ContentType = "application/edi-x12"
            }
        });

    log.LogInformation(
        "Uploaded outbound file {BlobName} ({FileSize} bytes)",
        blobName, request.Content.Length);

    return blobClient.Uri.ToString();
}

private string GenerateBlobName(
    string partnerCode,
    string transactionType,
    long isa13,
    DateTime timestamp)
{
    // Path: outbound/partner=X/transaction=Y/date=Z/filename.edi
    var datePath = timestamp.ToString("yyyy-MM-dd");
    var fileName = $"{partnerCode}_{transactionType}_{isa13:D9}_{timestamp:yyyyMMddHHmmss}.edi";

    return $"partner={partnerCode}/transaction={transactionType}/date={datePath}/{fileName}";
}
```

#### PublishOutboundReadyEvent

```csharp
[FunctionName("PublishOutboundReadyEvent")]
public async Task PublishOutboundReadyEvent(
    [ActivityTrigger] OutboundReadyEvent outboundEvent,
    [ServiceBus("edi-outbound-ready", Connection = "ServiceBusConnection")] IAsyncCollector<ServiceBusMessage> outputMessages,
    ILogger log)
{
    var messageBody = JsonSerializer.Serialize(outboundEvent);

    var message = new ServiceBusMessage(messageBody)
    {
        ContentType = "application/json",
        MessageId = outboundEvent.OutboundFileId.ToString(),
        Subject = $"{outboundEvent.PartnerCode}/{outboundEvent.TransactionType}"
    };

    message.ApplicationProperties["PartnerCode"] = outboundEvent.PartnerCode;
    message.ApplicationProperties["TransactionType"] = outboundEvent.TransactionType;
    message.ApplicationProperties["ISA13"] = outboundEvent.ISA13.ToString();

    await outputMessages.AddAsync(message);

    log.LogInformation(
        "Published outbound-ready event for file {OutboundFileId}",
        outboundEvent.OutboundFileId);
}
```

### Batch Window Configuration

Outbound responses are batched by response type to optimize throughput:

| Response Type | Batch Window | Trigger Frequency | Rationale |
|---------------|--------------|-------------------|-----------|
| TA1 | 0 minutes (real-time) | Immediate | Critical structural validation, < 5 min SLA |
| 271 | 2 minutes | Every 2 min | Real-time eligibility, minimize latency |
| 999 | 5 minutes | Every 5 min | Syntax validation, < 15 min SLA |
| 277CA | 60 minutes | Every hour | Claim acknowledgment, < 4 hr SLA |
| 835 | N/A (event-driven) | On payer file receipt | Weekly remittance cycle |

**Timer Trigger Function**:

```csharp
[FunctionName("OutboundAssemblyTimer")]
public async Task RunTimer(
    [TimerTrigger("0 */5 * * * *")] TimerInfo timer,  // Every 5 minutes
    [DurableClient] IDurableOrchestrationClient starter,
    ILogger log)
{
    var batchWindow = new OutboundBatchWindow
    {
        WindowStart = DateTime.UtcNow.AddMinutes(-5),
        WindowEnd = DateTime.UtcNow
    };

    var instanceId = await starter.StartNewAsync(
        "OutboundAssemblyOrchestrator",
        batchWindow);

    log.LogInformation(
        "Started outbound assembly orchestration {InstanceId} for window {WindowStart} to {WindowEnd}",
        instanceId, batchWindow.WindowStart, batchWindow.WindowEnd);
}
```

---

## Response File Management

### File Naming Convention

Outbound EDI files follow a consistent naming convention for partner reconciliation:

**Format**: `{partnerCode}_{transactionType}_{interchangeControl}_{timestampUTC}.edi`

**Examples**:
- `PARTNERA_271_000123456_20250106153045.edi`
- `INTERNALCLAIMS_277_000789012_20250106140000.edi`
- `TEST001_999_000000123_20250106120000.edi`

**Fields**:
- `partnerCode`: Trading partner identifier (e.g., 'PARTNERA', 'TEST001')
- `transactionType`: X12 transaction set code (e.g., '271', '999', '835')
- `interchangeControl`: ISA13 control number (9-digit zero-padded)
- `timestampUTC`: File generation timestamp in UTC (YYYYMMDDHHmmss)

### Storage Structure

Outbound files are organized hierarchically for efficient querying and lifecycle management:

**Blob Container**: `outbound`

**Path Structure**: `partner={partnerCode}/transaction={transactionType}/date={yyyy-MM-dd}/{filename}.edi`

**Examples**:
```text
outbound/
├── partner=PARTNERA/
│   ├── transaction=271/
│   │   ├── date=2025-01-06/
│   │   │   ├── PARTNERA_271_000123456_20250106153045.edi
│   │   │   └── PARTNERA_271_000123457_20250106154000.edi
│   │   └── date=2025-01-07/
│   │       └── PARTNERA_271_000123458_20250107080000.edi
│   └── transaction=999/
│       └── date=2025-01-06/
│           └── PARTNERA_999_000987654_20250106150000.edi
└── partner=INTERNALCLAIMS/
    └── transaction=277/
        └── date=2025-01-06/
            └── INTERNALCLAIMS_277_000456789_20250106140000.edi
```

**Benefits**:
- **Partition Pruning**: Date-based partitioning optimizes blob queries by date range
- **Partner Isolation**: Each partner has dedicated folder for access control
- **Transaction Segregation**: Simplifies monitoring and reporting by transaction type

### Blob Metadata

Each outbound file includes comprehensive metadata for tracking and reconciliation:

```csharp
var metadata = new Dictionary<string, string>
{
    // Core identifiers
    { "OutboundFileId", request.OutboundFileId.ToString() },
    { "PartnerCode", request.PartnerCode },
    { "TransactionType", request.TransactionType },
    
    // Control numbers
    { "ISA13", request.ControlNumbers.Isa13.ToString() },
    { "GS06", request.ControlNumbers.Gs06.ToString() },
    { "ST02", request.ControlNumbers.St02.ToString() },
    
    // Timestamps
    { "CreatedUtc", DateTime.UtcNow.ToString("O") },
    { "BatchWindowStart", batchWindow.WindowStart.ToString("O") },
    { "BatchWindowEnd", batchWindow.WindowEnd.ToString("O") },
    
    // Content metadata
    { "FileSize", request.Content.Length.ToString() },
    { "SegmentCount", segmentCount.ToString() },
    { "TransactionCount", transactionCount.ToString() },
    
    // Response metadata
    { "ResponseType", request.ResponseType },  // TA1, 999, 271, 277, 835
    { "OutcomeCount", batch.Outcomes.Count.ToString() },
    
    // Delivery tracking (updated by SFTP Connector)
    { "DeliveryStatus", "PENDING" },  // PENDING, DELIVERED, FAILED
    { "DeliveryAttempts", "0" },
    { "LastDeliveryAttemptUtc", "" }
};
```

**Metadata Usage**:
- **Reconciliation**: Match outbound file to control number audit records
- **Monitoring**: Track delivery status without downloading file content
- **Debugging**: Correlate file to batch window and source outcomes
- **Lifecycle**: Determine retention policy based on creation date

### Lifecycle Management

Outbound files follow automated lifecycle policies to optimize storage costs:

**Bicep Lifecycle Policy**:

```bicep
resource lifecyclePolicy 'Microsoft.Storage/storageAccounts/managementPolicies@2023-01-01' = {
  name: 'default'
  parent: storageAccount
  properties: {
    policy: {
      rules: [
        {
          name: 'MoveOutboundToCoolAfter90Days'
          enabled: true
          type: 'Lifecycle'
          definition: {
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['outbound/']
            }
            actions: {
              baseBlob: {
                tierToCool: {
                  daysAfterModificationGreaterThan: 90
                }
                tierToArchive: {
                  daysAfterModificationGreaterThan: 365
                }
                delete: {
                  daysAfterModificationGreaterThan: 2555  // 7 years HIPAA retention
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
      ]
    }
  }
}
```

**Lifecycle Stages**:

| Stage | Tier | Days | Access Latency | Cost (per GB/month) | Use Case |
|-------|------|------|----------------|---------------------|----------|
| **Active** | Hot | 0-90 | < 1 ms | $0.018 | Operational access, delivery retries |
| **Compliance** | Cool | 90-365 | < 1 ms | $0.010 (44% savings) | Audit, partner inquiries |
| **Archive** | Archive | 365-2,555 | 1-15 hrs | $0.002 (89% savings) | Legal hold, regulatory |
| **Purge** | Deleted | > 2,555 | N/A | $0 | HIPAA 7-year retention complete |

**Manual Rehydration** (Archive tier):

```csharp
public async Task<string> DownloadOutboundFile(string blobUri)
{
    var blobClient = new BlobClient(new Uri(blobUri));
    var properties = await blobClient.GetPropertiesAsync();

    if (properties.Value.AccessTier == AccessTier.Archive)
    {
        // Initiate rehydration to Hot tier
        await blobClient.SetAccessTierAsync(AccessTier.Hot, rehydratePriority: RehydratePriority.High);

        throw new BlobArchivedException(
            $"Blob is archived. Rehydration initiated (priority: High). Retry in 1-15 hours.");
    }

    // Download if available
    var downloadResponse = await blobClient.DownloadContentAsync();
    return downloadResponse.Value.Content.ToString();
}
```

---

## Error Handling

### Control Number Acquisition Timeout

**Scenario**: Control number store unresponsive or experiencing high concurrency collision rate.

**Symptoms**:
- Activity function `AcquireControlNumbers` exceeds 30-second timeout
- Audit logs show FAILED status with "Maximum retries exceeded" error
- Control number gaps appear in ControlNumberGaps view

**Root Causes**:
- SQL Database DTU exhaustion (> 90% utilization)
- Blocking queries on ControlNumberCounters table
- Network latency between Function App and SQL Database

**Remediation**:

1. **Check SQL DTU utilization**:
```powershell
$resourceGroup = "rg-edi-outbound-prod"
$serverName = "sql-edi-controlnumbers-prod"
$databaseName = "edi-controlnumbers"

# Query DTU metrics from Azure Monitor
az monitor metrics list `
    --resource "/subscriptions/{subId}/resourceGroups/$resourceGroup/providers/Microsoft.Sql/servers/$serverName/databases/$databaseName" `
    --metric "dtu_consumption_percent" `
    --start-time (Get-Date).AddHours(-1).ToString("yyyy-MM-ddTHH:mm:ssZ") `
    --interval PT1M `
    --output table
```

2. **Identify blocking queries**:
```sql
SELECT
    blocking_session_id,
    wait_type,
    wait_time,
    wait_resource,
    t.text AS blocking_query
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(sql_handle) t
WHERE blocking_session_id <> 0;
```

3. **Scale SQL Database tier** (if DTU > 90%):
```powershell
# Scale from S1 (20 DTU) to S2 (50 DTU)
az sql db update `
    --resource-group $resourceGroup `
    --server $serverName `
    --name $databaseName `
    --service-objective S2
```

4. **Increase connection timeout**:
```csharp
// In Function App application settings
"SqlConnectionTimeout": "60"  // Increase from 30s to 60s

// Update connection string builder
var builder = new SqlConnectionStringBuilder(_connectionString)
{
    ConnectTimeout = _configuration.GetValue<int>("SqlConnectionTimeout", 30)
};
```

5. **Kill blocking session** (last resort):
```sql
-- Identify long-running blocking session
SELECT session_id, blocking_session_id, wait_time, command, status, t.text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(sql_handle) t
WHERE blocking_session_id <> 0;

-- Kill blocking session (use with caution)
KILL 123;  -- Replace with actual blocking_session_id
```

### Envelope Validation Failure

**Scenario**: Generated EDI file fails segment count validation or control number consistency check.

**Symptoms**:
- Activity function `ValidateEnvelope` returns `IsValid = false`
- Control number audit shows FAILED status with validation error message
- Control number gap detected (number issued but file not persisted)

**Root Causes**:
- Segment builder logic error (incorrect SE01 count)
- Control number mismatch (GE02 ≠ GS06 or IEA02 ≠ ISA13)
- Envelope template corruption (missing segments)

**Remediation**:

1. **Query failed validation details**:
```sql
SELECT
    a.AuditId,
    a.ControlNumberIssued,
    a.OutboundFileId,
    a.Status,
    a.Notes,
    c.PartnerCode,
    c.TransactionType,
    c.CounterType
FROM dbo.ControlNumberAudit a
INNER JOIN dbo.ControlNumberCounters c ON a.CounterId = c.CounterId
WHERE a.Status = 'FAILED'
  AND a.IssuedUtc >= DATEADD(HOUR, -24, GETUTCDATE())
ORDER BY a.IssuedUtc DESC;
```

2. **Review validation error message**:
```csharp
log.LogError(
    "Envelope validation failed for OutboundFileId {OutboundFileId}: {ErrorMessage}. " +
    "Segment counts: ST={StCount}, SE={SeCount}, GS={GsCount}, GE={GeCount}, ISA={IsaCount}, IEA={IeaCount}",
    batch.OutboundFileId,
    validation.ErrorMessage,
    validation.StCount,
    validation.SeCount,
    validation.GsCount,
    validation.GeCount,
    validation.IsaCount,
    validation.IeaCount);
```

3. **Fix envelope template** (if segment count mismatch):
```csharp
// Correct SE01 segment count calculation
var segmentCount = transactionSegments.Count + 2;  // ST + segments + SE

// Verify GE01 matches number of ST segments
var stCount = segments.Count(s => s.StartsWith("ST*"));

// Verify IEA01 matches number of GS segments
var gsCount = segments.Count(s => s.StartsWith("GS*"));
```

4. **Document control number gap**:
```sql
-- Update audit record with gap explanation
UPDATE dbo.ControlNumberAudit
SET Notes = 'Gap due to envelope validation failure: ' + @ValidationErrorMessage
WHERE OutboundFileId = @OutboundFileId
  AND Status = 'FAILED';
```

5. **Alert Engineering** (for repeated validation failures):
```kql
AzureDiagnostics
| where Category == "FunctionAppLogs"
| where FunctionName == "ValidateEnvelope"
| where Level == "Error"
| where TimeGenerated > ago(1h)
| summarize FailureCount = count() by bin(TimeGenerated, 5m), PartnerCode = tostring(customDimensions.PartnerCode)
| where FailureCount > 10
| project TimeGenerated, PartnerCode, FailureCount
```

### Blob Storage Write Failure

**Scenario**: Outbound file cannot be persisted to Azure Blob Storage.

**Symptoms**:
- Activity function `PersistOutboundFile` throws `RequestFailedException`
- Durable Function retry attempts exhausted (3 attempts)
- Control number issued but no file created

**Root Causes**:
- Blob storage throttling (HTTP 503)
- Network connectivity issues
- Storage account firewall rules blocking Function App
- Insufficient RBAC permissions (missing Storage Blob Data Contributor)

**Remediation**:

1. **Check storage account metrics**:
```powershell
# Query throttling metrics
az monitor metrics list `
    --resource "/subscriptions/{subId}/resourceGroups/rg-edi-outbound-prod/providers/Microsoft.Storage/storageAccounts/stedioutboundprod" `
    --metric "Availability" `
    --start-time (Get-Date).AddHours(-1).ToString("yyyy-MM-ddTHH:mm:ssZ") `
    --interval PT1M `
    --output table
```

2. **Verify Function App network access**:
```powershell
# Check if Function App IP is allowed in storage firewall
az storage account network-rule list `
    --account-name stedioutboundprod `
    --resource-group rg-edi-outbound-prod `
    --output table
```

3. **Verify RBAC permissions**:
```powershell
# Check Function App managed identity has Storage Blob Data Contributor
$principalId = az functionapp identity show `
    --name func-edi-outbound-prod `
    --resource-group rg-edi-outbound-prod `
    --query principalId `
    --output tsv

az role assignment list `
    --assignee $principalId `
    --scope "/subscriptions/{subId}/resourceGroups/rg-edi-outbound-prod/providers/Microsoft.Storage/storageAccounts/stedioutboundprod" `
    --output table
```

4. **Enable diagnostic logging**:
```csharp
// Add detailed logging to PersistOutboundFile activity
try
{
    await blobClient.UploadAsync(stream, options);
}
catch (RequestFailedException ex) when (ex.Status == 503)
{
    log.LogError(ex,
        "Storage throttling detected for blob {BlobName}. " +
        "ErrorCode: {ErrorCode}, RequestId: {RequestId}",
        blobName, ex.ErrorCode, ex.ClientRequestId);
    throw;
}
catch (RequestFailedException ex) when (ex.Status == 403)
{
    log.LogError(ex,
        "Access denied to blob {BlobName}. Verify RBAC permissions. " +
        "ErrorCode: {ErrorCode}, RequestId: {RequestId}",
        blobName, ex.ErrorCode, ex.ClientRequestId);
    throw;
}
```

5. **Dead-letter failed orchestrations**:
```csharp
// In ProcessOutboundBatch sub-orchestrator
catch (RequestFailedException ex) when (context.CurrentRetryCount >= 3)
{
    // Move to dead-letter queue after max retries
    await context.CallActivityAsync(
        "DeadLetterOutboundBatch",
        new DeadLetterRequest
        {
            OutboundFileId = batch.OutboundFileId,
            ErrorMessage = ex.Message,
            ErrorCode = ex.ErrorCode,
            FailedOperation = "PersistOutboundFile"
        });

    return new OutboundFileResult
    {
        OutboundFileId = batch.OutboundFileId,
        Status = "DEAD_LETTER",
        ErrorMessage = $"Failed after {context.CurrentRetryCount} retries: {ex.Message}"
    };
}
```

### Service Bus Publish Failure

**Scenario**: Outbound file persisted but Service Bus event publication fails.

**Symptoms**:
- Activity function `PublishOutboundReadyEvent` throws `ServiceBusException`
- Outbound file exists in blob storage with `DeliveryStatus = PENDING`
- SFTP Connector never receives delivery notification

**Root Causes**:
- Service Bus throttling (exceeded messaging quota)
- Service Bus namespace not accessible from Function App VNet
- Insufficient RBAC permissions (missing Service Bus Data Sender)
- Message size exceeds 256 KB limit

**Remediation**:

1. **Check Service Bus metrics**:
```kql
AzureMetrics
| where ResourceProvider == "MICROSOFT.SERVICEBUS"
| where MetricName in ("IncomingMessages", "ThrottledRequests", "ServerErrors")
| where TimeGenerated > ago(1h)
| summarize
    TotalMessages = sum(Total),
    ThrottledRequests = sum(iff(MetricName == "ThrottledRequests", Total, 0)),
    ServerErrors = sum(iff(MetricName == "ServerErrors", Total, 0))
    by bin(TimeGenerated, 5m)
| project TimeGenerated, TotalMessages, ThrottledRequests, ServerErrors
```

2. **Verify Service Bus RBAC**:
```powershell
# Check Function App managed identity has Service Bus Data Sender
az role assignment list `
    --assignee $principalId `
    --scope "/subscriptions/{subId}/resourceGroups/rg-edi-outbound-prod/providers/Microsoft.ServiceBus/namespaces/sb-edi-routing-prod" `
    --output table
```

3. **Implement retry with exponential backoff**:
```csharp
// Already configured in ProcessOutboundBatch sub-orchestrator
await context.CallActivityWithRetryAsync(
    "PublishOutboundReadyEvent",
    new RetryOptions(TimeSpan.FromSeconds(1), 5)
    {
        BackoffCoefficient = 2.0,  // 1s, 2s, 4s, 8s, 16s
        Handle = ex => ex is ServiceBusException
    },
    outboundEvent);
```

4. **Enable dead-letter processing**:
```csharp
// Create compensation activity for cleanup
[FunctionName("RollbackPersistedFile")]
public async Task RollbackPersistedFile(
    [ActivityTrigger] RollbackRequest request,
    ILogger log)
{
    // Move file to error container or add metadata flag
    var blobClient = new BlobClient(new Uri(request.BlobUri));
    var metadata = (await blobClient.GetPropertiesAsync()).Value.Metadata;
    
    metadata["DeliveryStatus"] = "SERVICE_BUS_FAILED";
    metadata["ErrorMessage"] = request.ErrorMessage;
    
    await blobClient.SetMetadataAsync(metadata);
    
    log.LogWarning(
        "Marked outbound file {BlobUri} as SERVICE_BUS_FAILED due to: {ErrorMessage}",
        request.BlobUri, request.ErrorMessage);
}
```

5. **Manual recovery script**:
```powershell
# Query blobs with failed Service Bus delivery
$blobs = az storage blob list `
    --account-name stedioutboundprod `
    --container-name outbound `
    --query "[?metadata.DeliveryStatus=='SERVICE_BUS_FAILED'].{name:name,created:properties.creationTime,fileId:metadata.OutboundFileId}" `
    --output json | ConvertFrom-Json

# Retry Service Bus publish for each failed file
foreach ($blob in $blobs) {
    Write-Host "Retrying Service Bus publish for $($blob.name)..."
    
    # Call retry API endpoint
    Invoke-RestMethod -Method Post -Uri "https://func-edi-outbound-prod.azurewebsites.net/api/RetryServiceBusPublish" `
        -Headers @{ "x-functions-key" = $functionKey } `
        -Body (@{ "OutboundFileId" = $blob.fileId } | ConvertTo-Json) `
        -ContentType "application/json"
}
```

### Control Number Gap Detection & Remediation

**Scenario**: Control number gaps detected during routine monitoring.

**Symptoms**:
- `ControlNumberGaps` view shows missing control numbers
- Alert "Critical gap detected" fires from Azure Monitor
- Partner reports missing acknowledgments

**Root Causes**:
- Transient failures during outbound assembly (envelope validation, blob write, Service Bus)
- Orchestration timeout before file persistence
- Database failure during audit insert

**Remediation**:

1. **Run gap detection stored procedure**:
```sql
EXEC dbo.usp_DetectControlNumberGaps
    @PartnerCode = 'PARTNERA',
    @DaysToCheck = 7;
```

**Example Output**:
| PartnerCode | TransactionType | CounterType | GapStart | GapEnd | GapSize | Severity |
|-------------|-----------------|-------------|----------|--------|---------|----------|
| PARTNERA | 271 | ISA | 123455 | 123457 | 1 | MINOR |
| PARTNERA | 999 | GS | 789 | 795 | 5 | MODERATE |

2. **Investigate gap audit records**:
```sql
SELECT
    a.ControlNumberIssued,
    a.OutboundFileId,
    a.Status,
    a.Notes,
    a.IssuedUtc,
    a.RetryCount
FROM dbo.ControlNumberAudit a
INNER JOIN dbo.ControlNumberCounters c ON a.CounterId = c.CounterId
WHERE c.PartnerCode = 'PARTNERA'
  AND c.TransactionType = '271'
  AND c.CounterType = 'ISA'
  AND a.ControlNumberIssued BETWEEN 123455 AND 123457
ORDER BY a.ControlNumberIssued;
```

3. **Determine gap action**:

**Option A - Accept Gap** (transient failure, no business impact):
```sql
UPDATE dbo.ControlNumberAudit
SET
    Status = 'FAILED',
    Notes = 'Gap accepted: envelope validation failed due to segment count mismatch. No business impact.'
WHERE ControlNumberIssued = 123456
  AND Status = 'ISSUED';
```

**Option B - Reprocess** (re-run failed orchestration):
```powershell
# Get OutboundFileId from audit record
$outboundFileId = "guid-from-audit-record"

# Trigger reprocessing via HTTP endpoint
Invoke-RestMethod -Method Post -Uri "https://func-edi-outbound-prod.azurewebsites.net/api/ReprocessOutboundBatch" `
    -Headers @{ "x-functions-key" = $functionKey } `
    -Body (@{ "OutboundFileId" = $outboundFileId } | ConvertTo-Json) `
    -ContentType "application/json"
```

**Option C - Document Gap** (business accepts gap with explanation):
```sql
UPDATE dbo.ControlNumberAudit
SET
    Status = 'FAILED',
    Notes = 'Gap documented: orchestration timeout during blob write. Business accepts gap after partner notification.'
WHERE ControlNumberIssued IN (123456, 789, 790, 791, 792, 793)
  AND Status = 'ISSUED';
```

4. **Partner notification** (for critical gaps):
```text
Subject: EDI Control Number Gap Notification - PARTNERA 271

Dear Trading Partner,

We have detected a gap in outbound control numbers for transaction type 271:

- Gap Range: ISA13 123456 (missing)
- Date: 2025-01-06 15:30 UTC
- Cause: Transient system error during file generation
- Impact: 271 eligibility response not delivered
- Resolution: We have reprocessed the missing file. ISA13 123456 will be delivered within 1 hour.

Please confirm receipt of the following files:
- PARTNERA_271_000123455_20250106153000.edi
- PARTNERA_271_000123456_20250106160000.edi (reprocessed)
- PARTNERA_271_000123457_20250106153045.edi

If you have questions, contact EDI Support at edi-support@example.com.

Thank you,
EDI Operations Team
```

### High Retry Rate (Concurrency Collisions)

**Scenario**: Control number acquisition experiencing > 10% retry rate due to ROWVERSION collisions.

**Symptoms**:
- Application Insights traces show repeated "Concurrency collision - retry" messages
- p95 latency for `AcquireControlNumbers` exceeds 200ms
- SQL Database DTU > 80%

**Root Causes**:
- Too many concurrent Function App instances (> 10)
- Batch window too short (< 5 min) causing thundering herd
- SQL Database insufficient capacity (S1 20 DTU)

**Remediation**:

1. **Query retry rate**:
```kql
traces
| where message contains "Concurrency collision"
| where timestamp > ago(1h)
| extend RetryCount = toint(customDimensions.RetryCount)
| summarize
    TotalAcquisitions = count(),
    AvgRetryCount = avg(RetryCount),
    MaxRetryCount = max(RetryCount)
    by bin(timestamp, 5m), PartnerCode = tostring(customDimensions.PartnerCode)
| extend RetryRate = (AvgRetryCount / 5.0) * 100  // % of acquisitions requiring retry
| where RetryRate > 10
| project timestamp, PartnerCode, RetryRate, AvgRetryCount, MaxRetryCount
```

2. **Scale SQL Database to S3** (100 DTU):
```powershell
az sql db update `
    --resource-group rg-edi-outbound-prod `
    --server sql-edi-controlnumbers-prod `
    --name edi-controlnumbers `
    --service-objective S3
```

3. **Reduce Function App max instances** (limit concurrency):
```powershell
az functionapp config appsettings set `
    --name func-edi-outbound-prod `
    --resource-group rg-edi-outbound-prod `
    --settings "WEBSITE_MAX_DYNAMIC_APPLICATION_SCALE_OUT=5"
```

4. **Adjust batch window** (reduce thundering herd):
```csharp
// Increase batch window from 5 min to 10 min
var batchWindow = new OutboundBatchWindow
{
    WindowStart = DateTime.UtcNow.AddMinutes(-10),  // Changed from -5
    WindowEnd = DateTime.UtcNow
};
```

5. **Implement Redis cache** (future enhancement):
```csharp
// Cache control number reads, optimistic increment, async validation
public async Task<long> GetNextControlNumber(
    string partnerCode,
    string transactionType,
    string counterType)
{
    var cacheKey = $"counter:{partnerCode}:{transactionType}:{counterType}";
    
    // Try cache first
    var cachedValue = await _redis.StringGetAsync(cacheKey);
    if (cachedValue.HasValue)
    {
        var nextValue = await _redis.StringIncrementAsync(cacheKey);
        
        // Async validate against SQL (eventual consistency)
        _ = Task.Run(() => ValidateControlNumberAsync(cacheKey, nextValue));
        
        return nextValue;
    }
    
    // Cache miss - query SQL
    var sqlValue = await GetNextControlNumberFromSql(partnerCode, transactionType, counterType);
    await _redis.StringSetAsync(cacheKey, sqlValue, TimeSpan.FromMinutes(5));
    
    return sqlValue;
}
```

### Rollover Imminent (Control Number Approaching Max Value)

**Scenario**: Control number approaching 90% of MaxValue (899,999,999 for 9-digit ISA13).

**Symptoms**:
- Alert "Control number rollover warning" fires from Azure Monitor
- CurrentValue > 900,000,000

**Root Causes**:
- High-volume partner (> 1M files/day)
- MaxValue too low (should be 999,999,999 for 9-digit ISA13)

**Remediation**:

1. **Identify partners approaching rollover**:
```sql
SELECT
    PartnerCode,
    TransactionType,
    CounterType,
    CurrentValue,
    MaxValue,
    CAST((CurrentValue * 100.0 / MaxValue) AS DECIMAL(5,2)) AS PercentUsed,
    DATEADD(DAY,
        (MaxValue - CurrentValue) / (CurrentValue / DATEDIFF(DAY, CreatedUtc, GETUTCDATE())),
        GETUTCDATE()
    ) AS EstimatedRolloverDate
FROM dbo.ControlNumberCounters
WHERE CurrentValue > (MaxValue * 0.90)
ORDER BY PercentUsed DESC;
```

2. **Coordinate with trading partner** (7-day notice):
```text
Subject: EDI Control Number Rollover Notification - PARTNERA 271

Dear Trading Partner,

We will be performing a scheduled control number rollover for transaction type 271:

- Current ISA13: 899,123,456 (90% of max 999,999,999)
- Rollover Date: 2025-01-13 02:00 UTC (7 days)
- New Starting ISA13: 1
- Expected Downtime: None (seamless transition)

Action Required:
- Update your EDI reconciliation systems to expect ISA13 = 1 on 2025-01-13
- Verify your systems handle control number rollover correctly
- Contact us if you need to adjust the rollover date

Technical Details:
- All control numbers issued before rollover will complete normally
- After rollover, new control numbers start at 1
- No files will be lost or duplicated
- Gap detection systems will be updated with rollover marker

Please confirm receipt of this notification by replying to edi-support@example.com.

Thank you,
EDI Operations Team
```

3. **Schedule maintenance window**:
```powershell
# Set maintenance window tag on resources
az resource tag `
    --ids "/subscriptions/{subId}/resourceGroups/rg-edi-outbound-prod/providers/Microsoft.Sql/servers/sql-edi-controlnumbers-prod/databases/edi-controlnumbers" `
    --tags "MaintenanceWindow=2025-01-13T02:00:00Z" "MaintenanceType=ControlNumberRollover"
```

4. **Execute rollover**:
```sql
-- Execute stored procedure during maintenance window
EXEC dbo.usp_ResetControlNumber
    @PartnerCode = 'PARTNERA',
    @TransactionType = '271',
    @CounterType = 'ISA',
    @NewValue = 1,
    @Reason = 'Planned rollover: CurrentValue exceeded 90% of MaxValue (899,123,456 / 999,999,999). Partner notified 2025-01-06.';

-- Verify reset
SELECT CurrentValue, LastIncrementUtc, ModifiedUtc
FROM dbo.ControlNumberCounters
WHERE PartnerCode = 'PARTNERA'
  AND TransactionType = '271'
  AND CounterType = 'ISA';
```

5. **Post-rollover verification**:
```sql
-- Verify first file after rollover uses ISA13 = 1
SELECT TOP 1
    ControlNumberIssued,
    OutboundFileId,
    IssuedUtc,
    Status
FROM dbo.ControlNumberAudit
WHERE CounterId = (
    SELECT CounterId
    FROM dbo.ControlNumberCounters
    WHERE PartnerCode = 'PARTNERA'
      AND TransactionType = '271'
      AND CounterType = 'ISA'
)
ORDER BY IssuedUtc DESC;

-- Expected: ControlNumberIssued = 1
```

---

## Security & Compliance

### Authentication & Authorization

**Azure SQL Database**:
- **Managed Identity**: Function App uses system-assigned managed identity for passwordless authentication
- **Database Roles**: `db_datawriter` + `db_datareader` roles assigned to Function App principal
- **Network Isolation**: Private endpoint enforces VNet-only access

```sql
-- Grant Function App managed identity database access
CREATE USER [func-edi-outbound-prod] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datawriter ADD MEMBER [func-edi-outbound-prod];
ALTER ROLE db_datareader ADD MEMBER [func-edi-outbound-prod];
GO
```

**Azure Blob Storage**:
- **RBAC**: `Storage Blob Data Contributor` role assigned to Function App managed identity
- **Access Tier**: Hot tier for outbound container (operational access)
- **Network Isolation**: Private endpoint + firewall rules block public access

```powershell
# Assign Storage Blob Data Contributor role
az role assignment create `
    --assignee $principalId `
    --role "Storage Blob Data Contributor" `
    --scope "/subscriptions/{subId}/resourceGroups/rg-edi-outbound-prod/providers/Microsoft.Storage/storageAccounts/stedioutboundprod"
```

**Azure Service Bus**:
- **RBAC**: `Service Bus Data Sender` role assigned to Function App managed identity
- **Topic Authorization**: Only outbound orchestrator can publish to `edi-outbound-ready` topic
- **Network Isolation**: Private endpoint enforces VNet-only access

```powershell
# Assign Service Bus Data Sender role
az role assignment create `
    --assignee $principalId `
    --role "Azure Service Bus Data Sender" `
    --scope "/subscriptions/{subId}/resourceGroups/rg-edi-outbound-prod/providers/Microsoft.ServiceBus/namespaces/sb-edi-routing-prod/topics/edi-outbound-ready"
```

### Data Encryption

**At-Rest Encryption**:
- **SQL Database**: Transparent Data Encryption (TDE) enabled with Microsoft-managed keys
- **Blob Storage**: AES-256 encryption with Microsoft-managed keys (or customer-managed keys via Key Vault)
- **Service Bus**: Encrypted at rest with Microsoft-managed keys

**In-Transit Encryption**:
- **SQL Database**: TLS 1.2+ enforced via connection string (`Encrypt=True; TrustServerCertificate=False`)
- **Blob Storage**: HTTPS-only access enforced via storage account policy
- **Service Bus**: TLS 1.2+ enforced via Service Bus namespace policy

### PHI Protection

**Control Number Store**:
- **No PHI Storage**: Control number counters and audit records contain NO patient health information
- **Metadata Only**: PartnerCode, TransactionType, ControlNumber, OutboundFileId are non-PHI identifiers
- **HIPAA Compliant**: Audit trail provides chain of custody without storing sensitive data

**Outbound Files**:
- **PHI in Transit**: EDI files contain patient demographics, eligibility, claims (HIPAA Protected Health Information)
- **Blob Encryption**: AES-256 at rest + TLS 1.2 in transit
- **Access Logging**: All blob access logged to Log Analytics for audit trail

### Audit Logging

**SQL Database Auditing**:

```sql
-- Enable database auditing
ALTER DATABASE [edi-controlnumbers]
SET AUDIT ON
WITH (
    AUDIT_NAME = 'ControlNumberAudit',
    LOG_DESTINATION = 'LOG_ANALYTICS',
    LOG_ANALYTICS_WORKSPACE_ID = '{workspaceId}'
);

-- Audit categories
EXEC sys.sp_set_database_firewall_rule
    @name = N'AllowFunctionAppOnly',
    @start_ip_address = '{functionAppOutboundIp}',
    @end_ip_address = '{functionAppOutboundIp}';

-- Audit stored procedure executions
CREATE SERVER AUDIT SPECIFICATION ControlNumberOperations
FOR SERVER AUDIT ControlNumberAudit
ADD (EXECUTE ON OBJECT::dbo.usp_GetNextControlNumber BY [func-edi-outbound-prod]),
ADD (EXECUTE ON OBJECT::dbo.usp_ResetControlNumber BY [func-edi-outbound-prod]),
ADD (SCHEMA_OBJECT_CHANGE_GROUP),
ADD (DATABASE_OBJECT_CHANGE_GROUP);
GO
```

**Blob Storage Diagnostic Logs**:

```bicep
resource diagnosticLogs 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'outbound-blob-diagnostics'
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
    ]
  }
}
```

---

## Performance & Scalability

### Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| Control Number Acquisition | < 50ms (p95) | Application Insights `AcquireControlNumbers` duration |
| Outbound Assembly | < 10 min (p95) | Durable Function orchestration duration |
| File Persistence | < 5 sec (p95) | Blob upload duration for 100 KB file |
| Service Bus Publish | < 500ms (p95) | Service Bus send duration |
| Throughput | 100+ files/min | Files persisted per minute (peak) |
| Retry Rate | < 5% | Control number acquisition retries / total acquisitions |
| Gap Query | < 5 sec | `usp_DetectControlNumberGaps` execution time |

### SQL Database Optimization

**Indexing Strategy**:

```sql
-- Covering index for control number lookups
CREATE UNIQUE INDEX [UQ_ControlNumberCounters_Key]
    ON [dbo].[ControlNumberCounters] ([PartnerCode], [TransactionType], [CounterType])
    INCLUDE ([CurrentValue], [LastIncrementUtc], [RowVersion]);

-- Index for audit queries by OutboundFileId
CREATE INDEX [IX_ControlNumberAudit_OutboundFileId]
    ON [dbo].[ControlNumberAudit] ([OutboundFileId])
    INCLUDE ([ControlNumberIssued], [Status], [IssuedUtc]);

-- Index for gap detection (LAG window function)
CREATE INDEX [IX_ControlNumberAudit_CounterId_ControlNumber]
    ON [dbo].[ControlNumberAudit] ([CounterId], [ControlNumberIssued])
    INCLUDE ([Status], [IssuedUtc])
    WHERE Status IN ('ISSUED', 'PERSISTED');
```

**Connection Pooling**:

```json
{
  "SqlConnectionString": "Server=tcp:sql-edi-controlnumbers-prod.database.windows.net,1433;Database=edi-controlnumbers;Authentication=Active Directory Managed Identity;Min Pool Size=10;Max Pool Size=100;Connect Timeout=30;Encrypt=True;TrustServerCertificate=False;"
}
```

**Key Settings**:
- `Min Pool Size = 10`: Maintain 10 warm connections per Function App instance
- `Max Pool Size = 100`: Allow up to 100 concurrent connections
- `Connect Timeout = 30`: Fail fast if connection takes > 30 seconds

### Function App Scaling

**Premium Plan (EP1)**:
- **Always-On**: Enabled (eliminates cold start)
- **Min Instances**: 2 (high availability)
- **Max Instances**: 10 (controlled scale-out to limit SQL concurrency)
- **Auto-Scale Trigger**: CPU > 75% or Queue Length > 100

**Scale-Out Strategy**:

```bicep
resource autoScaleSettings 'Microsoft.Insights/autoscalesettings@2022-10-01' = {
  name: 'func-edi-outbound-autoscale'
  location: location
  properties: {
    enabled: true
    targetResourceUri: functionApp.id
    profiles: [
      {
        name: 'Auto scale based on CPU and Queue'
        capacity: {
          minimum: '2'
          maximum: '10'
          default: '2'
        }
        rules: [
          {
            metricTrigger: {
              metricName: 'CpuPercentage'
              metricResourceUri: functionApp.id
              timeGrain: 'PT1M'
              statistic: 'Average'
              timeWindow: 'PT5M'
              timeAggregation: 'Average'
              operator: 'GreaterThan'
              threshold: 75
            }
            scaleAction: {
              direction: 'Increase'
              type: 'ChangeCount'
              value: '1'
              cooldown: 'PT3M'
            }
          }
          {
            metricTrigger: {
              metricName: 'CpuPercentage'
              metricResourceUri: functionApp.id
              timeGrain: 'PT1M'
              statistic: 'Average'
              timeWindow: 'PT10M'
              timeAggregation: 'Average'
              operator: 'LessThan'
              threshold: 30
            }
            scaleAction: {
              direction: 'Decrease'
              type: 'ChangeCount'
              value: '1'
              cooldown: 'PT5M'
            }
          }
        ]
      }
    ]
  }
}
```

### Durable Functions Optimization

**Batch Processing**:
- Process outbound batches in parallel via sub-orchestrators
- Each sub-orchestrator handles 1 partner + transaction + response type
- Max 50 parallel sub-orchestrators (limited by Function App max instances)

**Replay Minimization**:
- Use `context.CurrentUtcDateTime` instead of `DateTime.UtcNow` (deterministic)
- Avoid side effects in orchestrator function (all I/O in activity functions)
- Use `context.CallActivityAsync` for all external operations

**Checkpoint Optimization**:
- Durable Functions automatically checkpoints after each activity function
- Failed activities retry without replaying entire orchestration
- Stateful workflow survives Function App restarts

---

## Monitoring & Observability

### KQL Queries

#### Control Number Acquisition Latency

```kql
dependencies
| where name == "AcquireControlNumbers"
| where timestamp > ago(24h)
| extend PartnerCode = tostring(customDimensions.PartnerCode)
| extend TransactionType = tostring(customDimensions.TransactionType)
| summarize
    p50 = percentile(duration, 50),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99),
    count = count()
    by bin(timestamp, 5m), PartnerCode
| project timestamp, PartnerCode, p50, p95, p99, count
| render timechart
```

#### Control Number Retry Rate

```kql
traces
| where message contains "Concurrency collision"
| where timestamp > ago(1h)
| extend PartnerCode = tostring(customDimensions.PartnerCode)
| extend RetryCount = toint(customDimensions.RetryCount)
| summarize
    TotalCollisions = count(),
    AvgRetryCount = avg(RetryCount),
    MaxRetryCount = max(RetryCount)
    by bin(timestamp, 5m), PartnerCode
| extend RetryRate = (AvgRetryCount / 5.0) * 100  // % requiring retry
| project timestamp, PartnerCode, RetryRate, TotalCollisions, AvgRetryCount
| render timechart
```

#### Outbound Assembly Success Rate

```kql
customEvents
| where name == "OutboundAssemblyCompleted"
| where timestamp > ago(24h)
| extend Status = tostring(customDimensions.Status)
| extend PartnerCode = tostring(customDimensions.PartnerCode)
| summarize
    SuccessCount = countif(Status == "SUCCESS"),
    FailureCount = countif(Status == "FAILED"),
    TotalCount = count()
    by bin(timestamp, 1h)
| extend SuccessRate = (SuccessCount * 100.0 / TotalCount)
| project timestamp, SuccessRate, SuccessCount, FailureCount, TotalCount
| render timechart
```

#### Control Number Gap Detection

```kql
// SQL query executed via Log Analytics connector
let GapQuery = ```
SELECT
    PartnerCode,
    TransactionType,
    CounterType,
    GapStart,
    GapEnd,
    GapSize
FROM dbo.ControlNumberGaps
WHERE GapSize > 0
```;
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SQL"
| where Category == "QueryStoreRuntimeStatistics"
| where query_hash_s == hash(GapQuery)
| summarize GapCount = count() by PartnerCode = tostring(customDimensions.PartnerCode)
| where GapCount > 0
| project PartnerCode, GapCount
```

#### Outbound File Creation Rate

```kql
AzureStorageBlobLogs
| where OperationName == "PutBlob"
| where Uri contains "outbound/"
| where TimeGenerated > ago(24h)
| extend PartnerCode = extract(@"partner=([^/]+)", 1, Uri)
| summarize FileCount = count() by bin(TimeGenerated, 5m), PartnerCode
| render timechart
```

#### Service Bus Publish Latency

```kql
dependencies
| where type == "Azure Service Bus"
| where name == "edi-outbound-ready"
| where timestamp > ago(1h)
| summarize
    p50 = percentile(duration, 50),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99)
    by bin(timestamp, 5m)
| project timestamp, p50, p95, p99
| render timechart
```

#### Control Number Rollover Alert

```kql
let RolloverThreshold = 0.90;
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SQL"
| where Category == "SQLSecurityAuditEvents"
| where statement_s contains "ControlNumberCounters"
| extend CurrentValue = tolong(extract(@"CurrentValue=(\d+)", 1, statement_s))
| extend MaxValue = tolong(extract(@"MaxValue=(\d+)", 1, statement_s))
| extend PartnerCode = tostring(extract(@"PartnerCode='([^']+)'", 1, statement_s))
| extend PercentUsed = (CurrentValue * 100.0 / MaxValue)
| where PercentUsed > (RolloverThreshold * 100)
| project TimeGenerated, PartnerCode, CurrentValue, MaxValue, PercentUsed
| order by PercentUsed desc
```

### Alert Rules

#### Critical Gap Detected

```bicep
resource criticalGapAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'alert-control-number-critical-gap'
  location: 'global'
  properties: {
    severity: 1  // Critical
    enabled: true
    scopes: [
      logAnalyticsWorkspace.id
    ]
    evaluationFrequency: 'PT15M'
    windowSize: 'PT15M'
    criteria: {
      allOf: [
        {
          query: '''
            AzureDiagnostics
            | where ResourceProvider == "MICROSOFT.SQL"
            | where Category == "QueryStoreRuntimeStatistics"
            | where query_sql_text_s contains "ControlNumberGaps"
            | where customDimensions.GapSize > 5
            | summarize GapCount = count()
          '''
          timeAggregation: 'Total'
          operator: 'GreaterThan'
          threshold: 0
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

#### Rollover Warning

```bicep
resource rolloverWarningAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'alert-control-number-rollover-warning'
  location: 'global'
  properties: {
    severity: 2  // Warning
    enabled: true
    scopes: [
      sqlDatabase.id
    ]
    evaluationFrequency: 'PT1H'
    windowSize: 'PT1H'
    criteria: {
      allOf: [
        {
          query: '''
            AzureDiagnostics
            | where CurrentValue > (MaxValue * 0.90)
            | summarize PartnerCount = dcount(PartnerCode)
          '''
          timeAggregation: 'Total'
          operator: 'GreaterThan'
          threshold: 0
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

#### High Retry Rate

```bicep
resource highRetryRateAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'alert-control-number-high-retry-rate'
  location: 'global'
  properties: {
    severity: 2  // Warning
    enabled: true
    scopes: [
      functionApp.id
    ]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT15M'
    criteria: {
      allOf: [
        {
          metricName: 'RetryRate'
          metricNamespace: 'Microsoft.Insights/components'
          operator: 'GreaterThan'
          threshold: 10  // 10% retry rate
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

### Dashboard

**Outbound Delivery Dashboard** (`OutboundDeliveryDashboard.json`):

```json
{
  "properties": {
    "lenses": [
      {
        "order": 0,
        "parts": [
          {
            "position": { "x": 0, "y": 0, "rowSpan": 4, "colSpan": 6 },
            "metadata": {
              "type": "Extension/HubsExtension/PartType/MonitorChartPart",
              "inputs": [
                {
                  "name": "options",
                  "value": {
                    "chart": {
                      "metrics": [
                        {
                          "resourceMetadata": { "id": "{functionAppResourceId}" },
                          "name": "AcquireControlNumbers",
                          "aggregationType": 7,
                          "namespace": "microsoft.insights/components",
                          "metricVisualization": {
                            "displayName": "Control Number Acquisition Latency (p95)"
                          }
                        }
                      ],
                      "title": "Control Number Acquisition Latency",
                      "timespan": { "relative": { "duration": 86400000 } }
                    }
                  }
                }
              ]
            }
          },
          {
            "position": { "x": 6, "y": 0, "rowSpan": 4, "colSpan": 6 },
            "metadata": {
              "type": "Extension/HubsExtension/PartType/MonitorChartPart",
              "inputs": [
                {
                  "name": "options",
                  "value": {
                    "chart": {
                      "metrics": [
                        {
                          "resourceMetadata": { "id": "{functionAppResourceId}" },
                          "name": "OutboundAssemblySuccessRate",
                          "aggregationType": 4,
                          "namespace": "microsoft.insights/components"
                        }
                      ],
                      "title": "Outbound Assembly Success Rate"
                    }
                  }
                }
              ]
            }
          },
          {
            "position": { "x": 0, "y": 4, "rowSpan": 4, "colSpan": 6 },
            "metadata": {
              "type": "Extension/HubsExtension/PartType/MonitorChartPart",
              "inputs": [
                {
                  "name": "options",
                  "value": {
                    "chart": {
                      "query": "AzureStorageBlobLogs | where OperationName == 'PutBlob' | summarize count() by PartnerCode",
                      "chartType": 2,
                      "title": "Files Created by Partner (Last 24h)"
                    }
                  }
                }
              ]
            }
          },
          {
            "position": { "x": 6, "y": 4, "rowSpan": 4, "colSpan": 6 },
            "metadata": {
              "type": "Extension/HubsExtension/PartType/MonitorChartPart",
              "inputs": [
                {
                  "name": "options",
                  "value": {
                    "chart": {
                      "query": "AzureDiagnostics | where CurrentValue > 0 | extend PercentUsed = (CurrentValue * 100.0 / MaxValue) | project PartnerCode, PercentUsed",
                      "chartType": 2,
                      "title": "Control Number Usage by Partner"
                    }
                  }
                }
              ]
            }
          },
          {
            "position": { "x": 0, "y": 8, "rowSpan": 4, "colSpan": 12 },
            "metadata": {
              "type": "Extension/HubsExtension/PartType/MonitorChartPart",
              "inputs": [
                {
                  "name": "options",
                  "value": {
                    "chart": {
                      "query": "AzureDiagnostics | where GapSize > 0 | summarize count() by PartnerCode",
                      "chartType": 3,
                      "title": "Control Number Gaps by Partner"
                    }
                  }
                }
              ]
            }
          }
        ]
      }
    ]
  }
}
```

---

## Troubleshooting

### Diagnostic PowerShell Scripts

**Check Control Number Store Health**:

```powershell
# Get connection string from Key Vault
$connectionString = az keyvault secret show `
    --vault-name kv-edi-prod `
    --name sql-controlnumbers-connection-string `
    --query value `
    --output tsv

# Test SQL connectivity
Test-NetConnection -ComputerName sql-edi-controlnumbers-prod.database.windows.net -Port 1433

# Query current control number status
$query = @"
SELECT
    PartnerCode,
    TransactionType,
    CounterType,
    CurrentValue,
    MaxValue,
    CAST((CurrentValue * 100.0 / MaxValue) AS DECIMAL(5,2)) AS PercentUsed,
    LastIncrementUtc,
    DATEDIFF(MINUTE, LastIncrementUtc, GETUTCDATE()) AS MinutesSinceLastIncrement
FROM dbo.ControlNumberCounters
ORDER BY PercentUsed DESC;
"@

Invoke-Sqlcmd -ConnectionString $connectionString -Query $query | Format-Table
```

**Detect and Report Gaps**:

```powershell
# Execute gap detection stored procedure
$gapQuery = @"
EXEC dbo.usp_DetectControlNumberGaps
    @PartnerCode = NULL,
    @DaysToCheck = 7;
"@

$gaps = Invoke-Sqlcmd -ConnectionString $connectionString -Query $gapQuery

if ($gaps.Count -gt 0) {
    Write-Host "⚠️ Control number gaps detected:" -ForegroundColor Yellow
    $gaps | Format-Table -AutoSize
    
    # Export to CSV
    $gaps | Export-Csv -Path "ControlNumberGaps_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv" -NoTypeInformation
    
    Write-Host "Gap report exported to CSV" -ForegroundColor Green
} else {
    Write-Host "✓ No control number gaps detected" -ForegroundColor Green
}
```

**Verify Outbound File Delivery**:

```powershell
# Query blobs with pending delivery status
$storageAccount = "stedioutboundprod"
$container = "outbound"

$pendingBlobs = az storage blob list `
    --account-name $storageAccount `
    --container-name $container `
    --query "[?metadata.DeliveryStatus=='PENDING'].{name:name,created:properties.creationTime,fileId:metadata.OutboundFileId,partner:metadata.PartnerCode}" `
    --output json | ConvertFrom-Json

if ($pendingBlobs.Count -gt 0) {
    Write-Host "⚠️ $($pendingBlobs.Count) files pending delivery:" -ForegroundColor Yellow
    $pendingBlobs | Format-Table -AutoSize
} else {
    Write-Host "✓ No files pending delivery" -ForegroundColor Green
}
```

**Force Control Number Rollover**:

```powershell
# Prompt for confirmation
$partnerCode = Read-Host "Enter PartnerCode"
$transactionType = Read-Host "Enter TransactionType (e.g., 271)"
$counterType = Read-Host "Enter CounterType (ISA, GS, or ST)"
$reason = Read-Host "Enter rollover reason"

$confirmation = Read-Host "Are you sure you want to reset control number for $partnerCode/$transactionType/$counterType? (yes/no)"

if ($confirmation -eq "yes") {
    $resetQuery = @"
EXEC dbo.usp_ResetControlNumber
    @PartnerCode = '$partnerCode',
    @TransactionType = '$transactionType',
    @CounterType = '$counterType',
    @NewValue = 1,
    @Reason = '$reason';
"@

    Invoke-Sqlcmd -ConnectionString $connectionString -Query $resetQuery
    
    Write-Host "✓ Control number reset to 1" -ForegroundColor Green
    
    # Verify reset
    $verifyQuery = @"
SELECT CurrentValue, LastIncrementUtc, ModifiedUtc
FROM dbo.ControlNumberCounters
WHERE PartnerCode = '$partnerCode'
  AND TransactionType = '$transactionType'
  AND CounterType = '$counterType';
"@
    
    Invoke-Sqlcmd -ConnectionString $connectionString -Query $verifyQuery | Format-Table
} else {
    Write-Host "Rollover cancelled" -ForegroundColor Yellow
}
```

---

**Document Complete**: 3 of 3 Parts  
**Total Sections**: 12 (Overview, Architecture, Control Number Store, Envelope Generation, Acknowledgment Types, Outbound Assembly Orchestration, Response File Management, Error Handling, Security & Compliance, Performance & Scalability, Monitoring & Observability, Troubleshooting)  
**Last Updated**: 2025-01-06
