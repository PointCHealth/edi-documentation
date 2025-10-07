# SFTP Connector - Implementation Plan

**Ready-to-Copy Code Templates**  
**Date:** October 7, 2025

---

## Implementation Schedule

### Day 1: HealthCheckFunction (3 hours)

**Morning (1.5 hours):**
- ✅ 9:00-10:00 AM: Create HealthCheckService.cs
- ✅ 10:00-10:30 AM: Create HealthCheckFunction.cs
- ✅ 10:30-11:00 AM: Create models (HealthStatus, ComponentHealth, HealthMetrics)

**Afternoon (1.5 hours):**
- ✅ 1:00-1:30 PM: Register services in DI container
- ✅ 1:30-2:30 PM: Write unit tests
- ✅ 2:30-3:00 PM: Test locally

### Day 2: ConfigurationUpdateFunction (4 hours)

**Morning (2 hours):**
- ✅ 9:00-10:30 AM: Create ConfigUpdateService.cs
- ✅ 10:30-11:00 AM: Create ConfigurationValidator.cs

**Afternoon (2 hours):**
- ✅ 1:00-1:30 PM: Create ConfigurationUpdateFunction.cs
- ✅ 1:30-2:00 PM: Create models (ConfigurationUpdateEvent, ConfigurationAudit)
- ✅ 2:00-2:30 PM: Write unit tests
- ✅ 2:30-3:00 PM: Test locally

### Day 3: Deployment & Testing (2 hours)

**Morning (1 hour):**
- ✅ 9:00-9:30 AM: Deploy to Azure Dev
- ✅ 9:30-10:00 AM: Setup Event Grid subscription

**Afternoon (1 hour):**
- ✅ 1:00-1:30 PM: Configure Azure Monitor availability test
- ✅ 1:30-2:00 PM: End-to-end testing

---

## File Structure

```
edi-sftp-connector/
├── src/
│   └── EDI.SFTP.Functions/
│       ├── Functions/
│       │   ├── SftpDownloadFunction.cs (existing)
│       │   ├── SftpUploadFunction.cs (existing)
│       │   ├── HealthCheckFunction.cs (NEW)
│       │   └── ConfigurationUpdateFunction.cs (NEW)
│       ├── Services/
│       │   ├── SftpService.cs (existing)
│       │   ├── TrackingService.cs (existing)
│       │   ├── HealthCheckService.cs (NEW)
│       │   ├── ConfigUpdateService.cs (NEW)
│       │   └── ConfigurationValidator.cs (NEW)
│       └── Models/
│           ├── HealthStatus.cs (NEW)
│           ├── ComponentHealth.cs (NEW)
│           ├── HealthMetrics.cs (NEW)
│           ├── ConfigurationUpdateEvent.cs (NEW)
│           ├── ConfigurationAudit.cs (NEW)
│           └── ValidationResult.cs (NEW)
```

---

## Step-by-Step Implementation

### Step 1: Create Models (15 minutes)

#### 1.1 Create `Models/HealthStatus.cs`

```csharp
using System;
using System.Collections.Generic;

namespace EDI.SFTP.Functions.Models
{
    public class HealthStatus
    {
        public bool IsHealthy { get; set; }
        public DateTime Timestamp { get; set; }
        public string Version { get; set; }
        public Dictionary<string, ComponentHealth> Components { get; set; } = new();
        public HealthMetrics Metrics { get; set; }
    }
}
```

#### 1.2 Create `Models/ComponentHealth.cs`

```csharp
using System;

namespace EDI.SFTP.Functions.Models
{
    public class ComponentHealth
    {
        public string Name { get; set; }
        public bool IsHealthy { get; set; }
        public string Status { get; set; } // "Healthy", "Degraded", "Unhealthy"
        public string Message { get; set; }
        public TimeSpan ResponseTime { get; set; }
        public DateTime LastChecked { get; set; }
    }
}
```

#### 1.3 Create `Models/HealthMetrics.cs`

