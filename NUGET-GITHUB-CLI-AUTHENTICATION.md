# NuGet Authentication with GitHub CLI

**Last Updated:** October 9, 2025  
**Audience:** EDI Platform Developers

---

## Overview

NuGet **can use GitHub CLI managed credentials** through **Git Credential Manager (GCM)**. The existing `nuget.config` files support both authentication methods automatically.

---

## How It Works

### Authentication Flow

```
┌─────────────────────────────────────────────────────────┐
│  NuGet Restore (dotnet restore)                        │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│  Read nuget.config                                      │
│  ClearTextPassword = "%GITHUB_TOKEN%"                   │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
         Is GITHUB_TOKEN set?
                  │
        ┌─────────┴─────────┐
        │                   │
       YES                 NO
        │                   │
        ▼                   ▼
  ┌──────────┐      ┌──────────────────┐
  │ Use Env  │      │ Git Credential   │
  │ Variable │      │ Manager (GCM)    │
  └──────────┘      └─────────┬────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ Query GitHub CLI │
                    │ Keyring          │
                    └─────────┬────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ Use Credentials  │
                    └──────────────────┘
```

### Key Components

1. **nuget.config** - Specifies GitHub Packages as source
2. **%GITHUB_TOKEN%** - Environment variable placeholder
3. **Git Credential Manager (GCM)** - Credential helper (installed with Git for Windows)
4. **GitHub CLI** - Stores credentials securely in system keyring

---

## Current Configuration

### nuget.config Structure

All consumer repositories have this configuration:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="github" value="https://nuget.pkg.github.com/PointCHealth/index.json" />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <github>
      <add key="Username" value="%GITHUB_ACTOR%" />
      <!-- 
        Supports both authentication methods:
        1. GitHub CLI managed credentials (via GCM)
        2. Environment variable fallback
      -->
      <add key="ClearTextPassword" value="%GITHUB_TOKEN%" />
    </github>
  </packageSourceCredentials>
</configuration>
```

### Why This Works for Both Methods

**Environment Variable Expansion:**
- If `GITHUB_TOKEN` env var exists → NuGet expands `%GITHUB_TOKEN%` to actual token value
- If `GITHUB_TOKEN` env var does NOT exist → NuGet leaves it as literal `%GITHUB_TOKEN%`

**Git Credential Manager Fallback:**
- When NuGet fails to authenticate with literal `%GITHUB_TOKEN%`
- GCM intercepts the authentication request
- GCM queries GitHub CLI: `gh auth token`
- GCM provides the token from keyring to NuGet
- NuGet successfully authenticates

**Result:** Same config works for both methods! 🎉

---

## Setup Options

### Option A: GitHub CLI Managed Credentials (⭐ RECOMMENDED)

**Prerequisites:**
- Git for Windows 2.30+ (includes Git Credential Manager)
- GitHub CLI installed

**Setup:**

```powershell
# 1. Install GitHub CLI (if needed)
winget install GitHub.cli

# 2. Authenticate
gh auth login
# Select: GitHub.com → HTTPS → Authenticate via browser

# 3. Verify authentication includes read:packages scope
gh auth status
# Should show: 'gist', 'read:org', 'repo', 'workflow'

# 4. Add read:packages scope if missing
gh auth refresh -h github.com -s read:packages

# 5. IMPORTANT: Clear any GITHUB_TOKEN environment variable
Remove-Item Env:\GITHUB_TOKEN -ErrorAction SilentlyContinue

# 6. If you have GITHUB_TOKEN in your PowerShell profile, comment it out:
notepad $PROFILE
# Comment or delete: $env:GITHUB_TOKEN = 'ghp_...'

# 7. Restart PowerShell or reload profile
. $PROFILE

# 8. Test NuGet restore
cd c:\repos\edi-platform\edi-sftp-connector
dotnet restore
# Should work without prompting!
```

**Benefits:**
- ✅ More secure (credentials encrypted in keyring)
- ✅ No manual token management
- ✅ Automatic credential rotation via `gh auth refresh`
- ✅ Works across all tools (git, gh, NuGet, npm)
- ✅ No sensitive data in PowerShell profile

---

### Option B: Environment Variable (Legacy)

**Use this if:**
- You prefer explicit control over tokens
- You're using automation scripts that set GITHUB_TOKEN
- GitHub CLI is not available in your environment

**Setup:**

```powershell
# 1. Create Personal Access Token
# Go to: https://github.com/settings/tokens/new
# Scopes: read:packages
# Copy token: ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 2. Add to PowerShell profile
notepad $PROFILE

# Add this line:
$env:GITHUB_TOKEN = 'ghp_your_token_here'

# 3. Reload profile
. $PROFILE

