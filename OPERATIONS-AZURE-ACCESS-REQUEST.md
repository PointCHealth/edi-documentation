# Azure Access Request for GitHub Actions Deployment

## Purpose
This document outlines the Azure access requirements needed for automated deployment of the EDI Platform infrastructure and applications via GitHub Actions workflows.

## Overview
The EDI Platform uses GitHub Actions for CI/CD automation to deploy:
- Azure infrastructure (via Bicep/ARM templates)
- Azure Functions applications
- Database migrations
- Configuration updates

To enable these automated deployments, GitHub Actions workflows require authenticated access to Azure resources.

## Azure Environment Information

### Current Azure Subscriptions

The EDI Platform uses the following Azure subscriptions:

| Environment | Subscription Name | Subscription ID | Tenant ID |
|-------------|------------------|-----------------|------------|
| **DEV/TEST** | EDI-DEV | `0f02cf19-be55-4aab-983b-951e84910121` | `76888a14-162d-4764-8e6f-c5a34addbd87` |
| **PROD** | EDI-PROD | `85aa9a59-7b1c-49d2-84ba-0640040bc097` | `76888a14-162d-4764-8e6f-c5a34addbd87` |

### Resource Groups by Environment

**DEV Subscription (0f02cf19-be55-4aab-983b-951e84910121):**
- `rg-edi-dev-eastus2` - Development environment resources
- `rg-edi-test-eastus2` - Testing environment resources (same subscription as dev)

**PROD Subscription (85aa9a59-7b1c-49d2-84ba-0640040bc097):**
- `rg-edi-prod-eastus2` - Production environment resources

**Note:** The DEV and TEST environments share the same Azure subscription but use separate resource groups for isolation.

## Required Azure Access

### 1. Service Principal Creation
Operations must create an Azure Service Principal (App Registration) that will be used by GitHub Actions to authenticate to Azure.

**Steps:**
1. Navigate to Azure Portal → Azure Active Directory → App registrations
2. Click "New registration"
3. Configure the registration:
   - **Name**: `github-actions-edi-platform` 
   - **Supported account types**: Accounts in this organizational directory only
   - **Redirect URI**: Leave blank
4. Click "Register"
5. Note down the following values (needed for GitHub):
   - **Application (client) ID**
   - **Directory (tenant) ID**

### 2. Create Client Secret
1. In the App Registration, navigate to "Certificates & secrets"
2. Click "New client secret"
3. Configure the secret:
   - **Description**: `GitHub Actions deployment secret`
   - **Expires**: Choose appropriate expiration (recommend 12-24 months with calendar reminder)