```csharp
using System;

namespace EDI.SFTP.Functions.Models
{
    public class HealthMetrics
    {
        public int FilesProcessedLast24Hours { get; set; }
        public int ActivePartners { get; set; }
        public int FailedFilesLast24Hours { get; set; }
        public DateTime LastSuccessfulDownload { get; set; }
        public DateTime LastSuccessfulUpload { get; set; }
    }
}
```

#### 1.4 Create `Models/ConfigurationUpdateEvent.cs`

```csharp
using System;

namespace EDI.SFTP.Functions.Models
{
    public class ConfigurationUpdateEvent
    {
        public string PartnerCode { get; set; }
        public string BlobUrl { get; set; }
        public string ChangeType { get; set; } // "Created", "Updated", "Deleted"
        public DateTime EventTime { get; set; }
        public string ChangedBy { get; set; }
        public string OldVersion { get; set; }
        public string NewVersion { get; set; }
    }
}
```

#### 1.5 Create `Models/ValidationResult.cs`

```csharp
using System.Collections.Generic;

namespace EDI.SFTP.Functions.Models
{
    public class ValidationResult
    {
        public bool IsValid { get; set; }
        public List<string> Errors { get; set; } = new();
    }
}
```

---

### Step 2: Create HealthCheckService (1 hour)

#### 2.1 Create `Services/IHealthCheckService.cs`

```csharp
using System.Threading.Tasks;
using EDI.SFTP.Functions.Models;

namespace EDI.SFTP.Functions.Services
{
    public interface IHealthCheckService
    {
        Task<HealthStatus> GetHealthStatusAsync();
    }
}
```

#### 2.2 Create `Services/HealthCheckService.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Reflection;
using System.Threading.Tasks;
using Azure.Storage.Blobs;
using EDI.SFTP.Core.Data;
using EDI.SFTP.Functions.Models;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;
using Argus.Configuration;

namespace EDI.SFTP.Functions.Services
{
    public class HealthCheckService : IHealthCheckService
    {
        private readonly ISftpTrackingDbContext _dbContext;
        private readonly BlobServiceClient _blobClient;
        private readonly IPartnerConfigService _partnerConfigService;
        private readonly ILogger<HealthCheckService> _logger;
        
        public HealthCheckService(
            ISftpTrackingDbContext dbContext,
            BlobServiceClient blobClient,
            IPartnerConfigService partnerConfigService,
            ILogger<HealthCheckService> logger)
        {
            _dbContext = dbContext;
            _blobClient = blobClient;
            _partnerConfigService = partnerConfigService;
            _logger = logger;
        }
        
        public async Task<HealthStatus> GetHealthStatusAsync()
        {
            var components = new Dictionary<string, ComponentHealth>();
            
            // Check all components
            components["Database"] = await CheckDatabaseHealthAsync();
            components["BlobStorage"] = await CheckBlobStorageHealthAsync();
            components["PartnerConfiguration"] = await CheckPartnerConfigHealthAsync();
            
            // Get metrics
            var metrics = await GetHealthMetricsAsync();
            
            // Overall health is healthy if no components are unhealthy
            bool isHealthy = components.Values.All(c => c.Status != "Unhealthy");
            
            return new HealthStatus
            {
                IsHealthy = isHealthy,
                Timestamp = DateTime.UtcNow,
                Version = Assembly.GetExecutingAssembly().GetName().Version?.ToString() ?? "1.0.0",
                Components = components,
                Metrics = metrics
            };
        }
        
        private async Task<ComponentHealth> CheckDatabaseHealthAsync()
        {
            var stopwatch = Stopwatch.StartNew();
            try
            {
                // Simple query to verify connectivity
                var count = await _dbContext.FileTrackings
                    .Where(f => f.ProcessedDate >= DateTime.UtcNow.AddHours(-1))
                    .CountAsync();
                
                stopwatch.Stop();
                
                var status = stopwatch.Elapsed.TotalSeconds < 2 ? "Healthy" : "Degraded";
                
                return new ComponentHealth
                {
                    Name = "SQL Database",
                    IsHealthy = true,
                    Status = status,
                    Message = $"Connected. {count} files processed in last hour.",
                    ResponseTime = stopwatch.Elapsed,
                    LastChecked = DateTime.UtcNow
                };
            }
            catch (Exception ex)
            {
                stopwatch.Stop();
                _logger.LogError(ex, "Database health check failed");
                
                return new ComponentHealth
                {
                    Name = "SQL Database",
                    IsHealthy = false,
                    Status = "Unhealthy",
                    Message = $"Connection failed: {ex.Message}",
                    ResponseTime = stopwatch.Elapsed,
                    LastChecked = DateTime.UtcNow
                };
            }
        }
        
