# Managed Identity Application Configuration Guide

## Overview

All Function Apps in the EDI platform have been updated to support **passwordless authentication** using Azure Managed Identities. This guide explains the configuration pattern, how to use it for local development vs. Azure deployment, and the specific settings required for each service.

## Configuration Pattern

### Fallback Pattern

All Function Apps use a **fallback pattern** that supports both local development (with connection strings) and Azure deployment (with managed identity):

```csharp
// Example: BlobServiceClient registration
services.AddSingleton(sp =>
{
    var connectionString = configuration["StorageConnection"];
    var credential = new DefaultAzureCredential();
    
    if (!string.IsNullOrEmpty(connectionString))
    {
        // Local development: use connection string
        return new BlobServiceClient(connectionString);
    }
    else
    {
        // Azure: use managed identity
        var storageUri = configuration["StorageAccountUri"];
        return new BlobServiceClient(new Uri(storageUri!), credential);
    }
});
```

**How it works:**
1. **Local Development**: Provide connection strings in `local.settings.json` → Uses connection string authentication
2. **Azure Deployment**: Leave connection strings empty or remove them, provide resource URIs → Uses managed identity authentication

### Benefits

✅ **No passwords in production** - Eliminates secrets management overhead
✅ **Automatic credential rotation** - Azure handles token refresh
✅ **HIPAA compliant** - Meets healthcare security requirements
✅ **Developer friendly** - Works seamlessly with local development tools
✅ **Simplified deployment** - No need to manage connection strings in Azure

## Configuration Settings by Function App

### 1. edi-sftp-connector

**Local Development (`local.settings.json`):**
```json
{
  "Values": {
    "StorageConnection": "DefaultEndpointsProtocol=https;AccountName=...;AccountKey=...",
    "ServiceBusConnection": "Endpoint=sb://...;SharedAccessKeyName=...;SharedAccessKey=...",
    "SqlConnectionString": "Server=(localdb)\\mssqllocaldb;Database=EDITracking;Integrated Security=true",
    "KeyVaultUri": "https://kv-edi-dev-eastus2.vault.azure.net/"
  }
}
```

**Azure Deployment (Application Settings):**
```json
{
  "StorageAccountUri": "https://stedideveastus2.blob.core.windows.net",
  "ServiceBusNamespace": "sb-edi-dev-eastus2",
  "SqlServerName": "sql-edi-dev-eastus2",
  "SqlDatabaseName": "SftpTracking",
  "KeyVaultUri": "https://kv-edi-dev-eastus2.vault.azure.net/"
}
```

**Services Updated:**
- ✅ `Program.cs` - BlobServiceClient and ServiceBusClient registration
- ✅ `SftpUploadFunction.cs` - Now injects BlobServiceClient
- ✅ `TrackingService.cs` - SQL Server with managed identity
- ✅ `KeyVaultService.cs` - Already used DefaultAzureCredential

---

### 2. InboundRouter Function

**Local Development:**
```json
{
  "Values": {
    "StorageAccountConnectionString": "DefaultEndpointsProtocol=https;AccountName=...;AccountKey=...",
    "ServiceBusConnectionString": "Endpoint=sb://...;SharedAccessKeyName=...;SharedAccessKey=..."
  }
}
```

**Azure Deployment:**
```json
{
  "StorageAccountUri": "https://stedideveastus2.blob.core.windows.net",
  "ServiceBusNamespace": "sb-edi-dev-eastus2"
}
```

**Status:** ✅ Already has managed identity support

---

### 3. EligibilityMapper Function

**Local Development:**
```json
{
  "Values": {
    "StorageOptions:ConnectionString": "DefaultEndpointsProtocol=https;AccountName=...;AccountKey=...",
    "ServiceBusOptions:ConnectionString": "Endpoint=sb://...;SharedAccessKeyName=...;SharedAccessKey=..."
  }
}
```

**Azure Deployment:**
```json
{
  "StorageAccountUri": "https://stedideveastus2.blob.core.windows.net",
  "ServiceBusNamespace": "sb-edi-dev-eastus2"
}
```

**Services Updated:**
- ✅ `Program.cs` - Added managed identity fallback pattern

---

### 4. EnrollmentMapper Function

**Local Development:**
```json
{
  "Values": {
    "StorageAccountConnectionString": "DefaultEndpointsProtocol=https;AccountName=...;AccountKey=...",
    "ServiceBusConnectionString": "Endpoint=sb://...;SharedAccessKeyName=...;SharedAccessKey=..."
  }
}
```

**Azure Deployment:**
```json
{
  "StorageAccountUri": "https://stedideveastus2.blob.core.windows.net",
  "ServiceBusNamespace": "sb-edi-dev-eastus2"
}
```

