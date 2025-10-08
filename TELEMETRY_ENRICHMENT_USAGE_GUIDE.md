# EDI Platform - Telemetry Enrichment Usage Guide

**Document Version:** 1.0  
**Date:** October 7, 2025  
**Status:** Implementation Complete  
**Purpose:** Guide for implementing custom telemetry dimensions across all Azure Functions

---

## Executive Summary

This document provides step-by-step instructions for implementing custom telemetry dimensions in all EDI Platform Azure Functions. The `Argus.Logging` library (version 1.1.0) now includes comprehensive support for 17 custom dimensions required for ABC availability reports and operational monitoring.

**Implementation Status:**
- ✅ **Argus.Logging Library**: 100% Complete (Enhanced EDIContext + TelemetryEnricher)
- 🔄 **Function Apps**: 0% Complete (Awaiting integration)

**Key Benefits:**
- **ABC Report Compliance**: All custom dimensions required for availability tracking
- **Partner SLA Monitoring**: Track processing times and success rates by partner
- **Event Sourcing Observability**: Monitor projection lag and event flow
- **Operational Visibility**: Filter telemetry by partner, transaction type, file status, etc.

---

## Table of Contents

1. [Custom Dimensions Reference](#1-custom-dimensions-reference)
2. [SFTP Connector Functions](#2-sftp-connector-functions)
3. [Platform Core Functions](#3-platform-core-functions)
4. [Mapper Functions](#4-mapper-functions)
5. [Testing Telemetry](#5-testing-telemetry)
6. [Application Insights Queries](#6-application-insights-queries)

---

## 1. Custom Dimensions Reference

The following custom dimensions are automatically added to all Application Insights telemetry when `EDIContext` is set:

| Dimension | Purpose | Example Values | Used By |
|-----------|---------|----------------|---------|
| `CorrelationId` | Distributed tracing | Auto-generated | All functions |
| `PartnerCode` | Trading partner ID | "PARTNERA", "BCBS-IL" | All functions |
| `TransactionType` | X12 transaction code | "270", "271", "834", "837", "835" | Mappers, Router |
| `FileName` | File being processed | "eligibility_20251007.x12" | SFTP, Router |
| `BatchId` | Batch identifier | "BATCH123" | Mappers |
| `EventSequence` | Event sourcing seq | "1001" | Projection Builders |
| `FunctionName` | Azure Function name | "EligibilityMapper" | All functions |
| `FileStatus` | Processing status | "Received", "Validated", "Processing", "Completed", "Failed" | SFTP, Router |
| `ValidationStatus` | Validation result | "Valid", "Invalid", "Warning" | Mappers |
| `ProcessingTimeMs` | Duration (SLA) | "1250" | All functions |
| `EventType` | Event sourcing type | "ClaimSubmitted", "RemittanceReceived" | Projection Builders |
| `ControlNumberType` | Control number type | "ISA", "GS", "ST" | ControlNumberGenerator |
| `JobName` | Scheduled job name | "DailyEnrollmentExport" | EnterpriseScheduler |
| `ExecutionStatus` | Job execution result | "Success", "Failed", "Timeout" | EnterpriseScheduler |
| `NotificationType` | Notification type | "Alert", "Info", "Error" | NotificationService |
| `Severity` | Severity level | "Critical", "Warning", "Info" | NotificationService |
| `Channel` | Notification channel | "Email", "Teams", "SolarWinds" | NotificationService |

---

## 2. SFTP Connector Functions

**Repository**: `edi-sftp-connector`

### 2.1 SftpDownloadFunction

**File**: `Functions/SftpDownloadFunction.cs`

**Changes Required:**

1. Add `using Argus.Logging.Models;` to imports
2. Wrap timer execution in EDI context scope
3. Update context as file processing progresses

**Implementation:**

```csharp
using Argus.Logging.Models;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;

public class SftpDownloadFunction
{
    private readonly ILogger<SftpDownloadFunction> _logger;
    private readonly ISftpService _sftpService;

    [Function("SftpDownload")]
    public async Task Run([TimerTrigger("0 */5 * * * *")] TimerInfo timer)
    {
        var partners = await GetPartnersAsync();
        
        foreach (var partner in partners)
        {
            var startTime = DateTime.UtcNow;
            
            // Create EDI context for this partner
            using var ediScope = EDIContext.CreateScope()
                .WithFunctionName("SftpDownload")
                .WithPartner(partner.Code)
                .WithFileStatus("Connecting")
                .Begin();
            
            try
            {
                _logger.LogInformation("Connecting to SFTP for partner {PartnerCode}", partner.Code);
                
                var files = await _sftpService.ListFilesAsync(partner);
                
                foreach (var file in files)
                {
                    // Update context for each file
                    if (EDIContext.Current != null)
                    {
                        EDIContext.Current.FileName = file.Name;
                        EDIContext.Current.FileStatus = "Downloading";
                    }
                    
                    _logger.LogInformation("Downloading file {FileName}", file.Name);
                    
                    var content = await _sftpService.DownloadFileAsync(partner, file.Name);
                    
                    // Update status
                    if (EDIContext.Current != null)
                    {
                        EDIContext.Current.FileStatus = "Uploading";
                    }
                    
                    await UploadToBlobAsync(content, partner.Code, file.Name);
                    
                    // Mark complete
                    if (EDIContext.Current != null)
                    {
                        EDIContext.Current.FileStatus = "Completed";
                        EDIContext.Current.ProcessingTimeMs = 
                            (long)(DateTime.UtcNow - startTime).TotalMilliseconds;
                    }
                    
                    _logger.LogInformation("File {FileName} processed in {DurationMs}ms", 
                        file.Name, EDIContext.Current?.ProcessingTimeMs);
                }
            }
            catch (Exception ex)
            {
                if (EDIContext.Current != null)
                {
                    EDIContext.Current.FileStatus = "Failed";
                }
                _logger.LogError(ex, "SFTP download failed for partner {PartnerCode}", partner.Code);
            }
        }
    }
}
```

### 2.2 SftpUploadFunction

**File**: `Functions/SftpUploadFunction.cs`

**Implementation:**

```csharp
using Argus.Logging.Models;

public class SftpUploadFunction
{
    [Function("SftpUpload")]
    public async Task Run(
        [ServiceBusTrigger("sftp-upload-queue")] string message)
    {
        var startTime = DateTime.UtcNow;
        var request = JsonSerializer.Deserialize<UploadRequest>(message);
        
        using var ediScope = EDIContext.CreateScope()
            .WithFunctionName("SftpUpload")
            .WithPartner(request.PartnerCode)
            .WithFileName(request.FileName)
            .WithFileStatus("Pending")
            .Begin();
        
        try
        {
            _logger.LogInformation("Uploading file {FileName} to partner {PartnerCode}", 
                request.FileName, request.PartnerCode);
            
            // Download from blob
            if (EDIContext.Current != null)
            {
                EDIContext.Current.FileStatus = "Downloading";
            }
            
            var content = await DownloadFromBlobAsync(request.BlobPath);
            
            // Upload to SFTP
            if (EDIContext.Current != null)
            {
                EDIContext.Current.FileStatus = "Uploading";
            }
            
            await _sftpService.UploadFileAsync(request.PartnerCode, request.FileName, content);
            
            // Mark complete
            if (EDIContext.Current != null)
            {
                EDIContext.Current.FileStatus = "Completed";
                EDIContext.Current.ProcessingTimeMs = 
                    (long)(DateTime.UtcNow - startTime).TotalMilliseconds;
            }
            
            _logger.LogInformation("Upload completed successfully in {DurationMs}ms", 
                EDIContext.Current?.ProcessingTimeMs);
        }
        catch (Exception ex)
        {
            if (EDIContext.Current != null)
            {
                EDIContext.Current.FileStatus = "Failed";
            }
            _logger.LogError(ex, "Upload failed");
            throw;
        }
    }
}
```

---

## 3. Platform Core Functions

**Repository**: `edi-platform-core/functions`

### 3.1 InboundRouter.Function

**File**: `Functions/InboundRouterFunction.cs`

**Custom Dimensions Used:**
- `PartnerCode`, `TransactionType`, `FileName`, `FileStatus`, `ProcessingTimeMs`

**Implementation:**

```csharp
using Argus.Logging.Models;

public class InboundRouterFunction
{
    [Function("RouteFile")]
    public async Task Run([BlobTrigger("inbound/{name}")] Stream fileStream, string name)
    {
        var startTime = DateTime.UtcNow;
        
        using var ediScope = EDIContext.CreateScope()
            .WithFunctionName("InboundRouter")
            .WithFileName(name)
            .WithFileStatus("Received")
            .Begin();
        
        try
        {
            _logger.LogInformation("Routing file {FileName}", name);
            
            // Parse X12 envelope
            var envelope = await _x12Parser.ParseEnvelopeAsync(fileStream);
            
            // Update context with parsed data
            if (EDIContext.Current != null)
            {
                EDIContext.Current.PartnerCode = envelope.SenderId;
                EDIContext.Current.TransactionType = envelope.TransactionSetCode;
                EDIContext.Current.FileStatus = "Routing";
            }
            
            _logger.LogInformation("Parsed {TransactionType} from {PartnerCode}", 
                envelope.TransactionSetCode, envelope.SenderId);
            
            // Route to appropriate queue
            await _routingService.RouteAsync(envelope);
            
            if (EDIContext.Current != null)
            {
                EDIContext.Current.FileStatus = "Routed";
                EDIContext.Current.ProcessingTimeMs = 
                    (long)(DateTime.UtcNow - startTime).TotalMilliseconds;
            }
            
            _logger.LogInformation("File routed successfully in {DurationMs}ms", 
                EDIContext.Current?.ProcessingTimeMs);
        }
        catch (Exception ex)
        {
            if (EDIContext.Current != null)
            {
                EDIContext.Current.FileStatus = "Failed";
            }
            _logger.LogError(ex, "Routing failed");
            throw;
        }
    }
}
```

### 3.2 ControlNumberGenerator.Function

**File**: `Functions/ControlNumberGeneratorFunction.cs`

**Custom Dimensions Used:**
- `PartnerCode`, `ControlNumberType`, `ProcessingTimeMs`

**Implementation:**

```csharp
using Argus.Logging.Models;

public class ControlNumberGeneratorFunction
{
    [Function("GetControlNumber")]
    public async Task<HttpResponseData> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req)
    {
        var startTime = DateTime.UtcNow;
        var request = await req.ReadFromJsonAsync<ControlNumberRequest>();
        
        using var ediScope = EDIContext.CreateScope()
            .WithFunctionName("ControlNumberGenerator")
            .WithPartner(request.PartnerCode)
            .WithControlNumberType(request.Type)
            .Begin();
        
        try
        {
            _logger.LogInformation("Generating {ControlNumberType} for {PartnerCode}", 
                request.Type, request.PartnerCode);
            
            var controlNumber = await _controlNumberService.GenerateAsync(request);
            
            if (EDIContext.Current != null)
            {
                EDIContext.Current.ProcessingTimeMs = 
                    (long)(DateTime.UtcNow - startTime).TotalMilliseconds;
            }
            
            _logger.LogInformation("Generated control number: {ControlNumber}", controlNumber);
            
            var response = req.CreateResponse(HttpStatusCode.OK);
            await response.WriteAsJsonAsync(new { ControlNumber = controlNumber });
            return response;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Control number generation failed");
            throw;
        }
    }
}
```

### 3.3 EnterpriseScheduler.Function

**File**: `Functions/EnterpriseSchedulerFunction.cs`

**Custom Dimensions Used:**
- `JobName`, `ExecutionStatus`, `ProcessingTimeMs`

**Implementation:**

```csharp
using Argus.Logging.Models;

public class EnterpriseSchedulerFunction
{
    [Function("EnterpriseScheduler")]
    public async Task Run([TimerTrigger("0 * * * * *")] TimerInfo timer)
    {
        var schedules = await LoadSchedulesAsync();
        var dueJobs = schedules.Where(s => s.IsDue()).ToList();
        
        foreach (var job in dueJobs)
        {
            var startTime = DateTime.UtcNow;
            
            using var ediScope = EDIContext.CreateScope()
                .WithFunctionName("EnterpriseScheduler")
                .WithJobName(job.Name)
                .WithExecutionStatus("Running")
                .Begin();
            
            try
            {
                _logger.LogInformation("Executing job {JobName}", job.Name);
                
                await ExecuteJobAsync(job);
                
                if (EDIContext.Current != null)
                {
                    EDIContext.Current.ExecutionStatus = "Success";
                    EDIContext.Current.ProcessingTimeMs = 
                        (long)(DateTime.UtcNow - startTime).TotalMilliseconds;
                }
                
                _logger.LogInformation("Job {JobName} completed in {DurationMs}ms", 
                    job.Name, EDIContext.Current?.ProcessingTimeMs);
            }
            catch (TimeoutException)
            {
                if (EDIContext.Current != null)
                {
                    EDIContext.Current.ExecutionStatus = "Timeout";
                }
                _logger.LogWarning("Job {JobName} timed out", job.Name);
            }
            catch (Exception ex)
            {
                if (EDIContext.Current != null)
                {
                    EDIContext.Current.ExecutionStatus = "Failed";
                }
                _logger.LogError(ex, "Job {JobName} failed", job.Name);
            }
        }
    }
}
```

### 3.4 NotificationService.Function

**File**: `Functions/NotificationServiceFunction.cs`

**Custom Dimensions Used:**
- `NotificationType`, `Severity`, `Channel`, `ProcessingTimeMs`

**Implementation:**

```csharp
using Argus.Logging.Models;

public class NotificationServiceFunction
{
    [Function("NotificationService")]
    public async Task Run(
        [ServiceBusTrigger("notification-requests")] string message)
    {
        var startTime = DateTime.UtcNow;
        var request = JsonSerializer.Deserialize<NotificationRequest>(message);
        
        using var ediScope = EDIContext.CreateScope()
            .WithFunctionName("NotificationService")
            .WithNotificationType(request.Type)
            .WithSeverity(request.Severity)
            .WithChannel(request.Channel)
            .Begin();
        
        try
        {
            _logger.LogInformation("Sending {NotificationType} notification via {Channel}", 
                request.Type, request.Channel);
            
            await _notificationRouter.SendAsync(request);
            
            if (EDIContext.Current != null)
            {
                EDIContext.Current.ProcessingTimeMs = 
                    (long)(DateTime.UtcNow - startTime).TotalMilliseconds;
            }
            
            _logger.LogInformation("Notification sent successfully in {DurationMs}ms", 
                EDIContext.Current?.ProcessingTimeMs);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to send notification");
            throw;
        }
    }
}
```

---

## 4. Mapper Functions

**Repository**: `edi-mappers/functions`

### 4.1 EligibilityMapper.Function

**File**: `Functions/EligibilityMapperFunction.cs`

**Custom Dimensions Used:**
- `PartnerCode`, `TransactionType`, `FileName`, `ValidationStatus`, `ProcessingTimeMs`

**Implementation:**

```csharp
using Argus.Logging.Models;

public class EligibilityMapperFunction
{
    [Function("EligibilityMapper")]
    public async Task Run(
        [ServiceBusTrigger("eligibility-mapper-queue")] string message)
    {
        var startTime = DateTime.UtcNow;
        
        using var ediScope = EDIContext.CreateScope()
            .WithFunctionName("EligibilityMapper")
            .WithTransactionType("270")
            .WithValidationStatus("Pending")
            .Begin();
        
        try
        {
            var request = JsonSerializer.Deserialize<MapperRequest>(message);
            
            // Update context with request data
            if (EDIContext.Current != null)
            {
                EDIContext.Current.PartnerCode = request.PartnerCode;
                EDIContext.Current.FileName = request.FileName;
            }
            
            _logger.LogInformation("Mapping eligibility request from {PartnerCode}", 
                request.PartnerCode);
            
            // Validate X12
            var validationResult = await _validator.ValidateAsync(request.X12Content);
            
            if (EDIContext.Current != null)
            {
                EDIContext.Current.ValidationStatus = validationResult.IsValid ? "Valid" : "Invalid";
            }
            
            if (!validationResult.IsValid)
            {
                _logger.LogWarning("Validation failed: {Errors}", 
                    string.Join(", ", validationResult.Errors));
                throw new ValidationException("X12 validation failed");
            }
            
            // Map transaction
            var mappedData = await _mapper.MapAsync(request.X12Content);
            
            // Save result
            await SaveMappedDataAsync(mappedData);
            
            if (EDIContext.Current != null)
            {
                EDIContext.Current.TransactionType = "271"; // Response
                EDIContext.Current.ProcessingTimeMs = 
                    (long)(DateTime.UtcNow - startTime).TotalMilliseconds;
            }
            
            _logger.LogInformation("Mapping completed in {DurationMs}ms", 
                EDIContext.Current?.ProcessingTimeMs);
        }
        catch (Exception ex)
        {
            if (EDIContext.Current != null && EDIContext.Current.ValidationStatus == "Pending")
            {
                EDIContext.Current.ValidationStatus = "Invalid";
            }
            _logger.LogError(ex, "Mapping failed");
            throw;
        }
    }
}
```

### 4.2 ClaimsMapper.Function, EnrollmentMapper.Function, RemittanceMapper.Function

Apply similar patterns:

1. Create EDI context scope at function entry
2. Set `FunctionName`, `TransactionType`, `PartnerCode`
3. Update `ValidationStatus` after validation
4. Set `ProcessingTimeMs` before completion
5. Update status to "Invalid" or "Failed" in catch blocks

---

## 5. Testing Telemetry

### 5.1 Local Testing with Application Insights

1. **Configure local.settings.json:**

```json
{
  "Values": {
    "APPLICATIONINSIGHTS_CONNECTION_STRING": "InstrumentationKey=...;IngestionEndpoint=...",
    "AzureWebJobsStorage": "UseDevelopmentStorage=true"
  }
}
```

2. **Run function locally:**

```powershell
cd c:\repos\edi-platform\edi-sftp-connector
func start
```

3. **Trigger the function** (timer, HTTP, or Service Bus)

4. **Verify telemetry in Application Insights:**
   - Open Azure Portal → Application Insights → Logs
   - Wait 1-2 minutes for ingestion
   - Run query:

```kusto
traces
| where timestamp > ago(5m)
| where customDimensions.FunctionName == "SftpDownload"
| project timestamp, message, 
    PartnerCode = tostring(customDimensions.PartnerCode),
    FileName = tostring(customDimensions.FileName),
    FileStatus = tostring(customDimensions.FileStatus)
| order by timestamp desc
```

### 5.2 Verify All Custom Dimensions

```kusto
// List all custom dimensions in the last hour
traces
| where timestamp > ago(1h)
| mvexpand customDimensions
| summarize count() by tostring(key)
| where key startswith "Partner" or key startswith "Transaction" 
    or key startswith "File" or key startswith "Validation"
    or key startswith "Processing" or key startswith "Event"
    or key startswith "Control" or key startswith "Job"
    or key startswith "Execution" or key startswith "Notification"
    or key startswith "Severity" or key startswith "Channel"
| order by key asc
```

Expected output:

```
| key                  | count_  |
|----------------------|---------|
| Channel              | 45      |
| ControlNumberType    | 12      |
| EventSequence        | 89      |
| EventType            | 89      |
| ExecutionStatus      | 23      |
| FileName             | 156     |
| FileStatus           | 156     |
| FunctionName         | 450     |
| JobName              | 23      |
| NotificationType     | 45      |
| PartnerCode          | 450     |
| ProcessingTimeMs     | 450     |
| Severity             | 45      |
| TransactionType      | 245     |
| ValidationStatus     | 112     |
```

---

## 6. Application Insights Queries

### 6.1 ABC Availability Report (Query 1)

```kusto
let timeRange = ago(24h);
requests
| where timestamp >= timeRange
| where customDimensions.PartnerCode != ""
| extend Partner = tostring(customDimensions.PartnerCode),
         TransactionType = tostring(customDimensions.TransactionType),
         Success = success == true
| summarize 
    TotalRequests = count(),
    SuccessfulRequests = countif(Success),
    FailedRequests = countif(not(Success)),
    AvailabilityPercent = round(100.0 * countif(Success) / count(), 2),
    AvgDurationMs = round(avg(duration), 2),
    P95DurationMs = round(percentile(duration, 95), 2)
  by Partner, TransactionType
| where TotalRequests > 0
| order by AvailabilityPercent asc, TotalRequests desc
```

### 6.2 File Processing Status (Query 20)

```kusto
traces
| where timestamp > ago(1h)
| where customDimensions.FileStatus != ""
| extend 
    Partner = tostring(customDimensions.PartnerCode),
    FileName = tostring(customDimensions.FileName),
    FileStatus = tostring(customDimensions.FileStatus),
    FunctionName = tostring(customDimensions.FunctionName)
| summarize count() by Partner, FileStatus, FunctionName
| order by count_ desc
```

### 6.3 Transaction Processing SLA (Query 5)

```kusto
traces
| where timestamp > ago(1d)
| where customDimensions.ProcessingTimeMs != ""
| extend 
    Partner = tostring(customDimensions.PartnerCode),
    TransactionType = tostring(customDimensions.TransactionType),
    DurationMs = tolong(customDimensions.ProcessingTimeMs)
| summarize 
    Count = count(),
    AvgMs = round(avg(DurationMs), 2),
    P50Ms = round(percentile(DurationMs, 50), 2),
    P95Ms = round(percentile(DurationMs, 95), 2),
    P99Ms = round(percentile(DurationMs, 99), 2),
    MaxMs = max(DurationMs),
    SLA_Violations = countif(DurationMs > 5000)  // 5 second SLA
  by Partner, TransactionType
| order by SLA_Violations desc
```

### 6.4 Partner Activity Summary (Query 31)

```kusto
let timeRange = ago(7d);
traces
| where timestamp >= timeRange
| where customDimensions.PartnerCode != ""
| extend 
    Partner = tostring(customDimensions.PartnerCode),
    TransactionType = tostring(customDimensions.TransactionType),
    FileStatus = tostring(customDimensions.FileStatus)
| summarize 
    TotalTransactions = count(),
    UniqueTransactionTypes = dcount(TransactionType),
    SuccessfulFiles = countif(FileStatus == "Completed"),
    FailedFiles = countif(FileStatus == "Failed"),
    LastActivity = max(timestamp)
  by Partner
| extend DaysSinceLastActivity = datetime_diff('day', now(), LastActivity)
| order by TotalTransactions desc
```

---

## 7. Next Steps

### 7.1 Implementation Checklist

- [ ] **SFTP Connector Functions** (4 hours)
  - [ ] SftpDownloadFunction - Add EDI context
  - [ ] SftpUploadFunction - Add EDI context
  - [ ] Test locally with Application Insights
  - [ ] Verify custom dimensions in telemetry
  
- [ ] **Platform Core Functions** (6 hours)
  - [ ] InboundRouter.Function - Add EDI context
  - [ ] ControlNumberGenerator.Function - Add EDI context
  - [ ] EnterpriseScheduler.Function - Add EDI context
  - [ ] FileArchiver.Function - Add EDI context
  - [ ] NotificationService.Function - Add EDI context
  - [ ] PaymentProjectionBuilder.Function - Add EDI context
  - [ ] Test each function locally
  
- [ ] **Mapper Functions** (4 hours)
  - [ ] EligibilityMapper.Function - Add EDI context
  - [ ] ClaimsMapper.Function - Add EDI context
  - [ ] EnrollmentMapper.Function - Add EDI context
  - [ ] RemittanceMapper.Function - Add EDI context
  - [ ] Test each mapper locally

### 7.2 Deployment Steps

1. **Publish Argus.Logging 1.1.0** to GitHub Packages
2. **Update consumer repositories** to use Argus.Logging 1.1.0
3. **Deploy to Dev environment** and verify telemetry
4. **Run ABC queries** to validate data collection
5. **Deploy to Test/Prod** after validation

### 7.3 Documentation Updates

- [ ] Update function README files with telemetry examples
- [ ] Add telemetry section to deployment guides
- [ ] Update 08-monitoring-operations.md with custom dimension usage
- [ ] Document troubleshooting for missing custom dimensions

---

## 8. Support & Troubleshooting

### Common Issues

**Issue**: Custom dimensions not appearing in Application Insights

**Solution**:
1. Verify `AddEDITelemetry()` is called in `Program.cs`
2. Check Application Insights connection string is configured
3. Ensure EDI context scope is created before logging
4. Wait 1-2 minutes for telemetry ingestion

**Issue**: `ProcessingTimeMs` always zero

**Solution**:
- Capture `startTime` at function entry
- Calculate duration before setting context: `(DateTime.UtcNow - startTime).TotalMilliseconds`
- Set `ProcessingTimeMs` in `finally` block to ensure it's always set

**Issue**: `PartnerCode` not set

**Solution**:
- Ensure partner code is extracted from X12 envelope or message properties
- Check partner configuration is loaded correctly
- Verify partner code is set early in function execution

---

## Appendix A: Complete Program.cs Example

**File**: `edi-sftp-connector/Program.cs`

```csharp
using Argus.Configuration.Extensions;
using Argus.Logging.Extensions;
using Argus.Storage.Extensions;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var host = new HostBuilder()
    .ConfigureFunctionsWebApplication()
    .ConfigureServices(services =>
    {
        // Application Insights
        services.AddApplicationInsightsTelemetryWorkerService();
        services.ConfigureFunctionsApplicationInsights();
        
        // EDI Telemetry Enrichment - REQUIRED FOR ABC REPORTS
        services.AddEDITelemetry();
        
        // Other services
        services.AddEDIConfiguration();
        services.AddEDIStorage();
        services.AddSingleton<ISftpService, SftpService>();
    })
    .Build();

host.Run();
```

---

**Document Status**: Ready for implementation  
**Next Action**: Implement EDI context in Function Apps (Section 7.1)  
**Blocking Issues**: None - Argus.Logging 1.1.0 is complete and ready to use
