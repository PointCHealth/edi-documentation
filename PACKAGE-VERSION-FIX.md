# Fixing the Current Package/Build Issue

**Date:** October 7, 2025  
**Issue:** edi-mappers function apps cannot build because they reference newer code that's not in the published NuGet packages.

---

## Problem Summary

The EligibilityMapper function references:
- `EDIContext` class from EDI.Logging.Models
- `AddEDITelemetry()` extension method from EDI.Logging.Extensions

These exist in the **current source code** but are not in the **published NuGet package version 1.0.0**.

The `EDI.Logging.csproj` file shows version **1.0.1**, but GitHub Packages only has version **1.0.0** published.

---

## Solution: Republish Packages

### Step 1: Verify Current State

```powershell
cd C:\repos\edi-platform\edi-platform-core

# Check current versions
.\scripts\Update-PackageVersions.ps1 -Action List
```

**Expected Output:**
```
Package                 Version  Path
-------                 -------  ----
EDI.Configuration       1.0.1    EDI.Configuration.csproj
EDI.Core               1.0.1    EDI.Core.csproj
EDI.Logging            1.0.1    EDI.Logging.csproj  ← Version is already bumped
EDI.Messaging          1.0.1    EDI.Messaging.csproj
EDI.Storage            1.0.1    EDI.Storage.csproj
EDI.X12                1.0.1    EDI.X12.csproj
```

---

### Step 2: Verify Package Configuration

```powershell
.\scripts\Update-PackageVersions.ps1 -Action Verify
```

This will check:
- All project files exist
- Version numbers are valid
- Required metadata is present
- No uncommitted changes

**Fix any issues** reported before proceeding.

---

### Step 3: Option A - Auto Publish (Recommended)

If versions are already bumped (like they are now), just push the changes:

```powershell
# Make sure all changes are committed
git add .
git commit -m "chore: Prepare for package republish with latest code"
git push origin main
```

GitHub Actions will automatically:
1. Detect the .csproj file push
2. Trigger `publish-nuget.yml` workflow
3. Build and publish all packages
4. Version 1.0.1 will be available in ~3-5 minutes

**Monitor progress:**
```powershell
# View workflow runs
gh run list --workflow=publish-nuget.yml

# Watch the current run
gh run watch
```

---

### Step 3: Option B - Manual Trigger

If you want to trigger publishing manually:

```powershell
.\scripts\Update-PackageVersions.ps1 -Action Publish
```

Or directly with GitHub CLI:
```powershell
gh workflow run publish-nuget.yml
```

---

### Step 4: Wait for Publishing to Complete

1. Go to: https://github.com/PointCHealth/edi-platform-core/actions
2. Watch the `Publish NuGet Packages` workflow
3. Wait for green checkmark (usually 3-5 minutes)

---

### Step 5: Verify Packages Published

```powershell
# Check GitHub Packages
gh api /orgs/PointCHealth/packages/nuget/EDI.Logging/versions | ConvertFrom-Json | Select-Object -First 3

# Or visit in browser
start https://github.com/orgs/PointCHealth/packages/nuget/package/EDI.Logging
```

You should see **version 1.0.1** listed.

---

### Step 6: Update Consumer Projects

Now that the packages are published, update the edi-mappers project:

```powershell
cd C:\repos\edi-platform\edi-mappers

# Clear NuGet cache (close VS Code first if it's open)
dotnet nuget locals all --clear

# Force restore to get new versions
dotnet restore --force-evaluate

# Build
dotnet build
```

**Expected:** Build should now succeed! ✅

---

## If Versions Need Bumping

If the versions were NOT already bumped to 1.0.1, use this command:

```powershell
cd C:\repos\edi-platform\edi-platform-core

# Bump all packages to 1.0.1 and commit
.\scripts\Update-PackageVersions.ps1 -Action Bump -VersionPart Patch -Commit
```