**Status:** ✅ Already has managed identity support

---

### 5. ControlNumberGenerator Function

**Local Development:**
```json
{
  "Values": {
    "SqlConnectionString": "Server=(localdb)\\mssqllocaldb;Database=ControlNumbers;Integrated Security=true"
  }
}
```

**Azure Deployment:**
```json
{
  "SqlServerName": "sql-edi-dev-eastus2",
  "SqlDatabaseName": "ControlNumbers"
}
```

**Services Updated:**
- ✅ `SqlConnectionFactory.cs` - Added managed identity token authentication
- ✅ `Program.cs` - Updated factory registration

---

### 6. FileArchiver Function

**Local Development:**
```json
{
  "Values": {
    "StorageAccountConnectionString": "DefaultEndpointsProtocol=https;AccountName=...;AccountKey=..."
  }
}
```

**Azure Deployment:**
```json
{
  "StorageAccountUri": "https://stedideveastus2.blob.core.windows.net"
}
```

**Status:** ✅ Already has managed identity support

---

### 7. NotificationService Function

**Local Development:**
```json
{
  "Values": {
    "CommunicationServicesConnectionString": "endpoint=https://...;accesskey=..."
  }
}
```

**Note:** NotificationService uses Azure Communication Services, which currently requires a connection string. Does NOT use Storage or Service Bus.

**Status:** ✅ No changes needed (different authentication method)

---

### 8. EnterpriseScheduler Function

**Local Development:**
```json
{
  "Values": {
    "StorageAccountConnectionString": "DefaultEndpointsProtocol=https;AccountName=...;AccountKey=...",
    "ServiceBusConnectionString": "Endpoint=sb://...;SharedAccessKeyName=...;SharedAccessKey=..."
  }
}
```

**Azure Deployment:**
```json
{
  "StorageAccountUri": "https://stedideveastus2.blob.core.windows.net",
  "ServiceBusNamespace": "sb-edi-dev-eastus2"
}
```

**Status:** ✅ Already has managed identity support

---

## Shared Libraries

### EDI.Storage.Services

**BlobStorageService:**
- ✅ Already architected for dependency injection
- ✅ Accepts `BlobServiceClient` via constructor
- ✅ No changes needed

**QueueStorageService:**
- ✅ Already architected for dependency injection
- ✅ Accepts `QueueServiceClient` via constructor
- ✅ No changes needed

### EDI.Messaging.Services

**ServiceBusPublisher:**
- ✅ Already architected for dependency injection
- ✅ Accepts `ServiceBusClient` via constructor
- ✅ No changes needed

**ServiceBusProcessor:**
- ✅ Already architected for dependency injection
- ✅ Accepts `ServiceBusClient` via constructor
- ✅ No changes needed

### EDI.Configuration.Services

**PartnerConfigService:**
- ✅ Already uses `DefaultAzureCredential`
- ✅ Creates `BlobServiceClient` with managed identity
- ✅ No changes needed

---

## Deployment Checklist

### Pre-Deployment

1. ✅ **Verify RBAC assignments** - Run `edi-azure-infrastructure/bicep/modules/rbac.bicep`
2. ✅ **Verify System-Assigned Managed Identity enabled** - All Function Apps must have it enabled
3. ✅ **Test locally first** - Ensure connection strings work in `local.settings.json`
4. ✅ **Review configuration keys** - Double-check resource names (storage account, service bus namespace, SQL server)

### Deployment Steps

1. **Deploy Infrastructure:**
   ```bash
   cd edi-azure-infrastructure
   az deployment group create \
     --resource-group rg-edi-dev-eastus2 \
     --template-file bicep/main.bicep \
     --parameters bicep/parameters/dev.parameters.json
   ```

2. **Configure Application Settings** (per Function App):
   ```bash
   # Example: edi-sftp-connector
   az functionapp config appsettings set \
     --name func-edi-sftp-connector-dev \
     --resource-group rg-edi-dev-eastus2 \
     --settings \
       "StorageAccountUri=https://stedideveastus2.blob.core.windows.net" \
       "ServiceBusNamespace=sb-edi-dev-eastus2" \
       "SqlServerName=sql-edi-dev-eastus2" \
       "SqlDatabaseName=SftpTracking" \
       "KeyVaultUri=https://kv-edi-dev-eastus2.vault.azure.net/"
   ```

3. **Deploy Function App Code:**
   ```bash
   cd edi-sftp-connector
   func azure functionapp publish func-edi-sftp-connector-dev
   ```

