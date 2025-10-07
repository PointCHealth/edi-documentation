# GitHub Packages - Developer Quick Start

**Date:** October 7, 2025  
**For:** EDI Platform Developers  
**Time Required:** 5 minutes

---

## ⚠️ Important: First-Time Setup Required

Before you can build any consumer repositories (edi-sftp-connector, edi-mappers, etc.), you need to configure GitHub Packages authentication **once**.

---

## Step 1: Create Personal Access Token (2 minutes)

1. **Go to:** https://github.com/settings/tokens/new

2. **Configure token:**
   - Note: `EDI Platform NuGet Packages`
   - Expiration: `90 days` (or your preference)
   - Select scope: ✅ `read:packages`

3. **Generate token** and copy it (starts with `ghp_`)

---

## Step 2: Configure PowerShell (2 minutes)

### Option A: Add to Profile (Recommended - Persists across sessions)

```powershell
# Open PowerShell profile
notepad $PROFILE

# If profile doesn't exist, create it
if (!(Test-Path $PROFILE)) {
    New-Item -Path $PROFILE -ItemType File -Force
}

# Add this line to your profile:
$env:GITHUB_TOKEN = 'ghp_your_token_here'

# Save and close notepad

# Reload profile
. $PROFILE

# Verify it's set
echo $env:GITHUB_TOKEN
```

### Option B: Set for Current Session Only

```powershell
# Set environment variable (only lasts until you close PowerShell)
$env:GITHUB_TOKEN = 'ghp_your_token_here'

# Verify it's set
echo $env:GITHUB_TOKEN
```

---

## Step 3: Test Authentication (1 minute)

```powershell
# Try restoring edi-sftp-connector
cd c:\repos\edi-platform\edi-sftp-connector
dotnet restore

# You should see:
# ✅ Restoring packages from GitHub Packages
# ✅ No authentication errors
```

---

## Troubleshooting

### ❌ Error: "Unable to load the service index"

**Problem:** Token not set

**Fix:**
```powershell
# Check if token is empty
echo $env:GITHUB_TOKEN

# If empty, set it again
$env:GITHUB_TOKEN = 'ghp_your_token_here'
```

---

### ❌ Error: "401 Unauthorized"

**Problem:** Token doesn't have correct scope

**Fix:**
1. Go back to https://github.com/settings/tokens
2. Find your token
3. Edit it
4. Make sure `read:packages` is checked
5. Regenerate token
6. Update your PowerShell profile with new token

---

### ❌ Error: "Package 'EDI.Configuration' not found"

**Problem:** Packages not yet published from edi-platform-core

**Fix:**
```powershell
# Check if packages are published
gh api /orgs/PointCHealth/packages/nuget/EDI.Configuration

# If not found, trigger publishing workflow
gh workflow run publish-nuget.yml --repo PointCHealth/edi-platform-core

# Wait 2-3 minutes for workflow to complete
# Then try dotnet restore again
```

---

## Daily Workflow

After initial setup, your normal workflow is unchanged:

```powershell
# Just use normal dotnet commands
cd c:\repos\edi-platform\edi-sftp-connector

dotnet restore  # Gets packages from GitHub automatically
dotnet build    # Builds with packages
dotnet run      # Runs your function
```

No extra steps needed! 🎉

---

## Which Repositories Need This?

**Need GitHub Token:**
- ✅ `edi-sftp-connector` - Uses EDI.Configuration, EDI.Storage
- ✅ `edi-mappers` - Uses all 6 shared libraries
- ✅ Any future repos that use shared libraries

**Don't Need GitHub Token:**
- ❌ `edi-platform-core` - Publisher, not consumer
- ❌ `edi-database-*` - Don't use shared libraries
- ❌ `edi-azure-infrastructure` - Infrastructure only
- ❌ `edi-documentation` - Documentation only

---

## FAQ

**Q: Do I need to do this on every machine?**  
A: Yes, each development machine needs its own GITHUB_TOKEN set.

**Q: Does my token expire?**  
A: Yes, based on the expiration you set. GitHub will email you before it expires.

**Q: What if I work on multiple GitHub organizations?**  
A: The token works for all orgs you have access to. One token is enough.

**Q: Can I use the same token for multiple repos?**  
A: Yes! One token works for all repos that need GitHub Packages.

**Q: What if my token gets compromised?**  
A: Revoke it immediately at https://github.com/settings/tokens and create a new one.

**Q: Do CI/CD pipelines need this?**  
A: No! GitHub Actions automatically provides `GITHUB_TOKEN` in workflows.

---

## Security Best Practices

✅ **DO:**
- Store token in PowerShell profile (not in code)
- Use minimal scopes (only `read:packages`)
- Set reasonable expiration (90 days recommended)
- Regenerate if compromised

❌ **DON'T:**
- Commit token to Git
- Share token with others
- Use personal token in CI/CD (use automatic `GITHUB_TOKEN`)
- Store in plain text files

---

## Getting Help

**If you're still having issues:**

1. Check your token at: https://github.com/settings/tokens
2. Verify it has `read:packages` scope
3. Make sure `$env:GITHUB_TOKEN` is set in PowerShell
4. Try running `dotnet nuget locals all --clear` to clear cache
5. Contact EDI Platform Team if still stuck

---

**Related Documentation:**
- Full Setup Guide: `edi-documentation/GITHUB-PACKAGES-IMPLEMENTATION-COMPLETE.md`
- Quick Reference: `edi-documentation/GITHUB-PACKAGES-QUICK-REFERENCE.md`
- Official Docs: https://docs.github.com/en/packages

---

**Status:** ✅ Ready to Use  
**Last Updated:** October 7, 2025