        private async Task<ComponentHealth> CheckBlobStorageHealthAsync()
        {
            var stopwatch = Stopwatch.StartNew();
            try
            {
                // Check if inbound container exists
                var containerClient = _blobClient.GetBlobContainerClient("inbound");
                var exists = await containerClient.ExistsAsync();
                
                stopwatch.Stop();
                
                return new ComponentHealth
                {
                    Name = "Blob Storage",
                    IsHealthy = exists.Value,
                    Status = exists.Value ? "Healthy" : "Unhealthy",
                    Message = exists.Value 
                        ? "Connected to blob storage" 
                        : "Blob container 'inbound' not found",
                    ResponseTime = stopwatch.Elapsed,
                    LastChecked = DateTime.UtcNow
                };
            }
            catch (Exception ex)
            {
                stopwatch.Stop();
                _logger.LogError(ex, "Blob storage health check failed");
                
                return new ComponentHealth
                {
                    Name = "Blob Storage",
                    IsHealthy = false,
                    Status = "Unhealthy",
                    Message = $"Connection failed: {ex.Message}",
                    ResponseTime = stopwatch.Elapsed,
                    LastChecked = DateTime.UtcNow
                };
            }
        }
        
        private async Task<ComponentHealth> CheckPartnerConfigHealthAsync()
        {
            var stopwatch = Stopwatch.StartNew();
            try
            {
                var partners = await _partnerConfigService.GetAllPartnerConfigsAsync();
                stopwatch.Stop();
                
                var activePartners = partners.Count(p => p.IsActive);
                
                return new ComponentHealth
                {
                    Name = "Partner Configuration",
                    IsHealthy = activePartners > 0,
                    Status = activePartners > 0 ? "Healthy" : "Degraded",
                    Message = $"{activePartners} active partners configured",
                    ResponseTime = stopwatch.Elapsed,
                    LastChecked = DateTime.UtcNow
                };
            }
            catch (Exception ex)
            {
                stopwatch.Stop();
                _logger.LogError(ex, "Partner config health check failed");
                
                return new ComponentHealth
                {
                    Name = "Partner Configuration",
                    IsHealthy = false,
                    Status = "Unhealthy",
                    Message = $"Config load failed: {ex.Message}",
                    ResponseTime = stopwatch.Elapsed,
                    LastChecked = DateTime.UtcNow
                };
            }
        }
        
        private async Task<HealthMetrics> GetHealthMetricsAsync()
        {
            try
            {
                var last24Hours = DateTime.UtcNow.AddHours(-24);
                
                var filesProcessed = await _dbContext.FileTrackings
                    .Where(f => f.ProcessedDate >= last24Hours && f.Status == ProcessingStatus.Completed)
                    .CountAsync();
                
                var filesFailed = await _dbContext.FileTrackings
                    .Where(f => f.ProcessedDate >= last24Hours && f.Status == ProcessingStatus.Failed)
                    .CountAsync();
                
                var lastDownload = await _dbContext.FileTrackings
                    .Where(f => f.Direction == FileDirection.Download && f.Status == ProcessingStatus.Completed)
                    .OrderByDescending(f => f.ProcessedDate)
                    .Select(f => f.ProcessedDate)
                    .FirstOrDefaultAsync();
                
                var lastUpload = await _dbContext.FileTrackings
                    .Where(f => f.Direction == FileDirection.Upload && f.Status == ProcessingStatus.Completed)
                    .OrderByDescending(f => f.ProcessedDate)
                    .Select(f => f.ProcessedDate)
                    .FirstOrDefaultAsync();
                
                var activePartners = await _partnerConfigService.GetAllPartnerConfigsAsync();
                
                return new HealthMetrics
                {
                    FilesProcessedLast24Hours = filesProcessed,
                    ActivePartners = activePartners.Count(p => p.IsActive),
                    FailedFilesLast24Hours = filesFailed,
                    LastSuccessfulDownload = lastDownload ?? DateTime.MinValue,
                    LastSuccessfulUpload = lastUpload ?? DateTime.MinValue
                };
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to get health metrics");
                return null;
            }
        }
    }
}
```

---

### Step 3: Create HealthCheckFunction (30 minutes)

#### 3.1 Create `Functions/HealthCheckFunction.cs`

```csharp
using System;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using EDI.SFTP.Functions.Services;

