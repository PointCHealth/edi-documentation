# Argus Administration Portal App Registration Setup

## Purpose
This document outlines the Azure Active Directory (Azure AD) App Registration setup required for the Argus Administration Portal web application. This enables secure user authentication and role-based access control (RBAC) for portal administrators.

## Overview
The Argus Administration Portal is a web-based application that provides administrative capabilities for managing the EDI Platform. The portal requires Azure AD integration for:
- User authentication (Single Sign-On)
- Role-based authorization
- Secure API access to backend services

## Azure Environment Information

### Target Azure Subscription

| Environment | Subscription Name | Subscription ID | Tenant ID |
|-------------|------------------|-----------------|------------|
| **DEV/TEST** | EDI-DEV | `0f02cf19-be55-4aab-983b-951e84910121` | `76888a14-162d-4764-8e6f-c5a34addbd87` |

### Target Resource Group
- `rg-edi-dev-eastus2` - Development environment (includes Argus Portal App Service)

### Application URL
- **DEV**: `https://app-argus-portal-dev-eastus2.azurewebsites.net`
- **TEST**: `https://app-argus-portal-test-eastus2.azurewebsites.net` (when deployed)

## Required Azure Access

### Prerequisites
- Azure AD Administrator role or Application Administrator role
- Access to EDI-DEV subscription

## Step 1: Create App Registration

