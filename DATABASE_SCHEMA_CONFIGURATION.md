# Database Schema Configuration Guide

**Document Version:** 1.0  
**Last Updated:** October 6, 2025  
**Status:** Implementation Guide

---

## Overview

This document provides implementation guidance for configuring the three EDI Platform database schemas (`controlnumbers`, `eventstore`, `sftptracking`) within a single Azure SQL database.

**Database Name:** `edi-platform`

---

## Table of Contents

1. [Schema Architecture](#schema-architecture)
2. [DbContext Configuration](#dbcontext-configuration)
3. [Connection String Setup](#connection-string-setup)
4. [Migration Commands](#migration-commands)
5. [Security Configuration](#security-configuration)
6. [Cross-Schema Queries](#cross-schema-queries)

---

## Schema Architecture

### Logical Separation

```
edi-platform (Database)
│
├── [controlnumbers] Schema
│   ├── ControlNumberCounters (Table)
│   ├── ControlNumberAudit (Table)
│   └── usp_GetNextControlNumber (Stored Procedure)
│
├── [eventstore] Schema
│   ├── DomainEvent (Table)
│   ├── TransactionBatch (Table)
│   ├── Member (Table)
│   ├── Enrollment (Table)
│   └── EventSnapshot (Table)
│
└── [sftptracking] Schema
    └── FileTracking (Table)
```

### Benefits

| Benefit | Description |
|---------|-------------|
| **Cost Efficiency** | Single database reduces costs by ~$800/month vs. 3 separate databases |
| **Simplified Management** | One connection string, single backup/restore operation |
| **Schema Isolation** | Logical separation maintains clean boundaries between components |
| **Security Granularity** | Schema-level permissions provide fine-grained access control |
| **Performance** | No cross-database network hops for queries |
| **Operational Simplicity** | Single managed identity, one private endpoint |

---

## DbContext Configuration

### 1. ControlNumbersDbContext

**File:** `edi-database-controlnumbers/EDI.ControlNumbers.Migrations/Data/ControlNumbersDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using EDI.ControlNumbers.Migrations.Entities;

namespace EDI.ControlNumbers.Migrations.Data;

public class ControlNumbersDbContext : DbContext
{
    public DbSet<ControlNumberCounter> ControlNumberCounters { get; set; } = null!;
    public DbSet<ControlNumberAudit> ControlNumberAudit { get; set; } = null!;

    public ControlNumbersDbContext(DbContextOptions<ControlNumbersDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Set default schema for all entities
        modelBuilder.HasDefaultSchema("controlnumbers");

        // Configure ControlNumberCounter entity
        modelBuilder.Entity<ControlNumberCounter>(entity =>
        {
            entity.ToTable("ControlNumberCounters", "controlnumbers");
            entity.HasKey(e => e.CounterId);

            entity.Property(e => e.PartnerCode).IsRequired().HasMaxLength(20);
            entity.Property(e => e.TransactionType).IsRequired().HasMaxLength(10);
            entity.Property(e => e.CounterType).IsRequired().HasMaxLength(20);
            entity.Property(e => e.CurrentValue).IsRequired();
            entity.Property(e => e.LastIncrementUtc).IsRequired().HasColumnType("datetime2");
            entity.Property(e => e.CreatedUtc).IsRequired().HasColumnType("datetime2");
            entity.Property(e => e.ModifiedUtc).IsRequired().HasColumnType("datetime2");

            // Unique constraint for partner + transaction + counter type
            entity.HasIndex(e => new { e.PartnerCode, e.TransactionType, e.CounterType })
                .IsUnique()
                .HasDatabaseName("UQ_ControlNumber_Partner_Transaction_Type");

            // Index for partner lookups
            entity.HasIndex(e => e.PartnerCode)
                .HasDatabaseName("IX_ControlNumberCounters_PartnerCode");
        });

        // Configure ControlNumberAudit entity
        modelBuilder.Entity<ControlNumberAudit>(entity =>
        {
            entity.ToTable("ControlNumberAudit", "controlnumbers");
            entity.HasKey(e => e.AuditId);

            entity.Property(e => e.CounterId).IsRequired();
            entity.Property(e => e.PartnerCode).IsRequired().HasMaxLength(20);
            entity.Property(e => e.TransactionType).IsRequired().HasMaxLength(10);
            entity.Property(e => e.CounterType).IsRequired().HasMaxLength(20);
            entity.Property(e => e.AssignedValue).IsRequired();
            entity.Property(e => e.AssignedUtc).IsRequired().HasColumnType("datetime2");

            // Foreign key to ControlNumberCounters
            entity.HasOne<ControlNumberCounter>()
                .WithMany()
                .HasForeignKey(e => e.CounterId)
                .OnDelete(DeleteBehavior.Restrict);

            // Index for date range queries
            entity.HasIndex(e => e.AssignedUtc)
                .HasDatabaseName("IX_ControlNumberAudit_AssignedUtc");

            // Index for partner audit trail
            entity.HasIndex(e => new { e.PartnerCode, e.AssignedUtc })
                .HasDatabaseName("IX_ControlNumberAudit_Partner_Date");
        });
    }
}
```

### 2. EventStoreDbContext

**File:** `edi-database-eventstore/EDI.EventStore.Migrations/Data/EventStoreDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using EDI.EventStore.Migrations.Entities;

namespace EDI.EventStore.Migrations.Data;

public class EventStoreDbContext : DbContext
{
    public DbSet<DomainEvent> DomainEvents { get; set; } = null!;
    public DbSet<TransactionBatch> TransactionBatches { get; set; } = null!;
    public DbSet<Member> Members { get; set; } = null!;
    public DbSet<Enrollment> Enrollments { get; set; } = null!;
    public DbSet<EventSnapshot> EventSnapshots { get; set; } = null!;

    public EventStoreDbContext(DbContextOptions<EventStoreDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Set default schema for all entities
        modelBuilder.HasDefaultSchema("eventstore");

        // Configure DomainEvent entity (append-only)
        modelBuilder.Entity<DomainEvent>(entity =>
        {
            entity.ToTable("DomainEvent", "eventstore");
            entity.HasKey(e => e.EventId);

            entity.Property(e => e.EventGuid).IsRequired();
            entity.Property(e => e.AggregateType).IsRequired().HasMaxLength(100);
            entity.Property(e => e.AggregateId).IsRequired().HasMaxLength(100);
            entity.Property(e => e.EventType).IsRequired().HasMaxLength(100);
            entity.Property(e => e.EventVersion).IsRequired();
            entity.Property(e => e.EventSequence).IsRequired();
            entity.Property(e => e.EventTimestamp).IsRequired().HasColumnType("datetime2");
            entity.Property(e => e.EventData).IsRequired().HasColumnType("nvarchar(max)");
            entity.Property(e => e.EventMetadata).HasColumnType("nvarchar(max)");

            // Unique constraint on EventGuid (idempotency)
            entity.HasIndex(e => e.EventGuid)
                .IsUnique()
                .HasDatabaseName("UQ_DomainEvent_EventGuid");

            // Index for aggregate queries
            entity.HasIndex(e => new { e.AggregateType, e.AggregateId, e.EventSequence })
                .HasDatabaseName("IX_DomainEvent_Aggregate_Sequence");

            // Index for sequential event replay
            entity.HasIndex(e => e.EventSequence)
                .HasDatabaseName("IX_DomainEvent_EventSequence");

            // Index for event type queries
            entity.HasIndex(e => e.EventType)
                .HasDatabaseName("IX_DomainEvent_EventType");
        });

        // Configure Member projection
        modelBuilder.Entity<Member>(entity =>
        {
            entity.ToTable("Member", "eventstore");
            entity.HasKey(e => e.MemberId);

            entity.Property(e => e.SubscriberId).IsRequired().HasMaxLength(50);
            entity.Property(e => e.FirstName).IsRequired().HasMaxLength(100);
            entity.Property(e => e.LastName).IsRequired().HasMaxLength(100);
            entity.Property(e => e.DateOfBirth).IsRequired().HasColumnType("date");
            entity.Property(e => e.Gender).HasMaxLength(1);
            entity.Property(e => e.LastEventSequence).IsRequired();
            entity.Property(e => e.ModifiedUtc).IsRequired().HasColumnType("datetime2");

            // Unique constraint on SubscriberId
            entity.HasIndex(e => e.SubscriberId)
                .IsUnique()
                .HasDatabaseName("UQ_Member_SubscriberId");
        });

        // Configure Enrollment projection
        modelBuilder.Entity<Enrollment>(entity =>
        {
            entity.ToTable("Enrollment", "eventstore");
            entity.HasKey(e => e.EnrollmentId);

            entity.Property(e => e.MemberId).IsRequired();
            entity.Property(e => e.PlanCode).IsRequired().HasMaxLength(50);
            entity.Property(e => e.EffectiveDate).IsRequired().HasColumnType("date");
            entity.Property(e => e.TerminationDate).HasColumnType("date");
            entity.Property(e => e.EnrollmentStatus).IsRequired().HasMaxLength(20);
            entity.Property(e => e.LastEventSequence).IsRequired();
            entity.Property(e => e.ModifiedUtc).IsRequired().HasColumnType("datetime2");

            // Foreign key to Member
            entity.HasOne<Member>()
                .WithMany()
                .HasForeignKey(e => e.MemberId)
                .OnDelete(DeleteBehavior.Restrict);

            // Index for member enrollments
            entity.HasIndex(e => e.MemberId)
                .HasDatabaseName("IX_Enrollment_MemberId");

            // Index for active enrollments
            entity.HasIndex(e => new { e.EnrollmentStatus, e.EffectiveDate })
                .HasDatabaseName("IX_Enrollment_Status_EffectiveDate");
        });

        // Configure TransactionBatch
        modelBuilder.Entity<TransactionBatch>(entity =>
        {
            entity.ToTable("TransactionBatch", "eventstore");
            entity.HasKey(e => e.BatchId);

            entity.Property(e => e.BatchGuid).IsRequired();
            entity.Property(e => e.PartnerCode).IsRequired().HasMaxLength(50);
            entity.Property(e => e.FileName).IsRequired().HasMaxLength(500);
            entity.Property(e => e.FileHash).IsRequired().HasMaxLength(100);
            entity.Property(e => e.ReceivedUtc).IsRequired().HasColumnType("datetime2");
            entity.Property(e => e.ProcessingStatus).IsRequired().HasMaxLength(20);

            // Unique constraint on BatchGuid
            entity.HasIndex(e => e.BatchGuid)
                .IsUnique()
                .HasDatabaseName("UQ_TransactionBatch_BatchGuid");

            // Index for partner queries
            entity.HasIndex(e => new { e.PartnerCode, e.ReceivedUtc })
                .HasDatabaseName("IX_TransactionBatch_Partner_Received");
        });

        // Configure EventSnapshot
        modelBuilder.Entity<EventSnapshot>(entity =>
        {
            entity.ToTable("EventSnapshot", "eventstore");
            entity.HasKey(e => e.SnapshotId);

            entity.Property(e => e.AggregateType).IsRequired().HasMaxLength(100);
            entity.Property(e => e.AggregateId).IsRequired().HasMaxLength(100);
            entity.Property(e => e.EventSequence).IsRequired();
            entity.Property(e => e.SnapshotData).IsRequired().HasColumnType("nvarchar(max)");
            entity.Property(e => e.CreatedUtc).IsRequired().HasColumnType("datetime2");

            // Index for snapshot retrieval
            entity.HasIndex(e => new { e.AggregateType, e.AggregateId, e.EventSequence })
                .HasDatabaseName("IX_EventSnapshot_Aggregate_Sequence");
        });
    }
}
```

### 3. SftpTrackingDbContext

**File:** `edi-database-sftptracking/EDI.SftpTracking.Migrations/Data/SftpTrackingDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using EDI.SftpTracking.Migrations.Entities;

namespace EDI.SftpTracking.Migrations.Data;

public class SftpTrackingDbContext : DbContext
{
    public DbSet<FileTracking> FileTracking { get; set; } = null!;

    public SftpTrackingDbContext(DbContextOptions<SftpTrackingDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Set default schema for all entities
        modelBuilder.HasDefaultSchema("sftptracking");

        modelBuilder.Entity<FileTracking>(entity =>
        {
            entity.ToTable("FileTracking", "sftptracking");
            entity.HasKey(e => e.Id);

            // Required fields
            entity.Property(e => e.PartnerCode).IsRequired().HasMaxLength(50);
            entity.Property(e => e.FileName).IsRequired().HasMaxLength(500);
            entity.Property(e => e.FileHash).IsRequired().HasMaxLength(100);
            entity.Property(e => e.FileSize).IsRequired();
            entity.Property(e => e.Direction).IsRequired().HasMaxLength(20);
            entity.Property(e => e.Status).IsRequired().HasMaxLength(20);
            entity.Property(e => e.ProcessedAt).IsRequired().HasColumnType("datetime2");

            // Optional fields
            entity.Property(e => e.BlobUrl).HasMaxLength(1000);
            entity.Property(e => e.ErrorMessage);
            entity.Property(e => e.CorrelationId).HasMaxLength(100);

            // Index for idempotency checks
            entity.HasIndex(e => new { e.PartnerCode, e.FileName, e.Direction, e.Status })
                .HasDatabaseName("IX_FileTracking_PartnerCode_FileName_Direction_Status")
                .IncludeProperties(e => e.ProcessedAt);

            // Index for date range queries
            entity.HasIndex(e => e.ProcessedAt)
                .HasDatabaseName("IX_FileTracking_ProcessedAt")
                .IsDescending()
                .IncludeProperties(e => new { e.PartnerCode, e.FileName, e.Direction, e.Status });

            // Filtered index for correlation tracking
            entity.HasIndex(e => e.CorrelationId)
                .HasDatabaseName("IX_FileTracking_CorrelationId")
                .HasFilter("[CorrelationId] IS NOT NULL");
        });
    }
}
```

---

## Connection String Setup

### Single Connection String Approach

All three DbContexts use the **same connection string** pointing to the `edi-platform` database. Schema separation is configured via `HasDefaultSchema()` in each DbContext.

### Development Environment (LocalDB)

**appsettings.Development.json**

```json
{
  "ConnectionStrings": {
    "EdiPlatform": "Server=(localdb)\\mssqllocaldb;Database=edi-platform;Trusted_Connection=True;TrustServerCertificate=True;"
  }
}
```

### Azure SQL Database (Production)

**appsettings.Production.json** (Managed Identity - Recommended)

```json
{
  "ConnectionStrings": {
    "EdiPlatform": "Server=tcp:sql-edi-prod.database.windows.net,1433;Database=edi-platform;Authentication=Active Directory Default;Encrypt=True;"
  }
}
```

**Alternative: Connection String with Managed Identity**

```json
{
  "ConnectionStrings": {
    "EdiPlatform": "Server=tcp:sql-edi-prod.database.windows.net,1433;Database=edi-platform;Authentication=Active Directory Managed Identity;Encrypt=True;"
  }
}
```

### Function App Configuration

**Program.cs**

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.EntityFrameworkCore;
using EDI.ControlNumbers.Migrations.Data;
using EDI.EventStore.Migrations.Data;
using EDI.SftpTracking.Migrations.Data;

var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults()
    .ConfigureServices((context, services) =>
    {
        // Single connection string for all DbContexts
        var connectionString = Environment.GetEnvironmentVariable("EdiPlatformDb")
            ?? throw new InvalidOperationException("EdiPlatformDb connection string not configured");
        
        // Register DbContexts - schema separation handled in OnModelCreating()
        services.AddDbContext<ControlNumbersDbContext>(options =>
            options.UseSqlServer(connectionString));
        
        services.AddDbContext<EventStoreDbContext>(options =>
            options.UseSqlServer(connectionString));
        
        services.AddDbContext<SftpTrackingDbContext>(options =>
            options.UseSqlServer(connectionString));
    })
    .Build();

await host.RunAsync();
```

**local.settings.json**

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "EdiPlatformDb": "Server=(localdb)\\mssqllocaldb;Database=edi-platform;Trusted_Connection=True;"
  }
}
```

---

## Migration Commands

### Initial Database Setup

```powershell
# Create the database (one time)
sqlcmd -S "(localdb)\mssqllocaldb" -Q "CREATE DATABASE [edi-platform]"

# Create schemas (one time)
sqlcmd -S "(localdb)\mssqllocaldb" -d "edi-platform" -Q "CREATE SCHEMA [controlnumbers]"
sqlcmd -S "(localdb)\mssqllocaldb" -d "edi-platform" -Q "CREATE SCHEMA [eventstore]"
sqlcmd -S "(localdb)\mssqllocaldb" -d "edi-platform" -Q "CREATE SCHEMA [sftptracking]"
```

### Create Migrations

```powershell
# Control Numbers schema
cd edi-database-controlnumbers/EDI.ControlNumbers.Migrations
dotnet ef migrations add InitialCreate --context ControlNumbersDbContext

# Event Store schema
cd ../../edi-database-eventstore/EDI.EventStore.Migrations
dotnet ef migrations add InitialCreate --context EventStoreDbContext

# SFTP Tracking schema
cd ../../edi-database-sftptracking/EDI.SftpTracking.Migrations
dotnet ef migrations add InitialCreate --context SftpTrackingDbContext
```

### Apply Migrations

```powershell
# Apply all migrations to the edi-platform database
cd edi-database-controlnumbers/EDI.ControlNumbers.Migrations
dotnet ef database update --context ControlNumbersDbContext

cd ../../edi-database-eventstore/EDI.EventStore.Migrations
dotnet ef database update --context EventStoreDbContext

cd ../../edi-database-sftptracking/EDI.SftpTracking.Migrations
dotnet ef database update --context SftpTrackingDbContext
```

### Generate SQL Scripts for Deployment

```powershell
# Generate idempotent scripts for each schema
cd edi-database-controlnumbers/EDI.ControlNumbers.Migrations
dotnet ef migrations script -i -o ../../sql/01-controlnumbers-schema.sql --context ControlNumbersDbContext

cd ../../edi-database-eventstore/EDI.EventStore.Migrations
dotnet ef migrations script -i -o ../../sql/02-eventstore-schema.sql --context EventStoreDbContext

cd ../../edi-database-sftptracking/EDI.SftpTracking.Migrations
dotnet ef migrations script -i -o ../../sql/03-sftptracking-schema.sql --context SftpTrackingDbContext

# Combine into single deployment script
cat sql/01-controlnumbers-schema.sql > sql/edi-platform-deployment.sql
echo "GO" >> sql/edi-platform-deployment.sql
cat sql/02-eventstore-schema.sql >> sql/edi-platform-deployment.sql
echo "GO" >> sql/edi-platform-deployment.sql
cat sql/03-sftptracking-schema.sql >> sql/edi-platform-deployment.sql
```

---

## Security Configuration

### Managed Identity Setup

**Create User-Assigned Managed Identity:**

```powershell
az identity create `
  --name "mi-edi-functions-prod" `
  --resource-group "rg-edi-prod"
```

**Create Database Users for Managed Identities:**

```sql
-- Create database users from Azure AD identities
CREATE USER [func-834-processor] FROM EXTERNAL PROVIDER;
CREATE USER [func-sftp-connector] FROM EXTERNAL PROVIDER;
CREATE USER [func-acknowledgments] FROM EXTERNAL PROVIDER;
CREATE USER [func-projector] FROM EXTERNAL PROVIDER;
```

### Schema-Level Permissions

**Grant permissions at the schema level for fine-grained access control:**

```sql
-- Control Numbers Schema: Read-only for most functions
GRANT SELECT ON SCHEMA::[controlnumbers] TO [func-834-processor];
GRANT SELECT ON SCHEMA::[controlnumbers] TO [func-sftp-connector];

-- Control Numbers Schema: Full access for acknowledgment generator
GRANT SELECT, INSERT, UPDATE, EXECUTE ON SCHEMA::[controlnumbers] TO [func-acknowledgments];

-- Event Store Schema: Full access for 834 processor and projector
GRANT SELECT, INSERT, UPDATE ON SCHEMA::[eventstore] TO [func-834-processor];
GRANT SELECT, INSERT, UPDATE ON SCHEMA::[eventstore] TO [func-projector];

-- SFTP Tracking Schema: Full access for SFTP connector
GRANT SELECT, INSERT ON SCHEMA::[sftptracking] TO [func-sftp-connector];

-- SFTP Tracking Schema: Read-only for monitoring
GRANT SELECT ON SCHEMA::[sftptracking] TO [func-834-processor];
```

### Row-Level Security (Optional - Future)

For multi-tenant scenarios, implement Row-Level Security:

```sql
-- Example: Restrict access to specific partner data
CREATE FUNCTION [controlnumbers].fn_SecurityPredicate(@PartnerCode VARCHAR(20))
    RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS result
    WHERE @PartnerCode = CAST(SESSION_CONTEXT(N'PartnerCode') AS VARCHAR(20));
GO

CREATE SECURITY POLICY [controlnumbers].PartnerFilter
    ADD FILTER PREDICATE [controlnumbers].fn_SecurityPredicate(PartnerCode)
    ON [controlnumbers].ControlNumberCounters,
    ADD BLOCK PREDICATE [controlnumbers].fn_SecurityPredicate(PartnerCode)
    ON [controlnumbers].ControlNumberCounters
WITH (STATE = ON);
```

---

## Cross-Schema Queries

### Example: Join Control Numbers with File Tracking

```csharp
// Example: Find all control numbers assigned for files downloaded today
using var controlNumbersContext = serviceProvider.GetRequiredService<ControlNumbersDbContext>();
using var sftpTrackingContext = serviceProvider.GetRequiredService<SftpTrackingDbContext>();

var today = DateTime.UtcNow.Date;

var query = from ft in sftpTrackingContext.FileTracking
            where ft.ProcessedAt >= today && ft.Status == "Downloaded"
            join cn in controlNumbersContext.ControlNumberCounters
            on ft.PartnerCode equals cn.PartnerCode
            select new
            {
                ft.PartnerCode,
                ft.FileName,
                ft.ProcessedAt,
                cn.CounterType,
                cn.CurrentValue
            };

var results = await query.ToListAsync();
```

### Example: Event Store to SFTP Tracking Correlation

```csharp
// Example: Find all 834 events for files processed by SFTP connector
using var eventStoreContext = serviceProvider.GetRequiredService<EventStoreDbContext>();
using var sftpTrackingContext = serviceProvider.GetRequiredService<SftpTrackingDbContext>();

var query = from batch in eventStoreContext.TransactionBatches
            join ft in sftpTrackingContext.FileTracking
            on batch.FileName equals ft.FileName
            where ft.Direction == "Inbound" && ft.Status == "Downloaded"
            select new
            {
                batch.BatchGuid,
                batch.PartnerCode,
                batch.FileName,
                batch.ProcessingStatus,
                ft.ProcessedAt,
                ft.BlobUrl
            };

var results = await query.ToListAsync();
```

### Raw SQL Queries (When Needed)

```csharp
// Example: Complex cross-schema query using raw SQL
using var connection = new SqlConnection(connectionString);
await connection.OpenAsync();

var sql = @"
    SELECT 
        cn.PartnerCode,
        cn.CounterType,
        cn.CurrentValue,
        COUNT(ft.Id) AS FilesProcessed,
        SUM(ft.FileSize) AS TotalBytes
    FROM [controlnumbers].ControlNumberCounters cn
    INNER JOIN [sftptracking].FileTracking ft ON cn.PartnerCode = ft.PartnerCode
    WHERE ft.ProcessedAt >= @StartDate
    GROUP BY cn.PartnerCode, cn.CounterType, cn.CurrentValue
    ORDER BY FilesProcessed DESC;
";

using var command = new SqlCommand(sql, connection);
command.Parameters.AddWithValue("@StartDate", DateTime.UtcNow.AddDays(-7));

using var reader = await command.ExecuteReaderAsync();
while (await reader.ReadAsync())
{
    var partnerCode = reader.GetString(0);
    var counterType = reader.GetString(1);
    var currentValue = reader.GetInt64(2);
    var filesProcessed = reader.GetInt32(3);
    var totalBytes = reader.GetInt64(4);
    
    Console.WriteLine($"{partnerCode}: {filesProcessed} files, {totalBytes} bytes");
}
```

---

## Best Practices

### 1. Schema Naming Conventions

- Use lowercase for schema names: `controlnumbers`, `eventstore`, `sftptracking`
- Use PascalCase for table names: `ControlNumberCounters`, `DomainEvent`
- Use PascalCase for column names: `PartnerCode`, `EventTimestamp`

### 2. Migration Management

- Create separate migration projects for each schema
- Generate idempotent scripts for CI/CD deployment
- Test migrations on a copy of production data before deploying
- Always generate SQL scripts for production deployments (never use `dotnet ef database update` in production)

### 3. Connection Pooling

```csharp
// Configure connection pooling in connection string
var connectionString = "Server=...;Database=edi-platform;...;Max Pool Size=200;Min Pool Size=10;";
```

### 4. Performance Monitoring

```sql
-- Monitor query performance by schema
SELECT 
    SCHEMA_NAME(o.schema_id) AS SchemaName,
    OBJECT_NAME(s.object_id) AS TableName,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.objects o ON s.object_id = o.object_id
WHERE s.database_id = DB_ID('edi-platform')
ORDER BY s.user_updates DESC;
```

### 5. Backup Strategy

```powershell
# Single database backup includes all schemas
az sql db create `
  --resource-group "rg-edi-prod" `
  --server "sql-edi-prod" `
  --name "edi-platform-backup-$(Get-Date -Format 'yyyyMMdd')" `
  --source-database "edi-platform" `
  --service-objective "GP_Gen5_2"
```

---

## Summary

The single-database, multi-schema approach provides:

✅ **Cost Efficiency**: ~$800/month savings vs. three separate databases  
✅ **Simplified Operations**: Single connection string, backup, and security model  
✅ **Schema Isolation**: Logical separation maintains clean boundaries  
✅ **Performance**: No cross-database network hops  
✅ **Flexibility**: Easy cross-schema queries when needed  
✅ **Security**: Fine-grained schema-level permissions  

**Next Steps:**

1. Update existing DbContext classes with schema configuration
2. Create initial migrations for each schema
3. Deploy to development environment and test
4. Generate SQL scripts for production deployment
5. Configure managed identity permissions

---

**Document Version:** 1.0  
**Last Updated:** October 6, 2025  
**Maintainer:** EDI Platform Team