This will:
1. Increment patch version (1.0.0 → 1.0.1)
2. Update all .csproj files
3. Commit with message: `chore: Bump package versions - ...`
4. Push to remote
5. Trigger automatic publishing via GitHub Actions

---

## Alternative: Bump to 1.0.2

If you want to bump to the NEXT version (1.0.1 → 1.0.2):

```powershell
cd C:\repos\edi-platform\edi-platform-core

# Bump to 1.0.2
.\scripts\Update-PackageVersions.ps1 -Action Bump -VersionPart Patch -Commit
```

---

## Verifying the Fix

After packages are published and consumer is updated:

```powershell
# In edi-mappers
cd C:\repos\edi-platform\edi-mappers

# Check resolved versions
dotnet list package | Select-String "EDI\."

# Should show:
# EDI.Configuration  1.0.*  1.0.1
# EDI.Core          1.0.*  1.0.1
# EDI.Logging       1.0.*  1.0.1  ← Now resolved to 1.0.1
# EDI.Messaging     1.0.*  1.0.1
# EDI.Storage       1.0.*  1.0.1
# EDI.X12           1.0.*  1.0.1

# Build specific function
cd functions\EligibilityMapper.Function
dotnet build

# Should succeed with 0 errors! ✅
```

---

## Complete Workflow (All Commands)

Here's the complete sequence to fix the issue:

```powershell
# 1. Check current versions
cd C:\repos\edi-platform\edi-platform-core
.\scripts\Update-PackageVersions.ps1 -Action List

# 2. Verify configuration
.\scripts\Update-PackageVersions.ps1 -Action Verify

# 3. Push changes (if not already pushed)
git push origin main

# 4. Monitor publish
gh run watch

# 5. Update consumer after publish completes
cd C:\repos\edi-platform\edi-mappers
dotnet nuget locals all --clear
dotnet restore --force-evaluate
dotnet build

# 6. Verify success
dotnet list package | Select-String "EDI\."
```

---

## Timeline

| Step | Duration | Description |
|------|----------|-------------|
| 1. Push changes | 30 seconds | Git push to trigger workflow |
| 2. Workflow starts | 30 seconds | GitHub Actions picks up changes |
| 3. Build & publish | 3-4 minutes | Workflow builds and publishes packages |
| 4. Package available | Immediate | Packages visible in GitHub Packages |
| 5. Update consumers | 1 minute | Clear cache, restore, build |
| **Total** | **~5 minutes** | End-to-end fix |

---

## Prevention for Future

To prevent this issue in the future:

1. **Always bump versions when adding new public APIs:**
   ```powershell
   .\scripts\Update-PackageVersions.ps1 -Action Bump -VersionPart Minor -Commit
   ```

2. **Use the script before making changes in consumer projects:**
   ```powershell
   # Check what's published
   gh api /orgs/PointCHealth/packages/nuget/EDI.Logging/versions
   
   # Make sure local code matches published packages
   ```

3. **Set up Dependabot** (optional) to auto-update consumer projects when new versions are published.

---

## Quick Command Reference

```powershell
# Check versions
.\scripts\Update-PackageVersions.ps1 -Action List

# Verify everything is OK
.\scripts\Update-PackageVersions.ps1 -Action Verify

# Bump patch version (bug fixes)
.\scripts\Update-PackageVersions.ps1 -Action Bump -VersionPart Patch -Commit

# Bump minor version (new features)
.\scripts\Update-PackageVersions.ps1 -Action Bump -VersionPart Minor -Commit

# Trigger publish manually
.\scripts\Update-PackageVersions.ps1 -Action Publish

# View published packages
gh api /orgs/PointCHealth/packages?package_type=nuget

# Clear NuGet cache
dotnet nuget locals all --clear

# Force restore
dotnet restore --force-evaluate
```

---

**Status:** Ready to Execute  
**Estimated Time:** 5 minutes  
**Next Step:** Run Step 1 above
