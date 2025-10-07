# SFTP Connector - Additional Functions Implementation Guide

**Document Version:** 1.0  
**Date:** October 7, 2025  
**Status:** Implementation Ready  
**Purpose:** Detailed implementation guide for optional SFTP Connector functions  

---

## Executive Summary

The EDI SFTP Connector currently has **2 production-ready functions** (`SftpDownloadFunction` and `SftpUploadFunction`, both at 95% completion). The README documents **2 additional optional functions** that can enhance operational monitoring and configuration management.

### Quick Overview

| Function | Status | Priority | Effort | Value Proposition |
|----------|--------|----------|--------|-------------------|
| **HealthCheckFunction** | ❌ Not Implemented | P2 (Medium) | 3 hours | Proactive monitoring, Azure Monitor integration |
| **ConfigurationUpdateFunction** | ❌ Not Implemented | P2 (Medium) | 4 hours | Hot configuration reload, zero-downtime updates |

**Total Effort**: 7 hours (~1 day)  
**Classification**: Optional enhancements (not blocking MVP)

---

## Table of Contents

1. [HealthCheckFunction - Implementation Guide](#1-healthcheckfunction)
2. [ConfigurationUpdateFunction - Implementation Guide](#2-configurationupdatefunction)
3. [Testing Strategy](#3-testing-strategy)
4. [Deployment Guide](#4-deployment-guide)
5. [Monitoring & Alerts](#5-monitoring--alerts)
6. [Cost-Benefit Analysis](#6-cost-benefit-analysis)

---

## 1. HealthCheckFunction

### 1.1 Purpose & Value Proposition

**What it does**: HTTP-triggered endpoint that performs connectivity checks and returns system health status in JSON format.

**Why implement it**:
- **Proactive Monitoring**: Detect issues before they impact file processing
- **Azure Monitor Integration**: Enable availability tests and uptime SLAs
- **Troubleshooting**: Quick diagnostics during incidents
- **Load Balancer Integration**: Health checks for multi-region deployments
- **Compliance**: Demonstrate system availability for audit requirements

**Use Cases**:
- Azure Monitor Availability Tests (synthetic monitoring)
- Load balancer health probes
- Status page integration
- On-call engineer quick diagnostics
- Partner SLA reporting

### 1.2 Technical Design

#### Function Signature

```csharp
[FunctionName("HealthCheckFunction")]
public async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "health")] HttpRequest req,
    ILogger log)
{
    log.LogInformation("Health check requested");
    
    var healthStatus = await _healthCheckService.GetHealthStatusAsync();
    
    return healthStatus.IsHealthy
        ? new OkObjectResult(healthStatus)
        : new StatusCodeResult(StatusCodes.Status503ServiceUnavailable);
}
```

#### Health Check Service Interface

```csharp
public interface IHealthCheckService
{
    Task<HealthStatus> GetHealthStatusAsync();
}

public class HealthStatus
{
    public bool IsHealthy { get; set; }
    public DateTime Timestamp { get; set; }
    public string Version { get; set; }
    public Dictionary<string, ComponentHealth> Components { get; set; }
    public HealthMetrics Metrics { get; set; }
}

public class ComponentHealth
{
    public string Name { get; set; }
    public bool IsHealthy { get; set; }
    public string Status { get; set; } // "Healthy", "Degraded", "Unhealthy"
    public string Message { get; set; }
    public TimeSpan ResponseTime { get; set; }
    public DateTime LastChecked { get; set; }
}

public class HealthMetrics
{
    public int FilesProcessedLast24Hours { get; set; }
    public int ActivePartners { get; set; }
    public int FailedFilesLast24Hours { get; set; }
    public DateTime LastSuccessfulDownload { get; set; }
    public DateTime LastSuccessfulUpload { get; set; }
}
```

#### Health Check Implementation

```csharp
public class HealthCheckService : IHealthCheckService
{
    private readonly ISftpTrackingDbContext _dbContext;
    private readonly BlobServiceClient _blobClient;
    private readonly IPartnerConfigService _partnerConfigService;
    private readonly ILogger<HealthCheckService> _logger;
    
    public async Task<HealthStatus> GetHealthStatusAsync()
    {
        var components = new Dictionary<string, ComponentHealth>();
        
        // 1. Check Database Connectivity
        components["Database"] = await CheckDatabaseHealthAsync();
        
        // 2. Check Blob Storage Connectivity
        components["BlobStorage"] = await CheckBlobStorageHealthAsync();
        
        // 3. Check Partner Configuration Availability
        components["PartnerConfiguration"] = await CheckPartnerConfigHealthAsync();
        
        // 4. Check Recent Processing Statistics
        var metrics = await GetHealthMetricsAsync();
        
        bool isHealthy = components.Values.All(c => c.Status != "Unhealthy");
        
        return new HealthStatus
        {
            IsHealthy = isHealthy,
            Timestamp = DateTime.UtcNow,
            Version = Assembly.GetExecutingAssembly().GetName().Version?.ToString() ?? "Unknown",
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
            
            return new ComponentHealth
            {
                Name = "SQL Database",
                IsHealthy = true,
                Status = stopwatch.Elapsed.TotalSeconds < 2 ? "Healthy" : "Degraded",
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
            // List containers to verify connectivity
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
                    : "Blob container not found",
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
}
```

### 1.3 Sample Response

#### Healthy System

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
    },
    "BlobStorage": {
      "name": "Blob Storage",
      "isHealthy": true,
      "status": "Healthy",
      "message": "Connected to blob storage",
      "responseTime": "00:00:00.0123",
      "lastChecked": "2025-10-07T14:30:00Z"
    },
    "PartnerConfiguration": {
      "name": "Partner Configuration",
      "isHealthy": true,
      "status": "Healthy",
      "message": "5 active partners configured",
      "responseTime": "00:00:00.0456",
      "lastChecked": "2025-10-07T14:30:00Z"
    }
  },
  "metrics": {
    "filesProcessedLast24Hours": 145,
    "activePartners": 5,
    "failedFilesLast24Hours": 2,
    "lastSuccessfulDownload": "2025-10-07T14:25:00Z",
    "lastSuccessfulUpload": "2025-10-07T14:28:00Z"
  }
}
```

#### Unhealthy System (Database Down)

```json
{
  "isHealthy": false,
  "timestamp": "2025-10-07T14:30:00Z",
  "version": "1.0.0",
  "components": {
    "Database": {
      "name": "SQL Database",
      "isHealthy": false,
      "status": "Unhealthy",
      "message": "Connection failed: Timeout expired",
      "responseTime": "00:00:30.0000",
      "lastChecked": "2025-10-07T14:30:00Z"
    },
    "BlobStorage": {
      "name": "Blob Storage",
      "isHealthy": true,
      "status": "Healthy",
      "message": "Connected to blob storage",
      "responseTime": "00:00:00.0123",
      "lastChecked": "2025-10-07T14:30:00Z"
    },
    "PartnerConfiguration": {
      "name": "Partner Configuration",
      "isHealthy": true,
      "status": "Healthy",
      "message": "5 active partners configured",
      "responseTime": "00:00:00.0456",
      "lastChecked": "2025-10-07T14:30:00Z"
    }
  },
  "metrics": null
}
```

### 1.4 Implementation Checklist

**Time Estimate: 3 hours**

- [ ] **Create HealthCheckService.cs** (1 hour)
  - Implement IHealthCheckService interface
  - Add CheckDatabaseHealthAsync() method
  - Add CheckBlobStorageHealthAsync() method
  - Add CheckPartnerConfigHealthAsync() method
  - Add GetHealthMetricsAsync() method
  
- [ ] **Create HealthCheckFunction.cs** (0.5 hours)
  - Add HTTP trigger with anonymous authorization
  - Call HealthCheckService
  - Return appropriate HTTP status codes
  - Add structured logging
  
- [ ] **Create Models** (0.5 hours)
  - HealthStatus.cs
  - ComponentHealth.cs
  - HealthMetrics.cs
  
- [ ] **Register Services** (0.25 hours)
  - Add IHealthCheckService to DI container
  - Configure service lifetime (Scoped)
  
- [ ] **Write Unit Tests** (0.5 hours)
  - Test healthy system response
  - Test unhealthy system response
  - Test degraded system response
  - Mock dependencies
  
- [ ] **Write Integration Tests** (0.25 hours)
  - Test with real Azure resources
  - Verify JSON serialization
  - Test HTTP status codes

### 1.5 Azure Monitor Integration

#### Availability Test Configuration

```json
{
  "name": "SFTP Connector Health Check",
  "enabled": true,
  "frequency": 300,
  "timeout": 120,
  "locations": [
    {
      "Id": "us-va-ash-azr"
    },
    {
      "Id": "us-il-ch1-azr"
    }
  ],
  "configuration": {
    "webTest": {
      "kind": "ping",
      "url": "https://edi-sftp-connector-func.azurewebsites.net/api/health",
      "expectedHttpStatusCode": 200,
      "validationRules": {
        "expectedStatusCode": 200,
        "contentValidation": {
          "contentMatch": "\"isHealthy\":true",
          "ignoreCase": true
        }
      }
    }
  },
  "alertRules": [
    {
      "name": "SFTP Connector Unhealthy",
      "severity": 1,
      "condition": "failedLocationCount >= 2"
    }
  ]
}
```

---

## 2. ConfigurationUpdateFunction

### 2.1 Purpose & Value Proposition

**What it does**: Event Grid-triggered function that reloads partner configurations when changes are detected, enabling zero-downtime configuration updates.

**Why implement it**:
- **Zero-Downtime Updates**: No need to restart functions when config changes
- **Hot Reload**: Configurations take effect within seconds
- **Cache Invalidation**: Automatic cache refresh across all function instances
- **Audit Trail**: Track all configuration changes
- **Rollback Support**: Validate new configs before applying

**Use Cases**:
- Add new trading partner without downtime
- Update SFTP credentials without restart
- Change polling intervals dynamically
- Enable/disable partners instantly
- Test partner configurations in production

### 2.2 Technical Design

#### Function Signature

```csharp
[FunctionName("ConfigurationUpdateFunction")]
public async Task Run(
    [EventGridTrigger] EventGridEvent eventGridEvent,
    ILogger log)
{
    log.LogInformation($"Configuration update event received: {eventGridEvent.EventType}");
    
    var configUpdateEvent = eventGridEvent.Data.ToObjectFromJson<ConfigurationUpdateEvent>();
    
    await _configUpdateService.ProcessConfigurationUpdateAsync(configUpdateEvent);
}
```

#### Event Grid Event Schema

**Event Type**: `Microsoft.Storage.BlobCreated` or custom `PartnerConfiguration.Updated`

```json
{
  "topic": "/subscriptions/.../resourceGroups/.../providers/Microsoft.Storage/storageAccounts/ediconfigs",
  "subject": "/blobServices/default/containers/partner-configs/blobs/PARTNERA.json",
  "eventType": "Microsoft.Storage.BlobCreated",
  "eventTime": "2025-10-07T14:30:00Z",
  "id": "12345678-1234-1234-1234-123456789012",
  "data": {
    "api": "PutBlob",
    "clientRequestId": "...",
    "requestId": "...",
    "eTag": "0x8D8...",
    "contentType": "application/json",
    "contentLength": 2048,
    "blobType": "BlockBlob",
    "url": "https://ediconfigs.blob.core.windows.net/partner-configs/PARTNERA.json",
    "sequencer": "...",
    "storageDiagnostics": {}
  },
  "dataVersion": "",
  "metadataVersion": "1"
}
```

#### Configuration Update Service

```csharp
public interface IConfigUpdateService
{
    Task ProcessConfigurationUpdateAsync(ConfigurationUpdateEvent updateEvent);
}

public class ConfigUpdateService : IConfigUpdateService
{
    private readonly IPartnerConfigService _partnerConfigService;
    private readonly IConfigurationValidator _validator;
    private readonly IConfigurationAuditor _auditor;
    private readonly IMemoryCache _cache;
    private readonly ILogger<ConfigUpdateService> _logger;
    
    public async Task ProcessConfigurationUpdateAsync(ConfigurationUpdateEvent updateEvent)
    {
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            _logger.LogInformation(
                "Processing configuration update for {PartnerCode}",
                updateEvent.PartnerCode);
            
            // Step 1: Download new configuration
            var newConfig = await DownloadConfigurationAsync(updateEvent.BlobUrl);
            
            // Step 2: Validate new configuration
            var validationResult = await _validator.ValidateAsync(newConfig);
            if (!validationResult.IsValid)
            {
                _logger.LogError(
                    "Configuration validation failed for {PartnerCode}: {Errors}",
                    updateEvent.PartnerCode,
                    string.Join(", ", validationResult.Errors));
                
                // Send alert about invalid configuration
                throw new ConfigurationValidationException(validationResult.Errors);
            }
            
            // Step 3: Backup current configuration (for rollback)
            await BackupCurrentConfigurationAsync(updateEvent.PartnerCode);
            
            // Step 4: Invalidate cache
            InvalidateCache(updateEvent.PartnerCode);
            
            // Step 5: Reload configuration in service
            await _partnerConfigService.ReloadPartnerConfigAsync(updateEvent.PartnerCode);
            
            // Step 6: Audit the change
            await _auditor.AuditConfigurationChangeAsync(new ConfigurationAudit
            {
                PartnerCode = updateEvent.PartnerCode,
                ChangeType = updateEvent.ChangeType,
                ChangedBy = updateEvent.ChangedBy,
                Timestamp = DateTime.UtcNow,
                OldVersion = updateEvent.OldVersion,
                NewVersion = updateEvent.NewVersion,
                BlobUrl = updateEvent.BlobUrl
            });
            
            stopwatch.Stop();
            
            _logger.LogInformation(
                "Configuration reloaded successfully for {PartnerCode} in {Duration}ms",
                updateEvent.PartnerCode,
                stopwatch.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            
            _logger.LogError(ex,
                "Failed to process configuration update for {PartnerCode}",
                updateEvent.PartnerCode);
            
            // Attempt rollback
            await RollbackConfigurationAsync(updateEvent.PartnerCode);
            
            throw;
        }
    }
    
    private async Task<PartnerConfig> DownloadConfigurationAsync(string blobUrl)
    {
        var blobClient = new BlobClient(new Uri(blobUrl));
        var response = await blobClient.DownloadContentAsync();
        var json = response.Value.Content.ToString();
        return JsonSerializer.Deserialize<PartnerConfig>(json);
    }
    
    private void InvalidateCache(string partnerCode)
    {
        var cacheKey = $"partner-config:{partnerCode}";
        _cache.Remove(cacheKey);
        
        _logger.LogInformation(
            "Cache invalidated for {PartnerCode}",
            partnerCode);
    }
    
    private async Task BackupCurrentConfigurationAsync(string partnerCode)
    {
        var currentConfig = await _partnerConfigService.GetPartnerConfigAsync(partnerCode);
        if (currentConfig != null)
        {
            var backupKey = $"partner-config-backup:{partnerCode}:{DateTime.UtcNow:yyyyMMddHHmmss}";
            _cache.Set(backupKey, currentConfig, TimeSpan.FromHours(24));
            
            _logger.LogInformation(
                "Current configuration backed up with key {BackupKey}",
                backupKey);
        }
    }
    
    private async Task RollbackConfigurationAsync(string partnerCode)
    {
        _logger.LogWarning(
            "Attempting rollback for {PartnerCode}",
            partnerCode);
        
        // Find most recent backup
        var backupKey = $"partner-config-backup:{partnerCode}:";
        // Implementation would search cache for most recent backup
        // and restore it
        
        // For production, consider storing backups in blob storage
    }
}
```

#### Configuration Validator

```csharp
public interface IConfigurationValidator
{
    Task<ValidationResult> ValidateAsync(PartnerConfig config);
}

public class ConfigurationValidator : IConfigurationValidator
{
    private readonly ILogger<ConfigurationValidator> _logger;
    
    public async Task<ValidationResult> ValidateAsync(PartnerConfig config)
    {
        var errors = new List<string>();
        
        // 1. Basic validation
        if (string.IsNullOrWhiteSpace(config.PartnerCode))
            errors.Add("PartnerCode is required");
        
        if (string.IsNullOrWhiteSpace(config.PartnerName))
            errors.Add("PartnerName is required");
        
        // 2. Endpoint validation
        foreach (var endpoint in config.Endpoints)
        {
            if (endpoint.Type == EndpointType.SFTP)
            {
                if (string.IsNullOrWhiteSpace(endpoint.Configuration["Host"]))
                    errors.Add($"SFTP Host is required for endpoint {endpoint.EndpointId}");
                
                if (!int.TryParse(endpoint.Configuration["Port"], out int port) || port < 1 || port > 65535)
                    errors.Add($"Valid SFTP Port (1-65535) is required for endpoint {endpoint.EndpointId}");
            }
        }
        
        // 3. X12 identifier validation
        if (config.X12Identifiers != null)
        {
            if (string.IsNullOrWhiteSpace(config.X12Identifiers.IsaSenderId))
                errors.Add("ISA Sender ID is required");
            
            if (config.X12Identifiers.IsaSenderId?.Length != 15)
                errors.Add("ISA Sender ID must be exactly 15 characters");
        }
        
        // 4. Test SFTP connectivity (optional, can be slow)
        // await TestSftpConnectionAsync(config);
        
        return new ValidationResult
        {
            IsValid = errors.Count == 0,
            Errors = errors
        };
    }
}

public class ValidationResult
{
    public bool IsValid { get; set; }
    public List<string> Errors { get; set; } = new();
}
```

### 2.3 Event Grid Subscription Setup

#### Using Azure CLI

```bash
# Create Event Grid subscription for blob created events
az eventgrid event-subscription create \
  --name partner-config-updates \
  --source-resource-id "/subscriptions/.../resourceGroups/edi-platform-rg/providers/Microsoft.Storage/storageAccounts/ediconfigs" \
  --endpoint "https://edi-sftp-connector-func.azurewebsites.net/runtime/webhooks/eventgrid?functionName=ConfigurationUpdateFunction" \
  --endpoint-type azurefunction \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with "/blobServices/default/containers/partner-configs/" \
  --subject-ends-with ".json"
```

#### Using Bicep

```bicep
resource eventSubscription 'Microsoft.EventGrid/eventSubscriptions@2021-12-01' = {
  name: 'partner-config-updates'
  scope: storageAccount
  properties: {
    destination: {
      endpointType: 'AzureFunction'
      properties: {
        resourceId: '${functionApp.id}/functions/ConfigurationUpdateFunction'
        maxEventsPerBatch: 1
        preferredBatchSizeInKilobytes: 64
      }
    }
    filter: {
      subjectBeginsWith: '/blobServices/default/containers/partner-configs/'
      subjectEndsWith: '.json'
      includedEventTypes: [
        'Microsoft.Storage.BlobCreated'
      ]
    }
    eventDeliverySchema: 'EventGridSchema'
    retryPolicy: {
      maxDeliveryAttempts: 3
      eventTimeToLiveInMinutes: 1440
    }
  }
}
```

### 2.4 Implementation Checklist

**Time Estimate: 4 hours**

- [ ] **Create ConfigUpdateService.cs** (1.5 hours)
  - Implement ProcessConfigurationUpdateAsync()
  - Add configuration download logic
  - Add cache invalidation
  - Add backup/rollback logic
  - Add audit logging
  
- [ ] **Create ConfigurationValidator.cs** (1 hour)
  - Implement ValidateAsync() method
  - Add endpoint validation
  - Add X12 identifier validation
  - Add optional connectivity tests
  
- [ ] **Create ConfigurationUpdateFunction.cs** (0.5 hours)
  - Add Event Grid trigger
  - Call ConfigUpdateService
  - Add error handling
  
- [ ] **Create Models** (0.25 hours)
  - ConfigurationUpdateEvent.cs
  - ConfigurationAudit.cs
  - ValidationResult.cs
  
- [ ] **Setup Event Grid Subscription** (0.25 hours)
  - Create subscription via Bicep or Azure CLI
  - Test event delivery
  
- [ ] **Write Unit Tests** (0.5 hours)
  - Test configuration validation
  - Test cache invalidation
  - Test rollback logic
  - Mock Event Grid events

### 2.5 Testing Strategy

#### Manual Testing

```bash
# 1. Upload new partner configuration to blob storage
az storage blob upload \
  --account-name ediconfigs \
  --container-name partner-configs \
  --name PARTNERA.json \
  --file ./PARTNERA.json \
  --overwrite

# 2. Monitor function logs
func logs --function-name ConfigurationUpdateFunction

# 3. Verify configuration reloaded
curl https://edi-sftp-connector-func.azurewebsites.net/api/health | jq
```

#### Integration Test

```csharp
[Fact]
public async Task ConfigUpdate_ValidConfig_ShouldReloadSuccessfully()
{
    // Arrange
    var mockEvent = new EventGridEvent
    {
        EventType = "Microsoft.Storage.BlobCreated",
        Data = new
        {
            url = "https://ediconfigs.blob.core.windows.net/partner-configs/TESTPARTNER.json"
        }
    };
    
    // Act
    await _function.Run(mockEvent, _logger);
    
    // Assert
    var reloadedConfig = await _partnerConfigService.GetPartnerConfigAsync("TESTPARTNER");
    Assert.NotNull(reloadedConfig);
    Assert.True(reloadedConfig.IsActive);
}
```

---

## 3. Testing Strategy

### 3.1 Unit Tests

**Target**: 80%+ code coverage

#### HealthCheckService Tests

```csharp
public class HealthCheckServiceTests
{
    [Fact]
    public async Task GetHealthStatus_AllComponentsHealthy_ReturnsHealthyStatus()
    {
        // Arrange
        var mockDbContext = CreateMockDbContext(healthy: true);
        var mockBlobClient = CreateMockBlobClient(healthy: true);
        var mockConfigService = CreateMockConfigService(healthy: true);
        
        var service = new HealthCheckService(mockDbContext, mockBlobClient, mockConfigService, _logger);
        
        // Act
        var result = await service.GetHealthStatusAsync();
        
        // Assert
        Assert.True(result.IsHealthy);
        Assert.Equal("Healthy", result.Components["Database"].Status);
        Assert.Equal("Healthy", result.Components["BlobStorage"].Status);
        Assert.Equal("Healthy", result.Components["PartnerConfiguration"].Status);
    }
    
    [Fact]
    public async Task GetHealthStatus_DatabaseUnhealthy_ReturnsUnhealthyStatus()
    {
        // Arrange
        var mockDbContext = CreateMockDbContext(healthy: false);
        var service = new HealthCheckService(mockDbContext, _mockBlobClient, _mockConfigService, _logger);
        
        // Act
        var result = await service.GetHealthStatusAsync();
        
        // Assert
        Assert.False(result.IsHealthy);
        Assert.Equal("Unhealthy", result.Components["Database"].Status);
    }
}
```

#### ConfigurationValidator Tests

```csharp
public class ConfigurationValidatorTests
{
    [Fact]
    public async Task Validate_ValidConfig_ReturnsValid()
    {
        // Arrange
        var config = CreateValidPartnerConfig();
        var validator = new ConfigurationValidator(_logger);
        
        // Act
        var result = await validator.ValidateAsync(config);
        
        // Assert
        Assert.True(result.IsValid);
        Assert.Empty(result.Errors);
    }
    
    [Fact]
    public async Task Validate_MissingIsaSenderId_ReturnsInvalid()
    {
        // Arrange
        var config = CreateValidPartnerConfig();
        config.X12Identifiers.IsaSenderId = null;
        
        var validator = new ConfigurationValidator(_logger);
        
        // Act
        var result = await validator.ValidateAsync(config);
        
        // Assert
        Assert.False(result.IsValid);
        Assert.Contains("ISA Sender ID is required", result.Errors);
    }
}
```

### 3.2 Integration Tests

#### HealthCheckFunction Integration Test

```csharp
[Fact]
public async Task HealthCheck_WithRealDependencies_ReturnsHealthStatus()
{
    // Arrange - use test database and blob storage
    var testHost = CreateTestFunctionHost();
    
    // Act
    var response = await testHost.HttpClient.GetAsync("/api/health");
    
    // Assert
    Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    
    var content = await response.Content.ReadAsStringAsync();
    var healthStatus = JsonSerializer.Deserialize<HealthStatus>(content);
    
    Assert.NotNull(healthStatus);
    Assert.True(healthStatus.IsHealthy);
}
```

---

## 4. Deployment Guide

### 4.1 Prerequisites

- Azure Function App deployed
- Azure Storage Account for configurations
- Event Grid topic/subscription configured
- Application Insights enabled

### 4.2 Deployment Steps

#### 1. Deploy Functions

```bash
# Build the project
dotnet build --configuration Release

# Publish to local folder
dotnet publish --configuration Release --output ./publish

# Create deployment package
cd publish
zip -r ../deploy.zip .

# Deploy to Azure
az functionapp deployment source config-zip \
  --resource-group edi-platform-rg \
  --name edi-sftp-connector-func \
  --src ../deploy.zip
```

#### 2. Configure Application Settings

```bash
az functionapp config appsettings set \
  --resource-group edi-platform-rg \
  --name edi-sftp-connector-func \
  --settings \
    "HealthCheck:Enabled=true" \
    "ConfigUpdate:Enabled=true" \
    "ConfigUpdate:ValidationEnabled=true" \
    "ConfigUpdate:RollbackEnabled=true"
```

#### 3. Setup Event Grid Subscription

```bash
# Create Event Grid subscription
az eventgrid event-subscription create \
  --name partner-config-updates \
  --source-resource-id "/subscriptions/.../storageAccounts/ediconfigs" \
  --endpoint "https://edi-sftp-connector-func.azurewebsites.net/runtime/webhooks/eventgrid?functionName=ConfigurationUpdateFunction" \
  --endpoint-type azurefunction \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with "/blobServices/default/containers/partner-configs/"
```

#### 4. Verify Deployment

```bash
# Test health check endpoint
curl https://edi-sftp-connector-func.azurewebsites.net/api/health | jq

# Upload test configuration
az storage blob upload \
  --account-name ediconfigs \
  --container-name partner-configs \
  --name TESTPARTNER.json \
  --file ./test-config.json

# Check function logs
az webapp log tail \
  --resource-group edi-platform-rg \
  --name edi-sftp-connector-func
```

---

## 5. Monitoring & Alerts

### 5.1 Azure Monitor Availability Test

**Purpose**: Synthetic monitoring of health check endpoint

```kusto
// Query to track availability
availabilityResults
| where name == "SFTP Connector Health Check"
| summarize 
    SuccessCount = countif(success == true),
    FailureCount = countif(success == false),
    AvgDuration = avg(duration)
    by bin(timestamp, 5m)
| render timechart
```

**Alert Rule**:
- **Name**: SFTP Connector Unhealthy
- **Condition**: 2 or more test locations fail
- **Severity**: Critical (Sev 1)
- **Action**: Page on-call engineer

### 5.2 Configuration Update Monitoring

```kusto
// Track configuration updates
traces
| where operation_Name == "ConfigurationUpdateFunction"
| where message contains "Configuration reloaded successfully"
| summarize Count = count() by PartnerCode = tostring(customDimensions.PartnerCode), bin(timestamp, 1h)
| render barchart
```

**Alert Rule**:
- **Name**: Configuration Update Failed
- **Condition**: Exception count > 0 in last 15 minutes
- **Severity**: High (Sev 2)
- **Action**: Email operations team

### 5.3 Application Insights Queries

#### Health Check Response Times

```kusto
requests
| where name == "HealthCheckFunction"
| summarize 
    P50 = percentile(duration, 50),
    P95 = percentile(duration, 95),
    P99 = percentile(duration, 99)
    by bin(timestamp, 5m)
| render timechart
```

#### Component Health Trends

```kusto
traces
| where message contains "Component health"
| extend ComponentName = tostring(customDimensions.ComponentName)
| extend Status = tostring(customDimensions.Status)
| summarize Count = count() by ComponentName, Status, bin(timestamp, 1h)
| render columnchart
```

---

## 6. Cost-Benefit Analysis

### 6.1 Implementation Costs

| Item | Hours | Hourly Rate | Cost |
|------|-------|-------------|------|
| **HealthCheckFunction** | 3 | $150 | $450 |
| **ConfigurationUpdateFunction** | 4 | $150 | $600 |
| **Testing** | 1 | $150 | $150 |
| **Deployment & Documentation** | 1 | $150 | $150 |
| **TOTAL** | **9** | - | **$1,350** |

### 6.2 Operational Costs

**Azure Resources** (per month):
- Event Grid events: ~100/month = $0.00 (free tier)
- Function executions: ~3,000/month (health checks) = $0.00 (free tier)
- Application Insights: Included in platform costs
- **Total Monthly Cost**: ~$0 (within free tiers)

### 6.3 Benefits

#### HealthCheckFunction

**Quantifiable Benefits**:
- **Reduced MTTR**: 30 minutes → 5 minutes (25 min savings per incident)
- **Proactive Detection**: Catch 80% of issues before user impact
- **Uptime SLA**: Enable 99.95% uptime monitoring

**Annual Value** (assuming 12 incidents/year):
- MTTR savings: 12 incidents × 25 min × $150/hr = **$750/year**
- Prevented downtime: 10 incidents × 30 min × $500/hr = **$2,500/year**
- **Total Annual Benefit**: **$3,250**

**ROI**: ($3,250 - $0) / $450 = **722% ROI**

#### ConfigurationUpdateFunction

**Quantifiable Benefits**:
- **Eliminated Downtime**: 8 config changes/year × 5 min downtime = 40 min/year saved
- **Reduced Deployment Time**: 8 changes × 15 min = 2 hours/year saved
- **Reduced Risk**: Hot reload prevents 90% of config-related incidents

**Annual Value**:
- Downtime avoidance: 40 min × $500/hr = **$333/year**
- Deployment efficiency: 2 hrs × $150/hr = **$300/year**
- **Total Annual Benefit**: **$633**

**ROI**: ($633 - $0) / $600 = **105% ROI**

### 6.4 Risk Mitigation Value

**Unquantifiable Benefits**:
- **Compliance**: Demonstrates system availability for audits
- **Partner SLAs**: Meet contractual uptime requirements
- **Developer Experience**: Faster troubleshooting and configuration updates
- **Operational Confidence**: Reduced stress for on-call engineers

---

## 7. Decision Matrix

### 7.1 Recommendation

**Should you implement these functions?**

| Criterion | HealthCheck | ConfigUpdate | Weight |
|-----------|-------------|--------------|--------|
| **Business Value** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 40% |
| **Technical Complexity** | ⭐⭐ | ⭐⭐⭐ | 20% |
| **Implementation Cost** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 20% |
| **Operational Impact** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 20% |
| **TOTAL SCORE** | **4.6/5** | **4.0/5** | 100% |

**Recommendation**: ✅ **Implement both functions** in Phase 2 (after MVP)

### 7.2 Implementation Priority

**Phase 1 (MVP)**: Focus on core SFTP functions (95% complete)
- SftpDownloadFunction
- SftpUploadFunction

**Phase 2 (Enhanced Operations)**: Add optional functions
- ✅ **HealthCheckFunction** (Week 10 - Day 1, 3 hours)
- ✅ **ConfigurationUpdateFunction** (Week 10 - Day 2, 4 hours)

**Phase 3 (Production Hardening)**: Monitoring & Alerts
- Azure Monitor availability tests
- Configuration change alerts
- Operational runbooks

---

## 8. Getting Started

### Quick Start (HealthCheckFunction)

```bash
# 1. Navigate to project
cd c:\repos\edi-platform\edi-sftp-connector

# 2. Create HealthCheckService.cs
# Copy implementation from Section 1.2

# 3. Create HealthCheckFunction.cs
# Copy implementation from Section 1.2

# 4. Run locally
func start --csharp

# 5. Test endpoint
curl http://localhost:7071/api/health | jq
```

### Quick Start (ConfigurationUpdateFunction)

```bash
# 1. Create ConfigUpdateService.cs
# Copy implementation from Section 2.2

# 2. Create ConfigurationUpdateFunction.cs
# Copy implementation from Section 2.2

# 3. Setup Event Grid subscription
az eventgrid event-subscription create ...

# 4. Test by uploading configuration
az storage blob upload --file ./test-config.json
```

---

## 9. Summary

### Key Takeaways

1. **Both functions are low-effort, high-value additions**
   - HealthCheckFunction: 3 hours, 722% ROI
   - ConfigurationUpdateFunction: 4 hours, 105% ROI

2. **Minimal operational costs** (~$0/month, within free tiers)

3. **Significant operational benefits**:
   - Proactive monitoring
   - Reduced downtime
   - Faster troubleshooting
   - Zero-downtime configuration updates

4. **Not blocking MVP**, but highly recommended for Phase 2

### Next Steps

1. **Review this document** with the team
2. **Prioritize implementation** in Phase 2 roadmap
3. **Allocate 1 day** (9 hours total including testing) in Week 10
4. **Setup Azure Monitor** availability tests post-deployment

---

**Document Status**: Implementation Ready  
**Last Updated**: October 7, 2025  
**Maintained By**: EDI Platform Team  
**Related Documents**:
- [18-implementation-task-list.md](./18-implementation-task-list.md)
- [edi-sftp-connector/README.md](../edi-sftp-connector/README.md)
- [08-monitoring-operations.md](./08-monitoring-operations.md)

---

## Appendix A: Code Templates

### A.1 HealthCheckFunction.cs (Complete)

```csharp
using System;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using EDI.SFTP.Services;

namespace EDI.SFTP.Functions
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

### A.2 ConfigurationUpdateFunction.cs (Complete)

```csharp
using System;
using System.Threading.Tasks;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.EventGrid;
using Microsoft.Azure.EventGrid.Models;
using Microsoft.Extensions.Logging;
using EDI.SFTP.Services;
using EDI.SFTP.Models;
using System.Text.Json;

namespace EDI.SFTP.Functions
{
    public class ConfigurationUpdateFunction
    {
        private readonly IConfigUpdateService _configUpdateService;
        private readonly ILogger<ConfigurationUpdateFunction> _logger;
        
        public ConfigurationUpdateFunction(
            IConfigUpdateService configUpdateService,
            ILogger<ConfigurationUpdateFunction> logger)
        {
            _configUpdateService = configUpdateService;
            _logger = logger;
        }
        
        [FunctionName("ConfigurationUpdateFunction")]
        public async Task Run(
            [EventGridTrigger] EventGridEvent eventGridEvent)
        {
            _logger.LogInformation(
                "Configuration update event received: {EventType}, Subject: {Subject}",
                eventGridEvent.EventType,
                eventGridEvent.Subject);
            
            try
            {
                // Parse Event Grid event data
                var blobCreatedEvent = JsonSerializer.Deserialize<StorageBlobCreatedEventData>(
                    eventGridEvent.Data.ToString());
                
                // Extract partner code from blob name
                var blobName = blobCreatedEvent.Url.Split('/')[^1];
                var partnerCode = Path.GetFileNameWithoutExtension(blobName);
                
                var configUpdateEvent = new ConfigurationUpdateEvent
                {
                    PartnerCode = partnerCode,
                    BlobUrl = blobCreatedEvent.Url,
                    ChangeType = "Updated",
                    EventTime = eventGridEvent.EventTime,
                    ChangedBy = "EventGrid"
                };
                
                await _configUpdateService.ProcessConfigurationUpdateAsync(configUpdateEvent);
                
                _logger.LogInformation(
                    "Configuration successfully reloaded for {PartnerCode}",
                    partnerCode);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex,
                    "Failed to process configuration update for event {EventId}",
                    eventGridEvent.Id);
                
                throw; // Re-throw to trigger Event Grid retry
            }
        }
    }
}
```

---

**End of Document**
