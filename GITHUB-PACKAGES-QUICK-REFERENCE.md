# GitHub Packages Quick Reference

**Last Updated:** October 7, 2025

---

## For Publishers (edi-platform-core)

### Publishing Workflow Trigger

```bash
# Manual publish (workflow_dispatch)
gh workflow run publish-nuget.yml --repo PointCHealth/edi-platform-core

# Or push to main with shared library changes
git add shared/EDI.Configuration/
git commit -m "feat: Add HTTPS endpoint support"
git push origin main
# Automatically triggers publish-nuget.yml
```

### Update Package Version

```xml
<!-- shared/EDI.Configuration/EDI.Configuration.csproj -->
<PropertyGroup>
  <Version>1.1.0</Version>  <!-- Update this -->
</PropertyGroup>
```

### Tag Release

```bash
git tag -a EDI.Configuration-v1.1.0 -m "Add HTTPS endpoint support"
git push origin EDI.Configuration-v1.1.0
# Triggers release workflow
```

---

## For Consumers (All Other Repos)

### Initial Setup (One-Time)

#### 1. Create nuget.config

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="github" value="https://nuget.pkg.github.com/PointCHealth/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <github>
      <add key="Username" value="%GITHUB_ACTOR%" />
      <add key="ClearTextPassword" value="%GITHUB_TOKEN%" />
    </github>
  </packageSourceCredentials>
</configuration>
```

#### 2. Update .csproj

```xml
<!-- Before -->
<ProjectReference Include="..\edi-platform-core\shared\EDI.Configuration\EDI.Configuration.csproj" />

<!-- After -->
<PackageReference Include="EDI.Configuration" Version="1.0.*" />
```

#### 3. Update CI/CD Workflow

```yaml
# Add to permissions
permissions:
  packages: read

# Update restore step
- name: Restore
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: dotnet restore
```

### Local Development Setup

```powershell
# 1. Create Personal Access Token
# Go to: https://github.com/settings/tokens/new
# Scopes: read:packages
# Save token: ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 2. Set environment variable (PowerShell)
$env:GITHUB_TOKEN = "ghp_your_token_here"

# 3. Restore packages
dotnet restore
```

**Or add to PowerShell profile:**

```powershell
# Add to: $PROFILE
$env:GITHUB_TOKEN = 'ghp_your_token_here'
```

### Update Packages

```bash
# Update all packages
dotnet restore --force-evaluate

# Update specific package
dotnet add package EDI.Configuration --version 1.1.0

# Or let Dependabot do it automatically (see below)
```

---

## Dependabot Configuration

**File:** `.github/dependabot.yml`

```yaml
version: 2
updates:
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
    groups:
      edi-shared-libraries:
        patterns:
          - "EDI.*"
```

Dependabot will automatically:
1. Check for updates every Monday
2. Create PRs for outdated packages
3. Group all EDI.* packages into one PR
4. Run CI tests on PR
5. Assign team for review

---

## Version Strategy

### Semantic Versioning

| Change Type | Version Bump | Example | Consumer Action |
|-------------|-------------|---------|-----------------|
| Bug fix | PATCH (1.0.0 → 1.0.1) | Fix parser bug | Optional update |
| New feature | MINOR (1.0.0 → 1.1.0) | Add new method | Optional update |
| Breaking change | MAJOR (1.0.0 → 2.0.0) | Remove method | **Required update** |

### Version Wildcards

```xml
<!-- Exact version (locks to specific) -->
<PackageReference Include="EDI.Configuration" Version="1.0.0" />

<!-- Latest patch (recommended) -->
<PackageReference Include="EDI.Configuration" Version="1.0.*" />

<!-- Latest minor (risky) -->
<PackageReference Include="EDI.Configuration" Version="1.*" />
```

**Recommendation:** Use `1.0.*` for automatic patch updates.

---

## Troubleshooting

### Error: Unable to load the service index

**Solution:**

```powershell
# Check token is set
echo $env:GITHUB_TOKEN

# If empty, set it
$env:GITHUB_TOKEN = "ghp_your_token"
```

### Error: 401 Unauthorized

**Solution:** Token needs `read:packages` scope. Create new PAT.

### Error: Package not found

**Solution:** Check package exists:

```bash
# View published packages
gh api /orgs/PointCHealth/packages/nuget/EDI.Configuration

# Or visit: https://github.com/orgs/PointCHealth/packages
```

### CI build fails with package restore error

**Solution:** Add to workflow:

```yaml
permissions:
  packages: read

steps:
  - name: Restore
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    run: dotnet restore
```

---

## Useful Commands

```bash
# List all packages from GitHub feed
dotnet nuget list source

# Clear NuGet cache (if corrupted)
dotnet nuget locals all --clear

# Restore packages with diagnostics
dotnet restore --verbosity detailed

# List all package versions available
dotnet list package --outdated

# Force re-evaluation of package versions
dotnet restore --force-evaluate
```

---

## GitHub CLI Shortcuts

```bash
# Trigger publish workflow
gh workflow run publish-nuget.yml --repo PointCHealth/edi-platform-core

# View workflow runs
gh run list --workflow=publish-nuget.yml --repo PointCHealth/edi-platform-core

# View packages in organization
gh api /orgs/PointCHealth/packages?package_type=nuget

# Create release (triggers publish)
gh release create EDI.Configuration-v1.1.0 --repo PointCHealth/edi-platform-core --title "EDI.Configuration v1.1.0" --notes "Add HTTPS support"
```

---

## Package URLs

- **Feed URL:** `https://nuget.pkg.github.com/PointCHealth/index.json`
- **Package Browser:** `https://github.com/orgs/PointCHealth/packages`
- **Package Details:** `https://github.com/orgs/PointCHealth/packages/nuget/EDI.Configuration`

---

## Support

- **Documentation:** See `MULTI-REPO-STRATEGY.md` for detailed guide
- **Issues:** Create issue in `edi-platform` repository
- **Questions:** Contact EDI Platform Team

---

**Quick Start Checklist:**

Consumer Repository Setup:
- [ ] Add `nuget.config`
- [ ] Replace `ProjectReference` with `PackageReference`
- [ ] Update workflow permissions
- [ ] Add `GITHUB_TOKEN` to restore step
- [ ] Test local build with PAT
- [ ] Test CI build

Publisher Setup:
- [ ] Add package metadata to `.csproj`
- [ ] Create `publish-nuget.yml` workflow
- [ ] Test manual publish
- [ ] Verify packages appear in GitHub

---

**Status:** ✅ Ready for Use  
**Last Updated:** October 7, 2025
