# Database Migrations Quick Reference

## Architecture Overview

**Single Database, Multiple Schemas**

The EDI Platform uses a **single Azure SQL database** (`edi-platform`) with three logical schemas for separation of concerns:

- **[controlnumbers]** - X12 control number management
- **[eventstore]** - Event sourcing for 834 enrollment
- **[sftptracking]** - File transfer tracking and idempotency

**Benefits:**
- ✅ **Cost Reduction**: ~$800/month savings vs. 3 separate databases
- ✅ **Simplified Management**: Single connection, backup, and security model
- ✅ **Cross-Schema Queries**: Easy joins across schemas when needed
- ✅ **Reduced Latency**: No cross-database network hops

## Project Locations

| Schema | Project Path | Status |
|--------|--------------|--------|
| controlnumbers | `edi-database-controlnumbers/EDI.ControlNumbers.Migrations/` | ✅ Ready |
| eventstore | `edi-database-eventstore/EDI.EventStore.Migrations/` | ✅ Ready |
| sftptracking | `edi-database-sftptracking/EDI.SftpTracking.Migrations/` | ✅ Ready |

## Common Commands

### Install EF Core Tools (One Time)

```powershell
dotnet tool install --global dotnet-ef
```

### Create Initial Migration

```powershell
# Navigate to project directory first
cd edi-database-controlnumbers/EDI.ControlNumbers.Migrations
dotnet ef migrations add InitialCreate

cd ../../edi-database-eventstore/EDI.EventStore.Migrations
dotnet ef migrations add InitialCreate

cd ../../edi-database-sftptracking/EDI.SftpTracking.Migrations
dotnet ef migrations add InitialCreate
```

### Apply Migrations

```powershell
# Using LocalDB (default)
dotnet ef database update

# With specific connection string
dotnet ef database update --connection "Server=myserver;Database=EDIControlNumbers;User Id=user;Password=pass;"
```

### Generate SQL Scripts

```powershell
# Generate migration script
dotnet ef migrations script -o migration.sql

# Generate idempotent script (safe to run multiple times)
dotnet ef migrations script -i -o migration-idempotent.sql

# Generate script for specific range
dotnet ef migrations script FromMigration ToMigration -o update.sql
```

### View Migration Status

```powershell
# List all migrations
dotnet ef migrations list

# Check if database exists
dotnet ef database drop --dry-run
```

## Connection Strings

### Development (LocalDB)

```json
{
  "ConnectionStrings": {
    "EdiPlatform": "Server=(localdb)\\mssqllocaldb;Database=edi-platform;Trusted_Connection=True;TrustServerCertificate=True;"
  }
}
```

**Note**: All three DbContexts use the same connection string. Schema separation is configured in the `OnModelCreating()` method.

### Azure SQL Database (Production)

```json
{
  "ConnectionStrings": {
    "EdiPlatform": "Server=tcp:sql-edi-prod.database.windows.net,1433;Database=edi-platform;Authentication=Active Directory Default;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
  }
}
```

### Azure SQL Database (Managed Identity - Recommended)

```json
{
  "ConnectionStrings": {
    "EdiPlatform": "Server=tcp:sql-edi-prod.database.windows.net,1433;Database=edi-platform;Authentication=Active Directory Managed Identity;Encrypt=True;"
  }
}
```

## Integration in Azure Functions

### Program.cs

```csharp
using Microsoft.EntityFrameworkCore;
using EDI.ControlNumbers.Migrations;
using EDI.EventStore.Migrations;
using EDI.SftpTracking.Migrations;

var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults()
    .ConfigureServices((context, services) =>
    {
        // Single connection string for all DbContexts
        var connectionString = Environment.GetEnvironmentVariable("EdiPlatformDb");
        
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

### local.settings.json

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

## Schema Configuration

### DbContext Setup

Each DbContext configures its default schema in `OnModelCreating()`:

**ControlNumbersDbContext.cs**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.HasDefaultSchema("controlnumbers");
    
    modelBuilder.Entity<ControlNumberCounter>(entity =>
    {
        entity.ToTable("ControlNumberCounters", "controlnumbers");
        // ... configuration
    });
}
```

**EventStoreDbContext.cs**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.HasDefaultSchema("eventstore");
    
    modelBuilder.Entity<DomainEvent>(entity =>
    {
        entity.ToTable("DomainEvent", "eventstore");
        // ... configuration
    });
}
```

**SftpTrackingDbContext.cs**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.HasDefaultSchema("sftptracking");
    
    modelBuilder.Entity<FileTracking>(entity =>
    {
        entity.ToTable("FileTracking", "sftptracking");
        // ... configuration
    });
}
```

### Cross-Schema Queries

When needed, you can query across schemas:

```csharp
// Example: Join control numbers with file tracking
var query = from cn in _controlNumbersContext.ControlNumberCounters
            join ft in _sftpTrackingContext.FileTracking
            on cn.PartnerCode equals ft.PartnerCode
            where ft.Status == "Downloaded"
            select new { cn.PartnerCode, cn.CurrentValue, ft.FileName };
```

### Schema Permissions

```sql
-- Grant schema-level permissions to Function App managed identities
CREATE USER [func-834-processor] FROM EXTERNAL PROVIDER;
CREATE USER [func-sftp-connector] FROM EXTERNAL PROVIDER;
CREATE USER [func-acknowledgments] FROM EXTERNAL PROVIDER;

GRANT SELECT, INSERT, UPDATE ON SCHEMA::[eventstore] TO [func-834-processor];
GRANT SELECT, INSERT ON SCHEMA::[sftptracking] TO [func-sftp-connector];
GRANT SELECT, EXECUTE ON SCHEMA::[controlnumbers] TO [func-acknowledgments];
```