# 4. Test
cd c:\repos\edi-platform\edi-sftp-connector
dotnet restore
```

**⚠️ Important:** If using this method, do NOT use GitHub CLI authentication, as env var takes precedence.

---

## Verifying Your Setup

### Test Which Method Is Active

```powershell
# Check if GITHUB_TOKEN env var is set
if ($env:GITHUB_TOKEN) {
    Write-Host "Using ENVIRONMENT VARIABLE authentication" -ForegroundColor Yellow
    Write-Host "Token: $($env:GITHUB_TOKEN.Substring(0,7))..." -ForegroundColor Gray
} else {
    Write-Host "Using GITHUB CLI MANAGED CREDENTIALS (recommended)" -ForegroundColor Green
    
    # Verify GitHub CLI is authenticated
    gh auth status
}
```

### Test NuGet Restore

```powershell
# Navigate to any consumer repository
cd c:\repos\edi-platform\edi-sftp-connector

# Clear NuGet cache to force fresh authentication
dotnet nuget locals all --clear

# Restore packages (watch for authentication errors)
dotnet restore --verbosity detailed
```

**Expected output:**
```
Restoring packages for c:\repos\edi-platform\edi-sftp-connector\edi-sftp-connector.csproj...
  GET https://nuget.pkg.github.com/PointCHealth/download/...
  OK https://nuget.pkg.github.com/PointCHealth/download/... 123ms
```

**If you see 401 Unauthorized:**
- GitHub CLI not authenticated OR
- Missing `read:packages` scope OR
- GITHUB_TOKEN env var set to invalid/expired token

---

## Troubleshooting

### Error: 401 Unauthorized from GitHub Packages

**Diagnosis:**
```powershell
# Check environment variable
echo $env:GITHUB_TOKEN

# Check GitHub CLI status
gh auth status
```

**Solutions:**

#### If Using GitHub CLI (Recommended)

```powershell
# 1. Clear environment variable
Remove-Item Env:\GITHUB_TOKEN -ErrorAction SilentlyContinue

# 2. Check authentication
gh auth status
# Should show: "Logged in to github.com account <username> (keyring)"

# 3. Verify read:packages scope
# Look for: Token scopes: 'read:packages' (or 'repo' which includes read:packages)

# 4. Add scope if missing
gh auth refresh -h github.com -s read:packages

# 5. Test credential retrieval
gh auth token
# Should output a token starting with gho_...

# 6. Test NuGet restore
cd c:\repos\edi-platform\edi-sftp-connector
dotnet nuget locals all --clear
dotnet restore
```

#### If Using Environment Variable

```powershell
# 1. Verify token is set
echo $env:GITHUB_TOKEN
# Should show: ghp_...

# 2. Test token validity
gh auth status
# Or manually test:
curl -H "Authorization: token $env:GITHUB_TOKEN" https://api.github.com/user

# 3. If expired, create new token
# Go to: https://github.com/settings/tokens/new
# Scopes: read:packages
# Copy new token

# 4. Update PowerShell profile
notepad $PROFILE
# Update: $env:GITHUB_TOKEN = 'ghp_new_token'

# 5. Reload profile
. $PROFILE

# 6. Test
dotnet restore
```

---

### Error: "Unable to load the service index"

**Cause:** Network connectivity or GitHub Packages service issue

**Solution:**
```powershell
# 1. Check GitHub status
curl https://www.githubstatus.com/api/v2/status.json

# 2. Test GitHub Packages endpoint
curl https://nuget.pkg.github.com/PointCHealth/index.json

# 3. Check proxy/VPN settings
# May need to configure NuGet proxy if behind corporate firewall
```

---

### Error: Package not found (but exists)

**Cause:** Package published but not yet indexed OR insufficient permissions

**Solution:**
```powershell
# 1. Verify package exists
gh api /orgs/PointCHealth/packages/nuget/EDI.Configuration

# 2. Check your GitHub account has access to PointCHealth organization
gh api /user/orgs
# Should include PointCHealth

# 3. Wait 5 minutes for indexing (if just published)

# 4. Try again
dotnet nuget locals all --clear
dotnet restore
```

---

### Git Credential Manager Not Working

**Symptoms:**
- Prompted for password even with GitHub CLI authenticated
- "Authentication failed" errors

**Diagnosis:**
```powershell
# Check if GCM is configured
git config --global credential.helper
# Should show: manager or manager-core

# Check GCM version
git credential-manager --version
# Should be 2.0+
```

**Solution:**
```powershell
# Reconfigure Git Credential Manager
git credential-manager configure

# Verify it's set as helper
git config --global credential.helper
# Should show: manager

# Test credential retrieval
echo "protocol=https
host=github.com
" | git credential-manager get