namespace EDI.SFTP.Functions.Functions
{
    public class HealthCheckFunction
    {
        private readonly IHealthCheckService _healthCheckService;
        private readonly ILogger<HealthCheckFunction> _logger;
        
        public HealthCheckFunction(
            IHealthCheckService healthCheckService,
            ILogger<HealthCheckFunction> logger)
        {
            _healthCheckService = healthCheckService;
            _logger = logger;
        }
        
        [FunctionName("HealthCheckFunction")]
        public async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "health")] HttpRequest req)
        {
            _logger.LogInformation("Health check requested at {Timestamp}", DateTime.UtcNow);
            
            try
            {
                var healthStatus = await _healthCheckService.GetHealthStatusAsync();
                
                if (healthStatus.IsHealthy)
                {
                    _logger.LogInformation(
                        "Health check passed. Components: {HealthyCount}/{TotalCount}",
                        healthStatus.Components.Count(c => c.Value.IsHealthy),
                        healthStatus.Components.Count);
                    
                    return new OkObjectResult(healthStatus);
                }
                else
                {
                    _logger.LogWarning(
                        "Health check failed. Unhealthy components: {UnhealthyComponents}",
                        string.Join(", ", healthStatus.Components.Where(c => !c.Value.IsHealthy).Select(c => c.Key)));
                    
                    return new ObjectResult(healthStatus)
                    {
                        StatusCode = StatusCodes.Status503ServiceUnavailable
                    };
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Health check failed with exception");
                
                return new ObjectResult(new
                {
                    isHealthy = false,
                    timestamp = DateTime.UtcNow,
                    error = ex.Message
                })
                {
                    StatusCode = StatusCodes.Status503ServiceUnavailable
                };
            }
        }
    }
}
```

---

### Step 4: Register Services (15 minutes)

#### 4.1 Update `Program.cs` or `Startup.cs`

Add to your DI configuration:

```csharp
// In ConfigureServices or equivalent
services.AddScoped<IHealthCheckService, HealthCheckService>();
services.AddScoped<IConfigUpdateService, ConfigUpdateService>();
services.AddScoped<IConfigurationValidator, ConfigurationValidator>();
```

---

### Step 5: Test Locally (30 minutes)

#### 5.1 Run the function

```powershell
# Navigate to project
cd c:\repos\edi-platform\edi-sftp-connector

# Start Functions runtime
func start --csharp
```

#### 5.2 Test health check endpoint

```powershell
# Test with curl
curl http://localhost:7071/api/health | ConvertFrom-Json | ConvertTo-Json -Depth 10