4. **Verify Deployment:**
   ```bash
   # Check Function App identity
   az functionapp identity show \
     --name func-edi-sftp-connector-dev \
     --resource-group rg-edi-dev-eastus2
   
   # Check application settings
   az functionapp config appsettings list \
     --name func-edi-sftp-connector-dev \
     --resource-group rg-edi-dev-eastus2
   
   # Check logs for authentication
   func azure functionapp logstream func-edi-sftp-connector-dev
   ```

### Post-Deployment

1. ✅ **Test each Function App** - Trigger functions and verify no authentication errors
2. ✅ **Monitor Application Insights** - Check for authentication failures or token issues
3. ✅ **Verify RBAC works** - Ensure Function Apps can access Storage, Service Bus, SQL Server
4. ✅ **Remove connection strings** - Clean up any leftover connection strings from application settings

---

## Troubleshooting

### Error: "No valid credentials found"

**Cause:** Managed identity not enabled or RBAC role not assigned

**Solution:**
```bash
# Enable managed identity
az functionapp identity assign \
  --name func-edi-sftp-connector-dev \
  --resource-group rg-edi-dev-eastus2

# Assign RBAC role (example: Storage Blob Data Contributor)
PRINCIPAL_ID=$(az functionapp identity show \
  --name func-edi-sftp-connector-dev \
  --resource-group rg-edi-dev-eastus2 \
  --query principalId -o tsv)

az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/{subscription-id}/resourceGroups/rg-edi-dev-eastus2/providers/Microsoft.Storage/storageAccounts/stedideveastus2"
```

---

### Error: "Login failed for user 'NT AUTHORITY\ANONYMOUS LOGON'"

**Cause:** SQL Server connection not using access token

**Solution:**
- Verify `SqlServerName` and `SqlDatabaseName` are configured (not `SqlConnectionString`)
- Ensure SQL Server has Azure AD authentication enabled
- Verify Function App managed identity has SQL role assigned

```bash
# Assign SQL role
az sql server ad-admin set \
  --resource-group rg-edi-dev-eastus2 \
  --server-name sql-edi-dev-eastus2 \
  --display-name "EDI Functions" \
  --object-id $PRINCIPAL_ID
```

---

### Error: "The specified blob does not exist"

**Cause:** Incorrect `StorageAccountUri` or missing RBAC permission

**Solution:**
- Verify `StorageAccountUri` format: `https://{storage-account-name}.blob.core.windows.net`
- Ensure managed identity has "Storage Blob Data Contributor" or "Storage Blob Data Reader" role
- Check container exists and blob path is correct

---

### Local Development Issues

**Error: "Connection string not found"**

**Solution:**
- Ensure `local.settings.json` exists (copy from `local.settings.sample.json`)
- Verify connection string keys match code expectations
- Use Azurite for local blob storage: `azurite --silent --location c:\azurite --debug c:\azurite\debug.log`

---

## Security Best Practices

1. ✅ **Never commit connection strings** - Use `local.settings.json` (gitignored)
2. ✅ **Use least-privilege RBAC** - Assign minimum required roles
3. ✅ **Enable diagnostic logging** - Monitor for authentication failures
4. ✅ **Rotate managed identity periodically** - Disable/re-enable if compromised
5. ✅ **Audit role assignments** - Review who has access to resources
6. ✅ **Use private endpoints** - Restrict network access to Azure resources

---

## Configuration Reference Table

| Function App | Storage | Service Bus | SQL Server | Key Vault | Status |
|--------------|---------|-------------|------------|-----------|--------|
| edi-sftp-connector | ✅ | ✅ | ✅ | ✅ | Updated |
| InboundRouter | ✅ | ✅ | - | - | Already Done |
| EligibilityMapper | ✅ | ✅ | - | - | Updated |
| EnrollmentMapper | ✅ | ✅ | - | - | Already Done |
| ControlNumberGenerator | - | - | ✅ | - | Updated |
| FileArchiver | ✅ | - | - | - | Already Done |
| NotificationService | - | - | - | - | N/A (ACS) |
| EnterpriseScheduler | ✅ | ✅ | - | - | Already Done |

---

## Additional Resources

- [Azure Managed Identity Documentation](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)
- [Azure SDK DefaultAzureCredential](https://learn.microsoft.com/en-us/dotnet/api/azure.identity.defaultazurecredential)
- [RBAC Role Assignments](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal)
- [SQL Server Managed Identity Authentication](https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity)

---

## Summary

✅ All Function Apps have been updated to support managed identity authentication
✅ Fallback pattern allows seamless local development with connection strings
✅ RBAC roles are automatically assigned via `rbac.bicep` module
✅ Configuration documented for each service
✅ Troubleshooting guide provided

**Next Steps:**
1. Deploy infrastructure with `main.bicep`
2. Configure application settings per Function App
3. Deploy Function App code
4. Test and verify authentication works
5. Remove connection strings from Azure application settings