# Should return credentials without prompting
```

---

## Migration Guide: Environment Variable → GitHub CLI

If you currently use `$env:GITHUB_TOKEN` in your PowerShell profile:

### Step 1: Set Up GitHub CLI

```powershell
# Install GitHub CLI
winget install GitHub.cli

# Authenticate via browser
gh auth login

# Add read:packages scope
gh auth refresh -h github.com -s read:packages

# Verify
gh auth status
# Should show keyring authentication with read:packages scope
```

### Step 2: Remove Environment Variable

```powershell
# Open profile
notepad $PROFILE

# Comment out or delete this line:
# $env:GITHUB_TOKEN = 'ghp_...'

# Save and close

# Reload profile
. $PROFILE

# Verify it's gone
echo $env:GITHUB_TOKEN
# Should show nothing
```

### Step 3: Test

```powershell
# Clear NuGet cache
dotnet nuget locals all --clear

# Test restore
cd c:\repos\edi-platform\edi-sftp-connector
dotnet restore --verbosity detailed

# Should succeed without prompting!
```

### Step 4: Cleanup

```powershell
# Optionally revoke old PAT token
# Go to: https://github.com/settings/tokens
# Find your old token
# Click "Revoke"
```

---

## CI/CD Pipelines

**Important:** GitHub Actions workflows should continue using `GITHUB_TOKEN` secret (auto-provided).

**Do NOT change workflow files:**

```yaml
# ✅ CORRECT - Keep this in workflows
- name: Restore
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: dotnet restore
```

GitHub CLI managed credentials are for **local development only**.

---

## Security Comparison

| Aspect | Environment Variable | GitHub CLI Managed |
|--------|---------------------|-------------------|
| **Storage** | Plain text in `$PROFILE` | Encrypted in Windows Credential Manager |
| **Exposure Risk** | Visible in process list, terminal history | Never exposed |
| **Scope Management** | Manual token recreation | `gh auth refresh -s <scope>` |
| **Rotation** | Update in profile, reload | Automatic via GitHub CLI |
| **Audit Trail** | None | GitHub logs CLI usage |
| **Multi-Tool Support** | Only tools that read GITHUB_TOKEN | All Git-integrated tools |
| **Accidental Commit** | Risk of committing token | Impossible (not in files) |

---

## Best Practices

### ✅ DO

- Use GitHub CLI managed credentials for local development
- Keep `nuget.config` in repository (no secrets in it)
- Use `gh auth refresh` to add/remove scopes as needed
- Revoke old PAT tokens after migrating to GitHub CLI
- Document which method your team uses

### ❌ DON'T

- Mix both methods (pick one per machine)
- Commit `GITHUB_TOKEN` to Git
- Share PAT tokens between team members
- Use admin-scoped tokens for NuGet restore (read:packages is sufficient)
- Set `GITHUB_TOKEN` env var if using GitHub CLI (causes conflicts)

---

## FAQ

### Q: Do I need to change nuget.config files?

**A:** No! The existing `%GITHUB_TOKEN%` placeholder works with both methods automatically.

---

### Q: Which method should I use?

**A:** GitHub CLI managed credentials (Option A) for better security and usability. Use environment variable (Option B) only if you have specific requirements.

---

### Q: Can I use different methods on different machines?

**A:** Yes! Each developer can choose their preferred method. The `nuget.config` supports both.

---

### Q: Will this break CI/CD pipelines?

**A:** No! GitHub Actions workflows use `secrets.GITHUB_TOKEN` which is unaffected. This change only applies to local development.

---

### Q: How often do I need to re-authenticate?

- **GitHub CLI:** Tokens don't expire (until you revoke them)
- **Environment Variable PAT:** 90 days (or your chosen expiration)

---

### Q: What if Git Credential Manager isn't installed?

**A:** It's included with Git for Windows 2.30+. If missing:

```powershell
# Standalone installer
winget install GitCredentialManager
```

---

### Q: Does this work on macOS/Linux?

**A:** Yes! GitHub CLI and Git Credential Manager support all platforms. Replace PowerShell commands with bash equivalents.

---

## Related Documentation

- **Quick Start:** `edi-documentation/GITHUB-PACKAGES-DEVELOPER-QUICKSTART.md`
- **Configuration Guide:** `scripts/README-GITHUB-CONFIGURATION.md`
- **Credentials Update Summary:** `scripts/GITHUB-CREDENTIALS-UPDATE-SUMMARY.md`
- **Official GCM Docs:** https://github.com/GitCredentialManager/git-credential-manager
- **Official GitHub CLI Docs:** https://cli.github.com/manual/gh_auth

---

**Last Updated:** October 9, 2025  
**Maintained By:** EDI Platform Team  
**Questions?** See troubleshooting section or ask in team chat