# Or use Invoke-RestMethod
$health = Invoke-RestMethod -Uri "http://localhost:7071/api/health"
$health | ConvertTo-Json -Depth 10
```

#### 5.3 Expected output

```json
{
  "isHealthy": true,
  "timestamp": "2025-10-07T14:30:00Z",
  "version": "1.0.0",
  "components": {
    "Database": {
      "name": "SQL Database",
      "isHealthy": true,
      "status": "Healthy",
      "message": "Connected. 12 files processed in last hour.",
      "responseTime": "00:00:00.0235",
      "lastChecked": "2025-10-07T14:30:00Z"
    }
  }
}
```

---

## Deployment Checklist

### Azure Function App Deployment

- [ ] **Build and publish**
  ```powershell
  dotnet build --configuration Release
  dotnet publish --configuration Release --output ./publish
  ```

- [ ] **Create deployment package**
  ```powershell
  Compress-Archive -Path ./publish/* -DestinationPath ./deploy.zip -Force
  ```

- [ ] **Deploy to Azure**
  ```powershell
  az functionapp deployment source config-zip `
    --resource-group edi-platform-rg `
    --name edi-sftp-connector-func `
    --src ./deploy.zip
  ```

### Azure Monitor Setup

- [ ] **Create availability test**
  ```powershell
  # See detailed commands in main implementation guide
  az monitor app-insights web-test create ...
  ```

- [ ] **Configure alert rules**
  - Alert 1: SFTP Connector Unhealthy (Sev 1)
  - Alert 2: Configuration Update Failed (Sev 2)

### Event Grid Setup (for ConfigurationUpdateFunction)

- [ ] **Create Event Grid subscription**
  ```powershell
  az eventgrid event-subscription create `
    --name partner-config-updates `
    --source-resource-id "/subscriptions/.../storageAccounts/ediconfigs" `
    --endpoint "https://edi-sftp-connector-func.azurewebsites.net/runtime/webhooks/eventgrid?functionName=ConfigurationUpdateFunction" `
    --endpoint-type azurefunction `
    --included-event-types Microsoft.Storage.BlobCreated `
    --subject-begins-with "/blobServices/default/containers/partner-configs/" `
    --subject-ends-with ".json"
  ```

---

## Testing Checklist

### Unit Tests

- [ ] **HealthCheckService tests**
  - Test healthy database connection
  - Test unhealthy database connection
  - Test blob storage connectivity
  - Test partner config loading
  - Test metrics calculation

- [ ] **ConfigurationValidator tests**
  - Test valid configuration
  - Test invalid configuration (missing fields)
  - Test invalid SFTP port
  - Test invalid ISA sender ID length

### Integration Tests

- [ ] **End-to-end health check**
  - Deploy to Dev environment
  - Call /api/health endpoint
  - Verify HTTP 200 response
  - Verify JSON structure

- [ ] **End-to-end config update**
  - Upload test partner config to blob
  - Verify Event Grid triggers function
  - Verify config reloaded in cache
  - Verify audit log created

### Load Tests

- [ ] **Health check performance**
  - 100 concurrent requests
  - Verify p95 < 500ms
  - Verify no timeouts

---

## Troubleshooting

### Common Issues

**Issue 1**: Health check always returns unhealthy for database

**Solution**:
```csharp
// Verify connection string in local.settings.json
"ConnectionStrings:DefaultConnection": "Server=localhost;Database=EDI_SftpTracking;Trusted_Connection=true;"

// Test database connection
dotnet ef database update --project src/EDI.SFTP.Core
```

**Issue 2**: Blob storage health check fails

**Solution**:
```powershell
# Verify storage emulator is running
azurite --silent --location c:\azurite --debug c:\azurite\debug.log

# Or use Azure Storage connection string
"ConnectionStrings:BlobStorage": "DefaultEndpointsProtocol=https;AccountName=...;AccountKey=..."
```

**Issue 3**: Partner config health check fails

**Solution**:
```csharp
// Verify partner config service is registered in DI
services.AddScoped<IPartnerConfigService, PartnerConfigService>();

// Verify partner configs exist in blob storage
az storage blob list --account-name ediconfigs --container-name partner-configs
```

---

## Next Steps

After implementing these functions:

1. **Monitor health check metrics** in Azure Monitor
2. **Setup alerts** for unhealthy components
3. **Test configuration hot reload** by uploading new partner config
4. **Document runbooks** for responding to health check alerts
5. **Train operations team** on using health check endpoint

---

## Related Documents

- **Detailed Guide**: `SFTP-CONNECTOR-ADDITIONAL-FUNCTIONS.md`
- **Quick Summary**: `SFTP-CONNECTOR-ADDITIONAL-FUNCTIONS-SUMMARY.md`
- **Task List**: `18-implementation-task-list.md` (Section 1.3)

---

**Status**: Ready to Implement  
**Estimated Time**: 9 hours (3 days @ 3 hours/day)  
**Last Updated**: October 7, 2025