1. **Navigate to Azure Active Directory**
   - Sign in to [Azure Portal](https://portal.azure.com)
   - Navigate to: **Azure Active Directory** → **App registrations**

2. **Create New Registration**
   - Click **"New registration"**
   - Configure the registration:
     - **Name**: `argus-administration-portal-dev`
     - **Supported account types**: **Accounts in this organizational directory only (Single tenant)**
     - **Redirect URI**: 
       - Type: **Web**
       - URI: `https://app-argus-portal-dev-eastus2.azurewebsites.net/signin-oidc`
   - Click **"Register"**

3. **Record Important Values**
   
   After registration, note down the following from the Overview page:
   
   ```yaml
   Application (client) ID: [Copy from Overview page]
   Directory (tenant) ID: 76888a14-162d-4764-8e6f-c5a34addbd87
   ```

## Step 2: Configure Authentication Settings

1. **Navigate to Authentication**
   - In the App Registration, go to **Authentication** (left menu)

2. **Add Additional Redirect URIs**
   - Click **"Add URI"** under Web platform
   - Add the following URIs:
     ```
     https://app-argus-portal-dev-eastus2.azurewebsites.net/signin-oidc
     https://localhost:5001/signin-oidc
     https://localhost:7001/signin-oidc
     ```
     *(The localhost URIs enable local development)*

3. **Configure Front-channel Logout URL**
   - Under **Front-channel logout URL**, enter:
     ```
     https://app-argus-portal-dev-eastus2.azurewebsites.net/signout-oidc
     ```

4. **Configure Token Settings**
   - Under **Implicit grant and hybrid flows**, ensure nothing is checked (modern auth uses authorization code flow)
   - Under **Advanced settings**:
     - **Allow public client flows**: **No**
     - **Enable the following mobile and desktop flows**: Leave unchecked

5. **Click "Save"**

## Step 3: Configure API Permissions

The portal needs permissions to sign users in and read their profile.

1. **Navigate to API Permissions**
   - In the App Registration, go to **API permissions** (left menu)

2. **Verify Default Permissions**
   - You should see `Microsoft Graph` → `User.Read` (Delegated)
   - This is sufficient for basic authentication

3. **Add Optional Permissions** (if needed for future features)
   - Click **"Add a permission"**
   - Select **Microsoft Graph**
   - Select **Delegated permissions**
   - Add:
     - `email`
     - `profile`
     - `openid` (usually already present)
   - Click **"Add permissions"**

4. **Grant Admin Consent**
   - Click **"Grant admin consent for [Your Organization]"**
   - Click **"Yes"** to confirm
   - Verify all permissions show green checkmarks under "Status"

## Step 4: Define Application Roles

Create custom application roles for role-based access control within the portal.

1. **Navigate to App Roles**
   - In the App Registration, go to **App roles** (left menu)

2. **Create Administrator Role**
   - Click **"Create app role"**
   - Configure the role:
     ```
     Display name: EDI Argus Administrator
     Value: edi-argus-administrator
     Description: Full administrative access to Argus Administration Portal for managing EDI platform configurations, trading partners, and system settings
     Allowed member types: Users/Groups
     ```
   - **Enable this app role**: Check the box
   - Click **"Apply"**

3. **Create Additional Roles** (Optional - for future use)
   
   You may want to create additional roles for different permission levels:
   
   **Read-Only Role:**
   - Click **"Create app role"**
   - Configure:
     ```
     Display name: EDI Argus Viewer
     Value: edi-argus-viewer
     Description: Read-only access to view EDI platform configurations and trading partner information without modification capabilities
     Allowed member types: Users/Groups
     ```
   - Click **"Apply"**

   **Operator Role:**
   - Click **"Create app role"**
   - Configure:
     ```
     Display name: EDI Argus Operator
     Value: edi-argus-operator
     Description: Operational access to manage EDI transactions and monitor platform health without changing core configurations
     Allowed member types: Users/Groups
     ```
   - Click **"Apply"**

## Step 5: Configure Token Configuration (Optional Claims)

Configure optional claims to include role information in tokens.

1. **Navigate to Token Configuration**
   - In the App Registration, go to **Token configuration** (left menu)

2. **Add Groups Claim** (if using groups)
   - Click **"Add groups claim"**
   - Select: **Security groups**
   - Under ID and Access tokens, check: **Group ID**
   - Click **"Add"**

3. **Add Optional Claims**
   - Click **"Add optional claim"**
   - Select token type: **ID**
   - Select claims:
     - `email`
     - `family_name`
     - `given_name`
   - Click **"Add"**

## Step 6: Assign Users to Application Roles

Now that roles are defined, assign users or groups to these roles.

1. **Navigate to Enterprise Applications**
   - Go to: **Azure Active Directory** → **Enterprise applications**
   - Search for: `argus-administration-portal-dev`
   - Click on the application

2. **Assign Users and Roles**
   - Click **"Users and groups"** (left menu)
   - Click **"Add user/group"**
   - Configure assignment:
     - **Users**: Click "None Selected" and select users who need access
     - **Select a role**: Click "None Selected" and choose **EDI Argus Administrator**
   - Click **"Assign"**

3. **Repeat for Additional Users**
   - Add all administrators who need portal access
   - Assign appropriate roles based on their responsibilities

## Step 7: Configure App Service Authentication

The App Service hosting the portal must be configured to use this App Registration.

1. **Navigate to App Service**
   - Go to: **Resource groups** → `rg-edi-dev-eastus2`
   - Find and click: `app-argus-portal-dev-eastus2`

2. **Configure Authentication**
   - In the App Service, go to **Authentication** (left menu under Settings)
   - Click **"Add identity provider"**

3. **Configure Identity Provider**
   - **Identity provider**: Select **Microsoft**
   - **App registration type**: Select **Pick an existing app registration in this directory**
   - **App registration**: Select `argus-administration-portal-dev`
   - **Supported account types**: **Current tenant - Single tenant**
   - **Issuer URL**: Leave as default (auto-populated)
   - **Client secret**: 
     - Option 1: Click **"Add a new secret"** (recommended for App Service)
     - Option 2: Create in App Registration first, then select here
   - **Client secret expiration**: Choose appropriate duration (12-24 months recommended)

4. **Configure Authentication Settings**
   - **Restrict access**: Select **Require authentication**
   - **Unauthenticated requests**: Select **HTTP 302 Found redirect: recommended for web sites**
   - **Token store**: **Enabled** (recommended)
   - **Advanced**:
     - **Allowed token audiences**: Leave empty (uses default)
     - **Allowed external redirect URLs**: Leave empty unless needed

5. **Click "Add"**

## Step 8: Provide Configuration to Development Team

After completing the setup, provide the following information to the development team:

**App Registration Details:**

```yaml
# Azure AD Configuration
AZURE_AD_TENANT_ID: 76888a14-162d-4764-8e6f-c5a34addbd87
AZURE_AD_CLIENT_ID: [Application (client) ID from Step 1]
AZURE_AD_INSTANCE: https://login.microsoftonline.com/

# Application Roles
ADMIN_ROLE: edi-argus-administrator
VIEWER_ROLE: edi-argus-viewer (if created)
OPERATOR_ROLE: edi-argus-operator (if created)

# Portal URL
PORTAL_URL_DEV: https://app-argus-portal-dev-eastus2.azurewebsites.net
```

**Delivery Method:**
- Share via secure channel (encrypted email, Teams, or documentation repository)
- Development team will add these to application configuration files

## Step 9: Testing and Validation

### Test User Authentication

1. **Access the Portal**
   - Navigate to: `https://app-argus-portal-dev-eastus2.azurewebsites.net`
   - You should be redirected to Microsoft login page

2. **Sign In**
   - Sign in with an Azure AD account that has been assigned a role
   - After authentication, you should be redirected back to the portal

3. **Verify Role Claims**
   - Development team can verify that role claims are present in the authentication token
   - Admin users should see all administrative features

### Test Unauthorized Access

1. **Test with Unassigned User**
   - Sign in with an Azure AD account that has NOT been assigned a role
   - User should authenticate but may see limited or no features (depending on app logic)

2. **Test Anonymous Access**
   - Open portal in incognito/private browser window
   - Should be redirected to login immediately

## Security Considerations

### Access Control
- **Principle of Least Privilege**: Only assign administrator roles to users who require full access
- **Regular Audits**: Review assigned users quarterly to remove access for users who no longer need it
- **Use Groups**: Consider creating Azure AD groups for role assignments instead of individual users

### Secret Management
- **Client Secret Rotation**: Set calendar reminders for secret expiration
- **Rotate Before Expiration**: Update secrets 30 days before expiration to avoid service disruption
- **Store Securely**: Never commit client secrets to source control

### Monitoring
- **Sign-in Logs**: Monitor Azure AD sign-in logs for unusual authentication patterns
- **App Registration Activity**: Review app registration changes in Azure AD audit logs
- **Failed Authentication**: Set up alerts for repeated failed sign-in attempts

### Token Configuration
- **Token Lifetime**: Use default token lifetimes unless specific requirements exist
- **Refresh Tokens**: Ensure refresh tokens are enabled for seamless user experience
- **Scope Minimization**: Only request the minimum required API permissions

## Troubleshooting

### Common Issues

**Issue: "AADSTS50011: The reply URL specified in the request does not match"**
- **Solution**: Verify the redirect URI in App Registration matches the exact URL in your application
- Check for trailing slashes, http vs https, and case sensitivity

**Issue: "User cannot access the application"**
- **Solution**: Verify the user has been assigned to the application with an appropriate role
- Go to Enterprise Applications → Users and groups to check assignments

**Issue: "Claims do not include roles"**
- **Solution**: Ensure admin consent was granted for API permissions
- Verify roles are properly defined in App Roles section
- Check that users are assigned to roles, not just the application

**Issue: "Authentication works but role-based features don't"**
- **Solution**: Development team needs to verify role claim name in application code matches the role `Value` field (e.g., `edi-argus-administrator`)

## Additional Configuration for TEST Environment

When deploying to TEST environment, repeat Steps 1-8 with the following changes:

**App Registration Name**: `argus-administration-portal-test`

**Redirect URIs**:
```
https://app-argus-portal-test-eastus2.azurewebsites.net/signin-oidc
https://localhost:5001/signin-oidc
```

**App Service**: `app-argus-portal-test-eastus2`

**Note**: Keep the same role definitions (values) across all environments for consistency.

## Maintenance and Lifecycle

### Regular Maintenance Tasks

- [ ] **Quarterly**: Review and audit user/group assignments
- [ ] **Before Secret Expiration**: Rotate client secrets (30 days notice)
- [ ] **Annually**: Review and update role definitions if needed
- [ ] **On User Offboarding**: Remove application access within 24 hours

### Secret Rotation Process

1. Create new client secret in App Registration
2. Update App Service authentication configuration with new secret
3. Test authentication still works
4. Delete old secret after confirming new secret works
5. Document rotation date

## Checklist for Operations Team

### Initial Setup
- [ ] Create App Registration `argus-administration-portal-dev`
- [ ] Configure authentication settings and redirect URIs
- [ ] Set up API permissions and grant admin consent
- [ ] Create application role: `edi-argus-administrator`
- [ ] Create optional application roles (viewer, operator) if needed
- [ ] Assign initial administrator users to roles
- [ ] Configure App Service authentication with App Registration
- [ ] Document and provide configuration details to development team:
  - [ ] AZURE_AD_TENANT_ID: `76888a14-162d-4764-8e6f-c5a34addbd87`
  - [ ] AZURE_AD_CLIENT_ID: [Generated during registration]
- [ ] Set calendar reminder for client secret expiration
- [ ] Test authentication with assigned user
- [ ] Test that unassigned user cannot access portal features

### Future Deployments
- [ ] Repeat setup for TEST environment when ready
- [ ] Create production App Registration when deploying to PROD subscription

## Reference Information

### Azure AD Application URLs
- **App Registrations**: https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps
- **Enterprise Applications**: https://portal.azure.com/#view/Microsoft_AAD_IAM/StartboardApplicationsMenuBlade/~/AppAppsPreview

### Useful Azure CLI Commands

**List App Registrations:**
```bash
az ad app list --display-name argus-administration-portal-dev
```

**Show App Roles:**
```bash
az ad app show --id <application-id> --query appRoles
```

**List Role Assignments:**
```bash
az ad app permission list --id <application-id>
```

### Related Documentation
- [Azure AD Authentication in App Service](https://learn.microsoft.com/en-us/azure/app-service/overview-authentication-authorization)
- [Configure App Roles](https://learn.microsoft.com/en-us/azure/active-directory/develop/howto-add-app-roles-in-azure-ad-apps)
- [Microsoft Identity Platform](https://learn.microsoft.com/en-us/azure/active-directory/develop/)

---

**Document Version**: 1.0  
**Last Updated**: January 8, 2025  
**Next Review**: Upon secret expiration or portal deployment to TEST/PROD
