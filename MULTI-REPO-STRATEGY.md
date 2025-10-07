# Multi-Repository Strategy for EDI Platform

**Document Version:** 1.0  
**Last Updated:** October 7, 2025  
**Status:** ✅ Recommended Approach  
**Owner:** EDI Platform Team

---

## Executive Summary

The EDI Platform uses a **multi-repository strategy** as the recommended approach for building and deploying the solution in GitHub. This decision is documented in **[ADR-012: Multi-Repository Strategy](./17-architecture-decisions.md#14-adr-012-multi-repository-strategy)** and is designed to optimize for:

- ✅ **Independent deployment lifecycles** (deploy configs 10x/day, functions 1x/week)
- ✅ **CI/CD efficiency** (build only what changed - 60% faster builds)
- ✅ **Clear team ownership** (each team owns their repositories)
- ✅ **Security & access control** (granular permissions per component)
- ✅ **Developer experience** (unified workspace despite multiple repos)

---

## The Answer: Multi-Repository (Not Monorepo)

### Why Multi-Repository is Best

**9 Separate Repositories + 1 Workspace Orchestrator**

```text
edi-platform (Workspace Orchestrator)
├── edi-platform-core (Shared libraries + core functions)
├── edi-sftp-connector (SFTP integration)
├── edi-mappers (Transaction mappers)
├── edi-connectors (Partner connectors)
├── edi-database-controlnumbers (Control number DB)
├── edi-database-eventstore (Event store DB)
├── edi-database-sftptracking (SFTP tracking DB)
├── edi-partner-configs (Partner configurations)
└── edi-documentation (Platform documentation)
```

### Key Benefits

| Benefit | Example | Impact |
|---------|---------|--------|
| **Independent Deployments** | Update partner config without deploying functions | 10x daily config deploys vs. 1x weekly function deploys |
| **Fast CI/CD** | Only build changed components | 5 minutes vs. 30 minutes for full build |
| **Clear Ownership** | Integration team owns `edi-mappers` | No cross-team conflicts or coordination overhead |
| **Granular Security** | Operations team can update configs only | Principle of least privilege |
| **Parallel Work** | Teams work independently | No merge conflicts across components |

---

## Comparison: Multi-Repo vs. Monorepo

### Deployment Lifecycle Example

**Monorepo Problem:**

```text
Changed files: edi-partner-configs/partners/AETNA-001.json
Triggered builds:
  ❌ edi-platform-core (unnecessary)
  ❌ edi-sftp-connector (unnecessary)
  ❌ edi-mappers (unnecessary)
  ❌ edi-connectors (unnecessary)
  ❌ All databases (unnecessary)
Total build time: 30 minutes
Deploy: All or nothing (high risk)
```

**Multi-Repo Solution:**

```text
Changed files: edi-partner-configs/partners/AETNA-001.json
Triggered builds:
  ✅ edi-partner-configs only
Total build time: 2 minutes
Deploy: Config blob upload only (low risk)
```

### Build Efficiency Comparison

| Scenario | Monorepo | Multi-Repo | Savings |
|----------|----------|------------|---------|
| Update partner config | 30 min (builds everything) | 2 min (config only) | **28 min (93%)** |
| Fix bug in one mapper | 30 min (builds all mappers) | 5 min (one mapper) | **25 min (83%)** |
| Update documentation | 30 min (CI runs) | 0 min (no builds) | **30 min (100%)** |
| Update shared library | 30 min (monolith) | 10 min (parallel) | **20 min (67%)** |

**Weekly Savings:** ~300 minutes/week = **5 hours/week** = **260 hours/year**

---

## Repository Structure

### 1. Core Platform Repositories

#### edi-platform-core
- **Purpose:** Shared .NET libraries and core Azure Functions
- **Technology:** C# .NET 9, NuGet packages
- **Deployment:** NuGet packages + Function Apps
- **Cadence:** Weekly patches
- **Team:** Platform Team

#### edi-sftp-connector
- **Purpose:** SFTP file ingestion connector
- **Technology:** C# .NET 9 Azure Function
- **Deployment:** Function App to dev/test/prod
- **Cadence:** Weekly
- **Team:** Platform Team

#### edi-mappers
- **Purpose:** Transaction transformation functions (834, 837, 270/271, 835)
- **Technology:** C# .NET 9 Azure Functions
- **Deployment:** Function Apps per transaction type
- **Cadence:** Bi-weekly
- **Team:** Integration Team

#### edi-connectors
- **Purpose:** Partner-specific integration connectors
- **Technology:** C# .NET 9 Azure Functions
- **Deployment:** Function Apps per partner
- **Cadence:** As needed
- **Team:** Integration Team

### 2. Database Repositories

#### edi-database-controlnumbers
- **Purpose:** X12 control number management
- **Technology:** SQL DACPAC project
- **Deployment:** Azure SQL Database
- **Cadence:** Monthly migrations
- **Team:** Database Team

#### edi-database-eventstore
- **Purpose:** Event sourcing for enrollment (834)
- **Technology:** SQL DACPAC project
- **Deployment:** Azure SQL Database
- **Cadence:** Monthly migrations
- **Team:** Database Team

#### edi-database-sftptracking
- **Purpose:** SFTP file tracking and metadata
- **Technology:** EF Core migrations
- **Deployment:** Azure SQL Database
- **Cadence:** Monthly migrations
- **Team:** Database Team

### 3. Configuration & Documentation

#### edi-partner-configs
- **Purpose:** Trading partner configurations (JSON)
- **Technology:** JSON schema validation
- **Deployment:** Azure Blob Storage
- **Cadence:** Multiple times daily
- **Team:** Operations Team

#### edi-documentation
- **Purpose:** Platform documentation (this document)
- **Technology:** Markdown
- **Deployment:** GitHub Pages (optional)
- **Cadence:** As needed
- **Team:** All teams contribute

---

## Workspace Orchestration

### The `edi-platform` Repository

The `edi-platform` repository serves as the **workspace orchestrator** that provides:

1. **Setup Scripts:** One-command clone and setup
2. **VS Code Workspace:** Unified development environment
3. **Shared Tooling:** Common scripts and utilities
4. **Documentation Links:** Connects all repositories

### Developer Experience

Despite having 9 repositories, developers get a **unified experience**:

```powershell
# One-time setup (5 minutes)
git clone https://github.com/PointCHealth/edi-platform.git
cd edi-platform
.\scripts\setup-core.ps1

# Opens VS Code with ALL 9 repos loaded as workspace folders
code edi-platform.code-workspace
```

**VS Code Workspace Features:**

- 🔍 **Unified search** across all 9 repositories
- 🐛 **Integrated debugging** with breakpoints across repos
- ⚙️ **Shared settings** (formatting, linting, extensions)
- 📂 **Folder organization** with logical grouping
- 🔗 **Cross-repo navigation** via "Go to Definition"

### Multi-Repo Operations

PowerShell scripts in `edi-platform/scripts/` automate common tasks:

```powershell
# Check status of all repos
.\scripts\check-status.ps1

# Create feature branch across multiple repos
.\scripts\create-branch.ps1 -BranchName "feature/add-partner-BCBS"

# Pull latest changes from all repos
.\scripts\pull-all.ps1

# Switch all repos to a specific branch
.\scripts\switch-all-branches.ps1 -BranchName "main"
```

---

## GitHub Actions Strategy

### Per-Repository CI/CD

Each repository has its own **tailored GitHub Actions workflows**:

```yaml
edi-platform-core/.github/workflows/
├── function-ci.yml    # Build, test, NuGet pack
└── function-cd.yml    # Deploy to dev/test/prod

edi-mappers/.github/workflows/
├── function-ci.yml    # Build, test mappers
└── function-cd.yml    # Deploy mapper functions

edi-partner-configs/.github/workflows/
├── config-ci.yml      # JSON schema validation
└── config-cd.yml      # Upload to blob storage

edi-database-controlnumbers/.github/workflows/
├── database-ci.yml    # Build DACPAC
└── database-cd.yml    # Deploy with migrations
```

### Orchestrated Deployments

For **coordinated releases**, use workflow dispatches:

```yaml
# edi-platform/.github/workflows/orchestrated-deploy.yml
name: Orchestrated Platform Deploy

on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [dev, test, prod]

jobs:
  deploy-core:
    uses: PointCHealth/edi-platform-core/.github/workflows/deploy.yml@main
    with:
      environment: ${{ inputs.environment }}
  
  deploy-mappers:
    needs: deploy-core
    uses: PointCHealth/edi-mappers/.github/workflows/deploy.yml@main
    with:
      environment: ${{ inputs.environment }}
  
  deploy-connectors:
    needs: deploy-core
    uses: PointCHealth/edi-sftp-connector/.github/workflows/deploy.yml@main
    with:
      environment: ${{ inputs.environment }}
```

**Benefits:**

- ✅ **Independent pipelines** run only when needed
- ✅ **Parallel builds** for independent components
- ✅ **Isolated failures** don't block other repos
- ✅ **Flexible sequencing** with `needs:` dependencies

---

## Why NOT Monorepo?

### Monorepo Drawbacks

| Problem | Impact | Multi-Repo Solution |
|---------|--------|---------------------|
| **Large clone size** | 500MB+ repo, slow `git clone` | Each repo <50MB, fast clones |
| **All-or-nothing deploys** | High risk, rollback everything | Deploy only changed components |
| **Complex CI/CD** | Conditional build matrix logic | Simple per-repo workflows |
| **Slow builds** | Build all 7 function apps always | Build only changed components |
| **Merge conflicts** | Teams conflict on shared files | Isolated changes per team |
| **Access control** | All-or-nothing permissions | Granular per-repo permissions |
| **Dependency conflicts** | Version lock-in across all | Independent versioning |

### Real-World Example: Conflicting Dependencies

**Monorepo Problem:**

```text
Scenario: X12 library needs to be upgraded
- Mappers team wants v2.0 (new features for 835 processing)
- Connectors team wants v1.5 (proven stability)
- Result: Blocked! Must coordinate upgrade or force one team to wait.
```

**Multi-Repo Solution:**

```text
Scenario: X12 library needs to be upgraded
- edi-mappers upgrades to v2.0 independently
- edi-connectors stays on v1.5 until ready
- Result: Both teams move at their own pace!
```

---

## Team Ownership Model

### Repository Ownership

| Repository | Primary Owner | Contributors | Access Level |
|------------|---------------|--------------|--------------|
| **edi-platform-core** | Platform Team | All teams | Write (Platform), Read (Others) |
| **edi-sftp-connector** | Platform Team | Integration Team | Write |
| **edi-mappers** | Integration Team | Platform Team | Write |
| **edi-connectors** | Integration Team | Partner Teams | Write |
| **edi-database-*** | Database Team | Platform Team | Write (DB), Read (Others) |
| **edi-partner-configs** | Operations Team | All teams | Write (Ops), Read (All) |
| **edi-documentation** | Architecture Team | All teams | Write (All) |

### CODEOWNERS Files

Each repository has a `CODEOWNERS` file for automatic PR assignments:

```text
# edi-mappers/.github/CODEOWNERS
* @PointCHealth/integration-team

/Functions/Eligibility/ @PointCHealth/eligibility-team
/Functions/Claims/ @PointCHealth/claims-team
/Functions/Enrollment/ @PointCHealth/enrollment-team
```

---

## Dependency Management

### Shared Library Publishing Strategy: GitHub Packages

The EDI Platform uses **GitHub Packages** as the NuGet feed for shared libraries. This provides:

- ✅ **Automatic Authentication**: Uses `GITHUB_TOKEN` in workflows (no manual setup)
- ✅ **Organization-Scoped**: All repos in PointCHealth org can access packages
- ✅ **Version Control**: Semantic versioning with Git integration
- ✅ **Free for Private Repos**: No additional cost within organization
- ✅ **Integrated with CI/CD**: Seamless GitHub Actions integration

### Publishing Packages from edi-platform-core

#### Package Metadata in Projects

**edi-platform-core** shared libraries include NuGet metadata:

```xml
<!-- shared/EDI.Configuration/EDI.Configuration.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    
    <!-- NuGet Package Metadata -->
    <PackageId>EDI.Configuration</PackageId>
    <Version>1.0.0</Version>
    <Authors>PointCHealth Platform Team</Authors>
    <Company>PointCHealth</Company>
    <Description>Partner configuration management for EDI Platform</Description>
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <RepositoryUrl>https://github.com/PointCHealth/edi-platform-core</RepositoryUrl>
    <PackageTags>edi;healthcare;configuration</PackageTags>
  </PropertyGroup>
</Project>
```

**All shared libraries** in `edi-platform-core/shared/` directory:
- `EDI.Configuration` - Partner config management
- `EDI.Core` - Common utilities (ProcessingResult, IMessagePublisher, IStorageService)
- `EDI.Logging` - Structured logging abstractions
- `EDI.Messaging` - Service Bus messaging wrappers
- `EDI.Storage` - Blob storage abstractions
- `EDI.X12` - X12 parsing and generation

#### GitHub Actions Workflow for Publishing

**File**: `edi-platform-core/.github/workflows/publish-nuget.yml`

```yaml
name: Publish NuGet Packages

on:
  push:
    branches: [main]
    paths:
      - 'shared/**'
      - '**/*.csproj'
  release:
    types: [published]
  workflow_dispatch:
    inputs:
      version:
        description: 'Package version override (optional)'
        required: false

permissions:
  contents: read
  packages: write

jobs:
  publish:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      
      - name: Restore Dependencies
        run: dotnet restore
      
      - name: Build Shared Libraries
        run: dotnet build shared --configuration Release --no-restore
      
      - name: Run Tests
        run: dotnet test shared --configuration Release --no-build
      
      - name: Pack NuGet Packages
        run: |
          for project in shared/*/; do
            echo "Packing $project"
            dotnet pack "$project" \
              --configuration Release \
              --no-build \
              --output ./packages \
              --include-symbols \
              --include-source
          done
      
      - name: Publish to GitHub Packages
        run: |
          dotnet nuget push ./packages/*.nupkg \
            --source https://nuget.pkg.github.com/${{ github.repository_owner }}/index.json \
            --api-key ${{ secrets.GITHUB_TOKEN }} \
            --skip-duplicate
      
      - name: Job Summary
        run: |
          echo "### ✅ NuGet Packages Published" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Packages:**" >> $GITHUB_STEP_SUMMARY
          ls -1 ./packages/*.nupkg | sed 's|./packages/||' | sed 's|^|- |' >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Feed URL:** https://nuget.pkg.github.com/PointCHealth/index.json" >> $GITHUB_STEP_SUMMARY
```

**Trigger Scenarios:**

| Trigger | When | Use Case |
|---------|------|----------|
| `push` to main | Shared library changes merged | Automatic publish on merge |
| `release` published | GitHub release created | Official versioned release |
| `workflow_dispatch` | Manual trigger | Ad-hoc package updates |

### Consuming Packages in Other Repositories

#### Step 1: Add nuget.config

**File**: `edi-sftp-connector/nuget.config` (or `edi-mappers/nuget.config`, etc.)

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <!-- Public NuGet feed for third-party packages -->
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
    
    <!-- GitHub Packages feed for EDI shared libraries -->
    <add key="github" value="https://nuget.pkg.github.com/PointCHealth/index.json" protocolVersion="3" />
  </packageSources>
  
  <packageSourceCredentials>
    <github>
      <add key="Username" value="%GITHUB_ACTOR%" />
      <add key="ClearTextPassword" value="%GITHUB_TOKEN%" />
    </github>
  </packageSourceCredentials>
</configuration>
```

**Why this works:**
- `%GITHUB_ACTOR%` and `%GITHUB_TOKEN%` are auto-populated by GitHub Actions
- No secrets needed in repository
- Works for both CI/CD and local development (with personal access token)

#### Step 2: Replace ProjectReference with PackageReference

**Before** (local development only):

```xml
<!-- edi-sftp-connector/SftpConnector.Function.csproj -->
<ItemGroup>
  <!-- This only works if edi-platform-core is cloned locally -->
  <ProjectReference Include="..\edi-platform-core\shared\EDI.Configuration\EDI.Configuration.csproj" />
  <ProjectReference Include="..\edi-platform-core\shared\EDI.Storage\EDI.Storage.csproj" />
  <ProjectReference Include="..\edi-platform-core\shared\EDI.Logging\EDI.Logging.csproj" />
</ItemGroup>
```

**After** (works everywhere):

```xml
<!-- edi-sftp-connector/SftpConnector.Function.csproj -->
<ItemGroup>
  <!-- Works in CI/CD and local dev -->
  <PackageReference Include="EDI.Configuration" Version="1.0.*" />
  <PackageReference Include="EDI.Storage" Version="1.0.*" />
  <PackageReference Include="EDI.Logging" Version="1.0.*" />
</ItemGroup>
```

**Version Strategies:**

| Pattern | Behavior | Use Case |
|---------|----------|----------|
| `Version="1.0.0"` | Exact version | Production stability (lock to specific version) |
| `Version="1.0.*"` | Latest patch | Get bug fixes automatically (1.0.0 → 1.0.1) |
| `Version="1.*"` | Latest minor | Get new features automatically (risky) |
| `Version="*"` | Latest version | Not recommended (unpredictable) |

**Recommended**: Use `1.0.*` for automatic patch updates, explicit version for major/minor.

#### Step 3: Update CI/CD Workflows

**File**: `edi-sftp-connector/.github/workflows/function-ci.yml`

```yaml
name: Function App CI

on:
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  packages: read  # Required to read from GitHub Packages

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      
      - name: Restore Dependencies
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Automatically injects token
        run: dotnet restore
      
      - name: Build
        run: dotnet build --configuration Release --no-restore
      
      - name: Test
        run: dotnet test --configuration Release --no-build
```

**Key Points:**
- Add `packages: read` permission
- Set `GITHUB_TOKEN` environment variable for restore
- No additional secrets needed (automatic authentication)

#### Step 4: Local Development Setup

**For local development**, developers need a Personal Access Token (PAT):

1. **Create PAT** (one-time setup):
   - Go to https://github.com/settings/tokens/new
   - Select scope: `read:packages`
   - Generate token and save it

2. **Configure local authentication**:

```powershell
# Windows PowerShell
$env:GITHUB_TOKEN = "ghp_your_personal_access_token_here"

# Or add to PowerShell profile for persistence
Add-Content $PROFILE "`n`$env:GITHUB_TOKEN = 'ghp_your_token_here'"
```

3. **Restore packages**:

```powershell
cd edi-sftp-connector
dotnet restore  # Will use GITHUB_TOKEN from environment
```

**Alternative**: Use `dotnet nuget add source` command:

```powershell
dotnet nuget add source https://nuget.pkg.github.com/PointCHealth/index.json \
  --name github \
  --username YOUR_GITHUB_USERNAME \
  --password ghp_your_personal_access_token \
  --store-password-in-clear-text
```

### Version Management Strategy

#### Semantic Versioning

All shared libraries follow **SemVer 2.0.0** (MAJOR.MINOR.PATCH):

- **MAJOR**: Breaking API changes (e.g., remove method, change signature)
- **MINOR**: New features, backward-compatible additions
- **PATCH**: Bug fixes, no API changes

**Examples:**
- `1.0.0` → `1.0.1`: Bug fix in X12 parser (consumer update optional)
- `1.0.1` → `1.1.0`: Add new method to IMessagePublisher (consumer update optional)
- `1.1.0` → `2.0.0`: Remove deprecated methods (consumer update **required**)

#### Version Tracking

**Update version in csproj** when making changes:

```xml
<Version>1.1.0</Version>
```

**Tag releases in Git**:

```powershell
git tag -a EDI.Configuration-v1.1.0 -m "Add support for HTTPS SFTP endpoints"
git push origin EDI.Configuration-v1.1.0
```

**Create GitHub Release** for official versions:
- Navigate to Releases → New Release
- Tag: `EDI.Configuration-v1.1.0`
- Title: `EDI.Configuration v1.1.0`
- Description: Changelog of what's new
- This triggers `release` workflow to publish packages

### Dependency Updates with Dependabot

Automated dependency updates across all consumer repositories:

**File**: `edi-mappers/.github/dependabot.yml`

```yaml
version: 2
updates:
  # Monitor NuGet dependencies
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
    open-pull-requests-limit: 5
    reviewers:
      - "PointCHealth/integration-team"
    labels:
      - "dependencies"
      - "nuget"
    commit-message:
      prefix: "chore"
      include: "scope"
    # Group shared library updates together
    groups:
      edi-shared-libraries:
        patterns:
          - "EDI.*"
        update-types:
          - "patch"
          - "minor"
```

**How it works:**
1. Dependabot checks for updates every Monday at 9 AM
2. Creates PRs for outdated packages
3. Groups all EDI.* packages into single PR (cleaner)
4. Auto-assigns integration team for review
5. CI runs tests on PR
6. Team reviews and merges

### Benefits of GitHub Packages Strategy

| Benefit | Impact |
|---------|--------|
| **Versioned Dependencies** | Track exactly which library version each function uses |
| **Independent Builds** | Each repo builds without needing others checked out |
| **Semantic Versioning** | Breaking changes = major version bump, clear migration path |
| **Caching** | CI/CD caches NuGet packages (5-10 minute build time savings) |
| **Automatic Auth** | `GITHUB_TOKEN` handles authentication (no secrets management) |
| **Dependency Updates** | Dependabot automates updates with testing |
| **Audit Trail** | Package publish history tracked in GitHub |
| **Rollback** | Consumers can pin to older versions if needed |

### Troubleshooting

#### Error: Unable to load the service index for source

**Cause**: Authentication not configured

**Solution**:
```powershell
# Check GITHUB_TOKEN is set
echo $env:GITHUB_TOKEN

# If empty, set it:
$env:GITHUB_TOKEN = "ghp_your_token"
```

#### Error: Response status code does not indicate success: 401 (Unauthorized)

**Cause**: Token doesn't have `read:packages` scope

**Solution**: Create new PAT with `read:packages` permission

#### Error: Package 'EDI.Configuration 1.0.0' is not found

**Cause**: Package not published yet, or version doesn't exist

**Solution**: Check published packages at https://github.com/orgs/PointCHealth/packages

#### CI build fails: dotnet restore fails

**Cause**: Missing `packages: read` permission or `GITHUB_TOKEN` not set

**Solution**: Add to workflow:
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

## Migration Path (If You're in a Monorepo)

If you're currently in a monorepo and want to migrate:

### Step 1: Identify Component Boundaries

```text
Current monorepo structure:
edi-platform/
├── shared/           → Extract to edi-platform-core
├── functions/
│   ├── routing/      → Extract to edi-sftp-connector
│   ├── mappers/      → Extract to edi-mappers
│   └── connectors/   → Extract to edi-connectors
├── databases/        → Extract to edi-database-* (3 repos)
└── configs/          → Extract to edi-partner-configs
```

### Step 2: Extract with History

Use `git filter-branch` or `git subtree` to preserve history:

```bash
# Example: Extract mappers to new repo
git clone edi-platform edi-mappers
cd edi-mappers
git filter-branch --subdirectory-filter functions/mappers -- --all
git remote set-url origin https://github.com/PointCHealth/edi-mappers.git
git push origin main
```

### Step 3: Update Dependencies

Convert internal references to NuGet packages:

```diff
- // Old: Direct project reference in monorepo
- <ProjectReference Include="../../shared/HealthcareEDI.X12/HealthcareEDI.X12.csproj" />

+ // New: NuGet package reference
+ <PackageReference Include="HealthcareEDI.X12" Version="1.0.0" />
```

### Step 4: Setup Workspace

Create `edi-platform` workspace repository with setup scripts.

---

## Success Metrics

### How to Measure Success

After adopting multi-repository strategy, track:

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Build time reduction** | 60% faster | Compare avg. build time before/after |
| **Deployment frequency** | 3x increase | Deployments per week |
| **Merge conflict rate** | 50% reduction | Conflicts per PR |
| **Team velocity** | 20% increase | Story points per sprint |
| **Onboarding time** | <1 week | New developer to first PR |

### Monitoring Dashboard

```kql
// GitHub Actions build time comparison
GitHubActions_CL
| where RepositoryName startswith "edi-"
| summarize 
    avg_build_minutes = avg(Duration) / 60,
    p95_build_minutes = percentile(Duration, 95) / 60,
    total_builds = count()
  by RepositoryName, bin(TimeGenerated, 1d)
| order by TimeGenerated desc
```

---

## FAQs

### Q: Won't multiple repositories be harder to manage?

**A:** The `edi-platform` workspace repository provides a unified experience. Developers still work in a single VS Code window with all repositories loaded. Setup scripts automate multi-repo operations.

### Q: How do we handle breaking changes in shared libraries?

**A:** Use semantic versioning (MAJOR.MINOR.PATCH) for NuGet packages. Breaking changes increment the MAJOR version. Consumer repos can upgrade at their own pace.

### Q: What about coordinated releases?

**A:** Use orchestrated GitHub Actions workflows (see section above) that trigger deployments across multiple repos with dependencies.

### Q: How do we search across all repositories?

**A:** VS Code workspace search works across all loaded folders. GitHub also supports cross-repo code search.

### Q: Won't we duplicate code across repositories?

**A:** Shared code lives in `edi-platform-core` as NuGet packages. No duplication needed.

### Q: How do we handle security vulnerabilities?

**A:** Dependabot automatically creates PRs in all repositories when vulnerabilities are detected. Review and merge across repos.

### Q: Can we switch back to monorepo if needed?

**A:** Yes, but it's unlikely you'll want to. Multi-repo advantages become more apparent at scale. If needed, git history can be merged.

---

## Conclusion

The **multi-repository strategy** is the recommended approach for the EDI Platform because it:

1. ✅ **Optimizes for independent deployment lifecycles**
2. ✅ **Reduces build times by 60%**
3. ✅ **Enables clear team ownership**
4. ✅ **Provides granular security controls**
5. ✅ **Maintains excellent developer experience via workspace orchestration**

This approach is documented in **[ADR-012: Multi-Repository Strategy](./17-architecture-decisions.md#14-adr-012-multi-repository-strategy)** and is already being used successfully by the EDI Platform team.

---

## Related Documentation

- **[ADR-012: Multi-Repository Strategy](./17-architecture-decisions.md#14-adr-012-multi-repository-strategy)** - Complete architecture decision record
- **[Deployment Automation Overview](./19-deployment-automation-overview.md)** - CI/CD architecture
- **[CI/CD Workflows](./21-cicd-workflows.md)** - GitHub Actions implementation
- **[GitHub Repos Created](../GITHUB_REPOS_CREATED.md)** - List of all repositories

---

**Document Owner:** EDI Platform Architecture Team  
**Last Updated:** October 7, 2025  
**Status:** ✅ Recommended Approach  
**Feedback:** Create GitHub issue or contact architecture team