4. Click "Add"
5. **IMPORTANT**: Copy the secret **Value** immediately (it won't be shown again)
6. Note down:
   - **Client Secret Value**

### 3. Assign Azure Permissions
The Service Principal needs permissions to deploy and manage Azure resources.

**Required Role Assignments:**

#### Subscription Level
Assign at the subscription level where EDI Platform resources will be deployed:

**For DEV/TEST Environment:**

1. Navigate to: Subscriptions → EDI-DEV (`0f02cf19-be55-4aab-983b-951e84910121`) → Access control (IAM)
2. Click "Add" → "Add role assignment"
3. Assign the following roles:
   - **Role**: `Contributor`
   - **Scope**: Resource Groups
     - `/subscriptions/0f02cf19-be55-4aab-983b-951e84910121/resourceGroups/rg-edi-dev-eastus2`
     - `/subscriptions/0f02cf19-be55-4aab-983b-951e84910121/resourceGroups/rg-edi-test-eastus2`
   - **Justification**: Deploy and manage Azure resources (Functions, Storage, Key Vault, Service Bus, etc.)

**For PROD Environment:**

1. Navigate to: Subscriptions → EDI-PROD (`85aa9a59-7b1c-49d2-84ba-0640040bc097`) → Access control (IAM)
2. Click "Add" → "Add role assignment"
3. Assign the following roles:
   - **Role**: `Contributor`
   - **Scope**: Resource Group
     - `/subscriptions/85aa9a59-7b1c-49d2-84ba-0640040bc097/resourceGroups/rg-edi-prod-eastus2`
   - **Justification**: Deploy and manage production Azure resources

**Note:** The Service Principal only needs Contributor access at the Resource Group level. Key Vault, Storage Account, and other service-specific permissions will be automatically configured by the infrastructure deployment workflows using Azure managed identities.

### 4. Provide Credentials to Development Team

After completing the Service Principal setup and permission assignments, provide the following information to the development team:

**Information to Provide:**

```yaml
AZURE_TENANT_ID: 76888a14-162d-4764-8e6f-c5a34addbd87
AZURE_CLIENT_ID: [Application (client) ID from step 1]
AZURE_CLIENT_SECRET: [Client Secret Value from step 2]
AZURE_SUBSCRIPTION_ID_DEV: 0f02cf19-be55-4aab-983b-951e84910121
AZURE_SUBSCRIPTION_ID_PROD: 85aa9a59-7b1c-49d2-84ba-0640040bc097
```

**Delivery Method:**
- Use secure communication channel (e.g., encrypted email, password manager, Azure Key Vault)
- Never send credentials via plain text email or Slack
- Consider using a time-limited secure sharing service

**What the Development Team Will Do:**

The development team will use this information to configure GitHub repository secrets and variables across all EDI Platform repositories. Detailed instructions for the development team are available in:

→ **[GITHUB-SECRETS-CONFIGURATION.md](./GITHUB-SECRETS-CONFIGURATION.md)**

**Repositories That Need Configuration:**
- edi-azure-infrastructure
- edi-sftp-connector
- edi-platform-core
- edi-mappers
- edi-database-controlnumbers
- edi-database-eventstore
- edi-database-sftptracking

### 5. Additional Permissions

**Important:** The Contributor role at the Resource Group level is sufficient for all deployment needs. The following permissions are handled automatically:

#### Automatically Configured (No Action Needed)
These are managed by the infrastructure workflows using Azure managed identities:
- ✅ **Key Vault access**: Configured via Key Vault access policies for Function Apps
- ✅ **Storage Account access**: Configured via Azure RBAC for Function Apps
- ✅ **Service Bus access**: Configured via Azure RBAC for Function Apps
- ✅ **SQL Database access**: Configured via Azure AD authentication for Function Apps
- ✅ **Application Insights**: Automatic integration via connection strings

#### Optional: Reader Access for Monitoring (Recommended)
Grant subscription-level Reader access for cost monitoring and resource health workflows:
- **Role**: `Reader`
- **Scope**: Subscription level
- **Justification**: Allows GitHub Actions to query costs, resource health, and deployment status

## Security Considerations

### Least Privilege Principle
- Start with minimum required permissions
- Grant additional permissions only as needed
- Use Resource Group scope instead of Subscription scope when possible

### Secret Rotation
- Set calendar reminders for secret expiration
- Rotate secrets before expiration to avoid deployment failures
- Update GitHub secrets after rotation

### Audit and Monitoring
- Enable Azure AD sign-in logs for the Service Principal
- Monitor Azure Activity Log for actions performed by the Service Principal
- Set up alerts for suspicious activities

### Network Security
- Consider using Managed Identity instead of Service Principal where possible
- Implement Conditional Access policies if your organization requires it
- Use GitHub Enterprise if requiring IP restrictions

## Validation Steps

After configuration, validate the setup:

1. **Test Authentication:**
   ```bash
   az login --service-principal \
     -u <AZURE_CLIENT_ID> \
     -p <AZURE_CLIENT_SECRET> \
     --tenant <AZURE_TENANT_ID>
   ```

2. **Test Permissions (DEV):**
   ```bash
   az account set --subscription 0f02cf19-be55-4aab-983b-951e84910121
   az group list
   az group show --name rg-edi-dev-eastus2
   ```

3. **Test Permissions (PROD):**
   ```bash
   az account set --subscription 85aa9a59-7b1c-49d2-84ba-0640040bc097
   az group list
   az group show --name rg-edi-prod-eastus2
   ```

3. **Run a Test Workflow:**
   - Trigger a GitHub Actions workflow manually
   - Monitor the workflow logs for authentication success
   - Verify resources are deployed correctly

## Troubleshooting

### Common Issues

**Authentication Failed:**
- Verify client ID, tenant ID, and secret are correct
- Check if secret has expired
- Ensure Service Principal is not disabled

**Permission Denied:**
- Verify role assignments are applied at correct scope
- Check if role assignment propagation is complete (can take 5-10 minutes)
- Ensure Service Principal has necessary permissions for the specific operation

**Subscription Not Found:**
- Verify subscription ID is correct
- Check if Service Principal has Reader access to subscription

## Reference Information

### Azure Resource Groups
The EDI Platform uses the following resource groups :
- `rg-edi-platform-dev`
- `rg-edi-platform-test`
- `rg-edi-platform-prod`


## Checklist for Operations Team

- [ ] Create Service Principal in Azure AD
- [ ] Generate and securely store client secret
- [ ] Assign Contributor role at appropriate scope
- [ ] Document the following and provide to development team:
  - [ ] AZURE_TENANT_ID: `76888a14-162d-4764-8e6f-c5a34addbd87`
  - [ ] AZURE_SUBSCRIPTION_ID_DEV: `0f02cf19-be55-4aab-983b-951e84910121`
  - [ ] AZURE_SUBSCRIPTION_ID_PROD: `85aa9a59-7b1c-49d2-84ba-0640040bc097`
  - [ ] AZURE_CLIENT_ID: [Generated during Service Principal creation]
  - [ ] AZURE_CLIENT_SECRET: [Generated during secret creation]
- [ ] Set calendar reminder for secret expiration
- [ ] Test authentication using Azure CLI
- [ ] Notify development team when complete


**Document Version**: 1.0  
**Last Updated**: October 7, 2025  
**Next Review**: Upon secret expiration or major platform changes