### Usage in Function

```csharp
using EDI.ControlNumbers.Migrations;
using EDI.ControlNumbers.Migrations.Entities;

public class MyFunction
{
    private readonly ControlNumbersDbContext _context;

    public MyFunction(ControlNumbersDbContext context)
    {
        _context = context;
    }

    [Function("GetNextControlNumber")]
    public async Task<HttpResponseData> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req)
    {
        var counter = await _context.ControlNumberCounters
            .FirstOrDefaultAsync(c => 
                c.PartnerCode == "PARTNERA" && 
                c.TransactionType == "834" && 
                c.CounterType == "ISA");

        if (counter == null)
        {
            counter = new ControlNumberCounter
            {
                PartnerCode = "PARTNERA",
                TransactionType = "834",
                CounterType = "ISA",
                CurrentValue = 1,
                LastIncrementUtc = DateTime.UtcNow,
                CreatedUtc = DateTime.UtcNow,
                ModifiedUtc = DateTime.UtcNow
            };
            _context.ControlNumberCounters.Add(counter);
        }
        else
        {
            counter.CurrentValue++;
            counter.ModifiedUtc = DateTime.UtcNow;
        }

        await _context.SaveChangesAsync();

        var response = req.CreateResponse(HttpStatusCode.OK);
        await response.WriteAsJsonAsync(new { ControlNumber = counter.CurrentValue });
        return response;
    }
}
```

## Troubleshooting

### "No DbContext was found"

Make sure you're in the correct project directory:
```powershell
cd edi-database-controlnumbers/EDI.ControlNumbers.Migrations
```

### "Unable to create an object of type 'XxxDbContext'"

Add a parameterless constructor to your DbContext (already included in all contexts).

### "The migration 'xxx' has already been applied to the database"

This is normal. Use `dotnet ef database update` to update to latest.

### "A network-related or instance-specific error"

- Check SQL Server is running
- Verify connection string
- For LocalDB: `sqllocaldb start mssqllocaldb`

### Performance Issues

Add indexes via migrations:
```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.CreateIndex(
        name: "IX_TableName_ColumnName",
        table: "TableName",
        column: "ColumnName");
}
```

## Best Practices

1. **Always create migrations** before modifying entities
2. **Name migrations descriptively**: `Add_PartnerCode_Index` not `Update1`
3. **Review generated SQL** before applying to production
4. **Use idempotent scripts** for deployment pipelines
5. **Test migrations** on development database first
6. **Keep migrations small** - one logical change per migration
7. **Never modify applied migrations** - create new ones instead
8. **Backup production** before running migrations

## CI/CD Integration

### GitHub Actions Example

```yaml
name: Database Migration

on:
  push:
    branches: [ main ]
    paths:
      - 'edi-database-*/**'

jobs:
  deploy-migrations:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '9.0.x'
      
      - name: Install EF Core Tools
        run: dotnet tool install --global dotnet-ef
      
      - name: Generate Migration Scripts
        run: |
          # Control Numbers Schema
          cd edi-database-controlnumbers/EDI.ControlNumbers.Migrations
          dotnet ef migrations script -i -o ../../sql/01-controlnumbers-schema.sql
          
          # Event Store Schema
          cd ../../edi-database-eventstore/EDI.EventStore.Migrations
          dotnet ef migrations script -i -o ../../sql/02-eventstore-schema.sql
          
          # SFTP Tracking Schema
          cd ../../edi-database-sftptracking/EDI.SftpTracking.Migrations
          dotnet ef migrations script -i -o ../../sql/03-sftptracking-schema.sql
      
      - name: Combine Scripts
        run: |
          cat sql/01-controlnumbers-schema.sql > sql/edi-platform-migration.sql
          echo "GO" >> sql/edi-platform-migration.sql
          cat sql/02-eventstore-schema.sql >> sql/edi-platform-migration.sql
          echo "GO" >> sql/edi-platform-migration.sql
          cat sql/03-sftptracking-schema.sql >> sql/edi-platform-migration.sql
      
      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      - name: Deploy to Azure SQL
        run: |
          sqlcmd -S sql-edi-prod.database.windows.net \
                 -d edi-platform \
                 -G \
                 -i sql/edi-platform-migration.sql
      
      - name: Upload Scripts (Artifact)
        uses: actions/upload-artifact@v3
        with:
          name: migration-scripts
          path: sql/*.sql
```

### Deployment Order

1. **Create Database**: `CREATE DATABASE [edi-platform]`
2. **Create Schemas**: 
   ```sql
   CREATE SCHEMA [controlnumbers];
   CREATE SCHEMA [eventstore];
   CREATE SCHEMA [sftptracking];
   ```
3. **Deploy Migrations**: Run EF Core migrations for each schema
4. **Configure Permissions**: Grant schema-level access to managed identities

## Additional Resources

- [EF Core Migrations Overview](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/)
- [EF Core CLI Tools](https://learn.microsoft.com/en-us/ef/core/cli/dotnet)
- Each project's README.md for specific details

---

**Last Updated**: October 6, 2025  
**Version**: 1.0  
**Maintainer**: EDI Platform Team
