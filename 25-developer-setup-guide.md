# Developer Setup Guide

## Overview

This guide provides step-by-step instructions for setting up the complete EDI Platform development environment on your local machine. The EDI Platform uses a multi-repository architecture with Git submodules, and this workspace approach allows you to work efficiently across all components.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installing Development Tools](#installing-development-tools)
3. [Cloning the Repository](#cloning-the-repository)
4. [Visual Studio Code Setup](#visual-studio-code-setup)
5. [Project-Specific Setup](#project-specific-setup)
6. [Azure Configuration](#azure-configuration)
7. [Verifying Your Installation](#verifying-your-installation)
8. [Common Issues and Troubleshooting](#common-issues-and-troubleshooting)
9. [Next Steps](#next-steps)

---

## Prerequisites

### Required Software

Before you begin, ensure you have the following installed on your development machine:

| Tool | Minimum Version | Purpose | Download Link |
|------|----------------|---------|---------------|
| **Git** | 2.30+ | Version control and submodule management | [git-scm.com](https://git-scm.com/downloads) |
| **Node.js** | 20.x LTS | Frontend development (Admin Portal) | [nodejs.org](https://nodejs.org/) |
| **npm** | 10.x | Package manager for Node.js | Included with Node.js |
| **.NET SDK** | 9.0 | Backend development (Functions, API) | [dotnet.microsoft.com](https://dotnet.microsoft.com/download) |
| **Visual Studio Code** | Latest | Primary IDE | [code.visualstudio.com](https://code.visualstudio.com/) |
| **PowerShell** | 5.1+ / 7.x | Scripting and automation | Pre-installed on Windows / [PowerShell](https://github.com/PowerShell/PowerShell) |
| **Azure CLI** | Latest | Azure resource management | [docs.microsoft.com/cli/azure](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) |

### Optional but Recommended

| Tool | Purpose | Download Link |
|------|---------|---------------|
| **GitHub CLI** | Streamlined GitHub operations | [cli.github.com](https://cli.github.com/) |
| **Docker Desktop** | Container development and testing | [docker.com](https://www.docker.com/products/docker-desktop) |
| **Azure Storage Explorer** | Manage Azure Storage accounts | [azure.microsoft.com](https://azure.microsoft.com/en-us/features/storage-explorer/) |
| **SQL Server Management Studio (SSMS)** | Database management | [microsoft.com](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms) |
| **Postman** | API testing | [postman.com](https://www.postman.com/downloads/) |

### System Requirements

- **Operating System**: Windows 10/11, macOS 10.15+, or Linux (Ubuntu 20.04+)
- **RAM**: 16 GB minimum, 32 GB recommended
- **Disk Space**: 20 GB free space minimum
- **Internet Connection**: Required for initial setup and package downloads

### Access Requirements

Before starting, ensure you have:

- **GitHub Access**: Membership in the PointCHealth GitHub organization
- **Repository Access**: Read access to all EDI Platform repositories
- **Azure Access**: Access to the Azure subscription for the EDI Platform (for production work)
- **Azure AD Account**: For authentication and authorization testing

---

## Installing Development Tools

### Windows Installation

#### Install Git

1. Download Git from [git-scm.com](https://git-scm.com/downloads)
2. Run the installer with these recommended settings:
   - Select "Git from the command line and also from 3rd-party software"
   - Select "Use bundled OpenSSH"
   - Select "Checkout Windows-style, commit Unix-style line endings"
   - Select "Use Windows' default console window"
   - Enable "Enable Git Credential Manager"

3. Verify installation:
   ```powershell
   git --version
   # Output: git version 2.40.0 or later
   ```

#### Install Node.js and npm

1. Download Node.js 20.x LTS from [nodejs.org](https://nodejs.org/)
2. Run the installer (includes npm)
3. Verify installation:
   ```powershell
   node --version
   # Output: v20.x.x
   
   npm --version
   # Output: 10.x.x
   ```

#### Install .NET SDK 9.0

1. Download .NET SDK 9.0 from [dotnet.microsoft.com](https://dotnet.microsoft.com/download)
2. Run the installer
3. Verify installation:
   ```powershell
   dotnet --version
   # Output: 9.0.x
   ```

#### Install Visual Studio Code

1. Download VS Code from [code.visualstudio.com](https://code.visualstudio.com/)
2. Run the installer with these options:
   - ✅ Add "Open with Code" action to Windows Explorer file context menu
   - ✅ Add "Open with Code" action to Windows Explorer directory context menu
   - ✅ Register Code as an editor for supported file types
   - ✅ Add to PATH

3. Verify installation:
   ```powershell
   code --version
   ```

#### Install Azure CLI

1. Download the MSI installer from [Azure CLI Install](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows)
2. Run the installer
3. Verify installation:
   ```powershell
   az --version
   # Output: azure-cli 2.x.x
   ```

4. Login to Azure (optional for initial setup):
   ```powershell
   az login
   ```

### macOS Installation

Use Homebrew for easy installation:

```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Git
brew install git

# Install Node.js 20 LTS
brew install node@20

# Install .NET SDK 9
brew install --cask dotnet-sdk

# Install Visual Studio Code
brew install --cask visual-studio-code

# Install Azure CLI
brew install azure-cli

# Verify installations
git --version
node --version
npm --version
dotnet --version
code --version
az --version
```

### Linux (Ubuntu/Debian) Installation

```bash
# Update package manager
sudo apt update && sudo apt upgrade -y

# Install Git
sudo apt install git -y

# Install Node.js 20.x
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Install .NET SDK 9
wget https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb
sudo apt update
sudo apt install -y dotnet-sdk-9.0

# Install Visual Studio Code
sudo snap install --classic code

# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Verify installations
git --version
node --version
npm --version
dotnet --version
code --version
az --version
```

---

## Cloning the Repository

The EDI Platform uses a **multi-repository architecture** with Git submodules. The main `edi-platform` repository contains references to all component repositories as submodules.

### Understanding the Repository Structure

```
edi-platform/                          # Main workspace repository
├── edi-admin-portal-backend/          # Submodule: Admin Portal API
├── edi-admin-portal-frontend/         # Submodule: Admin Portal Angular app
├── edi-azure-infrastructure/          # Submodule: Bicep/Terraform infrastructure
├── edi-database-controlnumbers/       # Submodule: Control Numbers DB
├── edi-database-eventstore/           # Submodule: Event Store DB
├── edi-database-sftptracking/         # Submodule: SFTP Tracking DB
├── edi-documentation/                 # Submodule: Platform documentation
├── edi-mappers/                       # Submodule: EDI mapper functions
├── edi-partner-configs/               # Submodule: Trading partner configs
├── edi-platform-core/                 # Submodule: Core libraries
├── edi-sftp-connector/                # Submodule: SFTP connector function
├── scripts/                           # Setup and automation scripts
├── .gitmodules                        # Submodule configuration
└── edi-platform.code-workspace        # VS Code workspace configuration
```

### Clone the Repository with Submodules

#### Option 1: Clone with Submodules in One Command (Recommended)

This is the fastest way to get started:

```powershell
# Windows (PowerShell)
cd c:\repos
git clone --recurse-submodules https://github.com/PointCHealth/edi-platform.git
cd edi-platform
```

```bash
# macOS/Linux
cd ~/repos
git clone --recurse-submodules https://github.com/PointCHealth/edi-platform.git
cd edi-platform
```

#### Option 2: Clone in Steps (More Control)

If you prefer to clone the main repository first, then initialize submodules:

```powershell
# Clone the main repository
git clone https://github.com/PointCHealth/edi-platform.git
cd edi-platform

# Initialize and update all submodules
git submodule init
git submodule update

# Or combine both commands
git submodule update --init --recursive
```

#### Option 3: Clone with Specific Branch

If you need to work on a specific branch:

```powershell
# Clone main repository on a specific branch
git clone -b develop https://github.com/PointCHealth/edi-platform.git
cd edi-platform

# Initialize submodules and checkout their respective branches
git submodule update --init --recursive --remote
```

### Verify Submodule Status

After cloning, verify all submodules are properly initialized:

```powershell
# Check submodule status
git submodule status

# Expected output (example):
# +abc123... edi-admin-portal-backend (heads/main)
# +def456... edi-admin-portal-frontend (heads/main)
# +ghi789... edi-azure-infrastructure (heads/main)
# ... (all submodules listed with their commit hashes)
```

### Update Submodules to Latest Commits

To pull the latest changes for all submodules:

```powershell
# Update all submodules to their latest commit on the main branch
git submodule update --remote --merge

# Or update recursively
git submodule update --init --recursive --remote
```

### Working with Individual Submodules

Each submodule is a full Git repository. To work within a submodule:

```powershell
# Navigate to a submodule
cd edi-admin-portal-backend

# Check current branch
git branch

# Create a new branch
git checkout -b feature/my-new-feature

# Make changes and commit
git add .
git commit -m "Add new feature"

# Push to remote
git push origin feature/my-new-feature

# Return to main workspace
cd ..
```

### Important Git Submodule Concepts

- **Submodules are pinned to specific commits**: The main repository tracks a specific commit hash for each submodule
- **Updating submodules requires two steps**: 
  1. Update the submodule itself (inside the submodule directory)
  2. Commit the new submodule reference in the main repository
- **Always check submodule status**: Before committing in the main repo, verify submodule states with `git submodule status`

---

## Visual Studio Code Setup

### Open the Workspace

After cloning the repository, open the pre-configured multi-root workspace:

```powershell
# From the edi-platform directory
code edi-platform.code-workspace
```

This will open all submodules as separate workspace folders in VS Code.

### Install Required Extensions

The workspace is configured with recommended extensions. When you open the workspace, VS Code will prompt you to install them.

#### Core Extensions (Required)

Install these extensions from the Extensions panel (`Ctrl+Shift+X` / `Cmd+Shift+X`):

1. **C# Dev Kit** (`ms-dotnettools.csdevkit`)
   - Official C# extension for .NET development
   - Includes IntelliSense, debugging, and testing

2. **C#** (`ms-dotnettools.csharp`)
   - C# language support and debugging
   - Required for Azure Functions and .NET projects

3. **Azure Functions** (`ms-azuretools.vscode-azurefunctions`)
   - Create, debug, and deploy Azure Functions
   - Required for SFTP Connector and Mapper projects

4. **Bicep** (`ms-azuretools.vscode-bicep`)
   - Infrastructure as Code support
   - Required for Azure infrastructure development

5. **Azure Resources** (`ms-azuretools.vscode-azureresourcegroups`)
   - Manage Azure resources directly from VS Code
   - View and manage Function Apps, Storage, Service Bus, etc.

6. **Azure Account** (`ms-vscode.azure-account`)
   - Sign in to Azure and manage subscriptions
   - Required for deployment and resource management

7. **Prettier - Code Formatter** (`esbenp.prettier-vscode`)
   - Code formatting for JSON, Markdown, and other files
   - Configured to format on save

8. **GitHub Copilot** (`GitHub.copilot`) *(Optional but highly recommended)*
   - AI-powered code completion and assistance
   - Significantly improves development speed

#### Admin Portal Extensions (Required for Frontend Work)

If you'll be working on the Admin Portal frontend:

9. **Angular Language Service** (`Angular.ng-template`)
   - Angular template IntelliSense and validation
   - Required for Admin Portal frontend development

10. **Tailwind CSS IntelliSense** (`bradlc.vscode-tailwindcss`)
    - Autocomplete and linting for Tailwind CSS
    - Enhances Admin Portal UI development

11. **ESLint** (`dbaeumer.vscode-eslint`)
    - JavaScript/TypeScript linting
    - Enforces code quality standards

#### Optional but Recommended Extensions

12. **GitLens** (`eamodio.gitlens`)
    - Supercharges Git capabilities
    - Visualize code authorship, navigate history

13. **REST Client** (`humao.rest-client`)
    - Test HTTP requests directly in VS Code
    - Alternative to Postman for API testing

14. **Azure Storage** (`ms-azuretools.vscode-azurestorage`)
    - Browse and manage Azure Storage accounts
    - View blobs, queues, and tables

15. **SQL Server (mssql)** (`ms-mssql.mssql`)
    - Connect to SQL Server and Azure SQL
    - Run queries and manage databases

16. **PowerShell** (`ms-vscode.powershell`)
    - PowerShell language support
    - Required for running setup scripts

### Install Extensions via Command Line

You can install all required extensions using the command line:

```powershell
# Core extensions
code --install-extension ms-dotnettools.csdevkit
code --install-extension ms-dotnettools.csharp
code --install-extension ms-azuretools.vscode-azurefunctions
code --install-extension ms-azuretools.vscode-bicep
code --install-extension ms-azuretools.vscode-azureresourcegroups
code --install-extension ms-vscode.azure-account
code --install-extension esbenp.prettier-vscode
code --install-extension GitHub.copilot

# Admin Portal extensions
code --install-extension Angular.ng-template
code --install-extension bradlc.vscode-tailwindcss
code --install-extension dbaeumer.vscode-eslint

# Optional extensions
code --install-extension eamodio.gitlens
code --install-extension humao.rest-client
code --install-extension ms-azuretools.vscode-azurestorage
code --install-extension ms-mssql.mssql
code --install-extension ms-vscode.powershell
```

### Configure VS Code Settings

The workspace includes pre-configured settings in `.vscode/settings.json`. Verify these settings:

```jsonc
{
  // Format on save
  "editor.formatOnSave": true,
  
  // Organize imports automatically
  "editor.codeActionsOnSave": {
    "source.organizeImports": "explicit"
  },
  
  // C# formatting
  "[csharp]": {
    "editor.defaultFormatter": "ms-dotnettools.csharp"
  },
  
  // Bicep formatting
  "[bicep]": {
    "editor.defaultFormatter": "ms-azuretools.vscode-bicep"
  },
  
  // JSON formatting
  "[json]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  
  // Exclude build artifacts from file explorer
  "files.exclude": {
    "**/.git": true,
    "**/bin": true,
    "**/obj": true,
    "**/__pycache__": true,
    "**/.vs": true
  },
  
  // Exclude from search
  "search.exclude": {
    "**/node_modules": true,
    "**/bin": true,
    "**/obj": true
  }
}
```

### Configure VS Code Tasks

The workspace includes pre-configured tasks for common operations. View available tasks:

- Press `Ctrl+Shift+P` (Windows/Linux) or `Cmd+Shift+P` (macOS)
- Type "Tasks: Run Task"
- Select from available tasks:
  - **Build (functions)** - Build the SFTP Connector Azure Function
  - **Clean (functions)** - Clean build artifacts
  - **Publish (functions)** - Publish Azure Function for deployment
  - **Admin Portal: Build Backend** - Build the Admin Portal API
  - **Admin Portal: Run Backend** - Run the Admin Portal API locally
  - **Admin Portal: Run Frontend** - Run the Admin Portal Angular app
  - **Admin Portal: Start Both (Backend + Frontend)** - Run both simultaneously

### Workspace Folder Structure in VS Code

The multi-root workspace organizes projects into logical groups:

```
📁 EDI Platform Workspace
  ├── 📂 Core Platform (edi-platform-core)
  ├── 📂 Mappers (edi-mappers)
  ├── 📂 Connectors (edi-connectors)
  ├── 📂 Partner Configs (edi-partner-configs)
  ├── 📂 Data Platform (edi-data-platform)
  ├── 📂 Azure Infrastructure (edi-azure-infrastructure)
  ├── 📂 Admin Portal Backend (edi-admin-portal-backend)
  ├── 📂 Admin Portal Frontend (edi-admin-portal-frontend)
  ├── 📂 Documentation (edi-documentation)
  └── 📂 Databases (edi-database-*)
```

---

## Project-Specific Setup

### 1. Admin Portal Backend (.NET Core 9 API)

#### Navigate to the Project

```powershell
cd edi-admin-portal-backend/ArgusAdminPortal.API
```

#### Restore NuGet Packages

```powershell
dotnet restore
```

#### Build the Project

```powershell
dotnet build
```

#### Configure Local Settings

Create `appsettings.Development.json` (this file is not committed to Git):

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedOrigins": [
    "http://localhost:4200"
  ],
  "ApplicationInsights": {
    "WorkspaceId": "your-log-analytics-workspace-id",
    "QueryTimeoutSeconds": 30,
    "CacheExpirationSeconds": 30
  }
}
```

#### Run the API Locally

```powershell
dotnet run
```

The API will start on `https://localhost:5122`. Swagger UI is available at the root URL.

#### Using VS Code Tasks

Alternatively, use the VS Code task:
- Press `Ctrl+Shift+P` → "Tasks: Run Task" → "Admin Portal: Run Backend"

### 2. Admin Portal Frontend (Angular 17+)

#### Navigate to the Project

```powershell
cd edi-admin-portal-frontend/argus-admin-portal
```

#### Install npm Packages

```powershell
npm install
```

This will install all dependencies defined in `package.json`, including:
- Angular 17+
- Tailwind CSS 3.4.17
- Angular Material
- RxJS
- Other frontend dependencies

#### Run the Development Server

```powershell
npm run start
```

The app will start on `http://localhost:4200` and automatically open in your browser.

#### Using VS Code Tasks

Alternatively, use the VS Code task:
- Press `Ctrl+Shift+P` → "Tasks: Run Task" → "Admin Portal: Run Frontend"

#### Run Both Backend and Frontend Together

Use the combined task to run both simultaneously:
- Press `Ctrl+Shift+P` → "Tasks: Run Task" → "Admin Portal: Start Both (Backend + Frontend)"

### 3. SFTP Connector (Azure Function)

#### Navigate to the Project

```powershell
cd edi-sftp-connector
```

#### Restore NuGet Packages

```powershell
dotnet restore
```

#### Configure Local Settings

Create `local.settings.json` (not committed to Git):

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "ServiceBusConnection": "your-service-bus-connection-string",
    "StorageConnection": "your-storage-connection-string"
  }
}
```

#### Run the Function Locally

```powershell
func start
```

Or use the Azure Functions extension in VS Code (F5 to debug).

### 4. EDI Mappers (Azure Functions)

#### Navigate to the Project

```powershell
cd edi-mappers
```

#### Restore NuGet Packages

```powershell
dotnet restore
```

#### Configure Local Settings

Create `local.settings.json` for each mapper function.

#### Run Functions Locally

Use the Azure Functions extension in VS Code or:

```powershell
func start
```

### 5. Database Projects

#### Control Numbers Database

```powershell
cd edi-database-controlnumbers/EDI.ControlNumbers.Database
dotnet build
```

#### Event Store Database

```powershell
cd edi-database-eventstore/EDI.EventStore.Database
dotnet build
```

#### SFTP Tracking Database (EF Core Migrations)

```powershell
cd edi-database-sftptracking/EDI.SftpTracking.Migrations
dotnet restore
dotnet ef database update
```

---

## Azure Configuration

### Sign in to Azure

```powershell
az login
```

This will open a browser window for authentication. Sign in with your Azure AD account.

### Set Your Subscription

If you have multiple Azure subscriptions:

```powershell
# List subscriptions
az account list --output table

# Set the active subscription
az account set --subscription "Your Subscription Name or ID"
```

### Verify Azure Access

```powershell
# List resource groups
az group list --output table

# List function apps (if deployed)
az functionapp list --output table
```

### Configure Azure CLI Defaults (Optional)

Set default resource group and location:

```powershell
az configure --defaults group=your-resource-group location=eastus
```

### Application Insights Access

To use Application Insights integration in the Admin Portal:

1. Obtain the **Log Analytics Workspace ID** from your Azure portal or DevOps team
2. Add it to `appsettings.Development.json` in the Admin Portal backend
3. Ensure you have **Log Analytics Reader** role on the workspace

```powershell
# Check your access to the Log Analytics Workspace
az role assignment list --assignee your-user@domain.com --all
```

---

## Verifying Your Installation

### Run the Complete Verification Script

Create a file `verify-setup.ps1` in the `edi-platform` root:

```powershell
# Verification Script
Write-Host "=== EDI Platform Setup Verification ===" -ForegroundColor Cyan

# Check Git
Write-Host "`nChecking Git..." -ForegroundColor Yellow
git --version
if ($LASTEXITCODE -eq 0) { Write-Host "✓ Git installed" -ForegroundColor Green } else { Write-Host "✗ Git not found" -ForegroundColor Red }

# Check Node.js
Write-Host "`nChecking Node.js..." -ForegroundColor Yellow
node --version
if ($LASTEXITCODE -eq 0) { Write-Host "✓ Node.js installed" -ForegroundColor Green } else { Write-Host "✗ Node.js not found" -ForegroundColor Red }

# Check npm
Write-Host "`nChecking npm..." -ForegroundColor Yellow
npm --version
if ($LASTEXITCODE -eq 0) { Write-Host "✓ npm installed" -ForegroundColor Green } else { Write-Host "✗ npm not found" -ForegroundColor Red }

# Check .NET SDK
Write-Host "`nChecking .NET SDK..." -ForegroundColor Yellow
dotnet --version
if ($LASTEXITCODE -eq 0) { Write-Host "✓ .NET SDK installed" -ForegroundColor Green } else { Write-Host "✗ .NET SDK not found" -ForegroundColor Red }

# Check Azure CLI
Write-Host "`nChecking Azure CLI..." -ForegroundColor Yellow
az --version | Select-String "azure-cli"
if ($LASTEXITCODE -eq 0) { Write-Host "✓ Azure CLI installed" -ForegroundColor Green } else { Write-Host "✗ Azure CLI not found" -ForegroundColor Red }

# Check VS Code
Write-Host "`nChecking Visual Studio Code..." -ForegroundColor Yellow
code --version
if ($LASTEXITCODE -eq 0) { Write-Host "✓ VS Code installed" -ForegroundColor Green } else { Write-Host "✗ VS Code not found" -ForegroundColor Red }

# Check Submodules
Write-Host "`nChecking Git Submodules..." -ForegroundColor Yellow
$submoduleStatus = git submodule status
if ($submoduleStatus -match "^-") {
    Write-Host "✗ Some submodules not initialized" -ForegroundColor Red
    Write-Host "Run: git submodule update --init --recursive" -ForegroundColor Yellow
} else {
    Write-Host "✓ All submodules initialized" -ForegroundColor Green
}

# Check Azure Login
Write-Host "`nChecking Azure Login..." -ForegroundColor Yellow
$azAccount = az account show 2>$null
if ($azAccount) {
    Write-Host "✓ Logged in to Azure" -ForegroundColor Green
    az account show --query "{Subscription:name, User:user.name}" --output table
} else {
    Write-Host "✗ Not logged in to Azure" -ForegroundColor Yellow
    Write-Host "Run: az login" -ForegroundColor Yellow
}

Write-Host "`n=== Verification Complete ===" -ForegroundColor Cyan
```

Run the script:

```powershell
.\verify-setup.ps1
```

### Manual Verification Checklist

- [ ] Git installed and accessible from command line
- [ ] Node.js 20.x and npm installed
- [ ] .NET SDK 9.0 installed
- [ ] Visual Studio Code installed with all required extensions
- [ ] Azure CLI installed
- [ ] Repository cloned with all submodules initialized
- [ ] Admin Portal backend builds and runs (`dotnet run`)
- [ ] Admin Portal frontend builds and runs (`npm run start`)
- [ ] SFTP Connector builds (`dotnet build`)
- [ ] Azure CLI authenticated (`az login`)
- [ ] Application Insights workspace ID configured (for Admin Portal)

---

## Common Issues and Troubleshooting

### Git Submodule Issues

#### Submodules Show as "Changed" but No Changes Made

**Problem**: `git status` shows submodules as modified even though you haven't changed them.

**Solution**:
```powershell
# Reset submodules to the commit referenced by the main repo
git submodule update --init --recursive
```

#### Submodule is Empty or Missing Files

**Problem**: Submodule directory exists but is empty.

**Solution**:
```powershell
# Initialize and update the specific submodule
git submodule update --init edi-admin-portal-backend
```

#### Submodule Stuck on Detached HEAD

**Problem**: Submodule is in detached HEAD state.

**Solution**:
```powershell
cd edi-admin-portal-backend
git checkout main
cd ..
```

### Admin Portal Frontend Issues

#### Tailwind CSS Not Working

**Problem**: Tailwind styles not applied or build errors.

**Solution**: Ensure Tailwind CSS 3.4.17 is installed (v4 is not compatible):
```powershell
npm install -D tailwindcss@3.4.17 postcss autoprefixer
```

#### npm Install Fails

**Problem**: Package installation fails with errors.

**Solution**:
```powershell
# Clear npm cache
npm cache clean --force

# Delete node_modules and package-lock.json
Remove-Item -Recurse -Force node_modules
Remove-Item -Force package-lock.json

# Reinstall
npm install
```

#### Port 4200 Already in Use

**Problem**: Another process is using port 4200.

**Solution**:
```powershell
# Windows: Kill process on port 4200
netstat -ano | findstr :4200
taskkill /PID <PID> /F

# Or run on a different port
npm run start -- --port 4201
```

### Admin Portal Backend Issues

#### Port 5122 Already in Use

**Problem**: Another process is using port 5122.

**Solution**: Configure a different port in `launchSettings.json`:
```json
"applicationUrl": "https://localhost:5123;http://localhost:5124"
```

#### CORS Errors in Browser

**Problem**: Frontend cannot connect to backend due to CORS policy.

**Solution**: Verify `appsettings.Development.json` includes frontend origin:
```json
"AllowedOrigins": ["http://localhost:4200"]
```

#### Database Connection Errors

**Problem**: Cannot connect to SQL Server.

**Solution**: Ensure connection string is correct and SQL Server is running. Use local SQL Server or Azure SQL Database.

### Azure Function Issues

#### "Cannot find module" Errors

**Problem**: Azure Functions Runtime cannot find dependencies.

**Solution**:
```powershell
dotnet clean
dotnet restore
dotnet build
```

#### local.settings.json Not Found

**Problem**: Function fails to start due to missing configuration.

**Solution**: Create `local.settings.json` with required connection strings (see project-specific setup).

#### Azure Storage Emulator Issues

**Problem**: Azure Storage emulator not running.

**Solution**:
```powershell
# Install Azurite (modern emulator)
npm install -g azurite

# Start Azurite
azurite --silent --location c:\azurite --debug c:\azurite\debug.log
```

### .NET Build Issues

#### NuGet Restore Fails

**Problem**: Package restore fails.

**Solution**:
```powershell
# Clear NuGet cache
dotnet nuget locals all --clear

# Restore packages
dotnet restore
```

#### "SDK not found" Errors

**Problem**: .NET SDK 9.0 not detected.

**Solution**: Verify installation:
```powershell
dotnet --list-sdks
# Ensure 9.0.x is listed
```

If not listed, reinstall .NET SDK 9.0.

### Azure CLI Issues

#### "az" Command Not Found

**Problem**: Azure CLI not in PATH.

**Solution**:
- Restart your terminal/PowerShell
- Reinstall Azure CLI
- Manually add to PATH: `C:\Program Files\Microsoft SDKs\Azure\CLI2\wbin`

#### Authentication Expired

**Problem**: `az` commands fail with authentication error.

**Solution**:
```powershell
az logout
az login
```

### VS Code Issues

#### Extensions Not Loading

**Problem**: Installed extensions don't work.

**Solution**:
- Reload VS Code: `Ctrl+Shift+P` → "Developer: Reload Window"
- Check for extension updates
- Reinstall problematic extensions

#### Tasks Not Showing Up

**Problem**: VS Code tasks missing from task list.

**Solution**:
- Ensure you opened the workspace file (`edi-platform.code-workspace`)
- Check `.vscode/tasks.json` exists
- Reload window

---

## Next Steps

### Learning Resources

- **EDI Platform Documentation**: Explore `edi-documentation/` for comprehensive guides
  - [00-executive-overview.md](./00-executive-overview.md) - Platform overview
  - [24-argus-administration-portal.md](./24-argus-administration-portal.md) - Admin Portal guide
  - [08-monitoring-operations.md](./08-monitoring-operations.md) - Monitoring and KQL queries
  - [09-security-compliance.md](./09-security-compliance.md) - Security practices

- **Architecture Specification**: Review the [ai-adf-edi-spec](https://github.com/PointCHealth/ai-adf-edi-spec) repository

- **.NET Documentation**: [Microsoft Learn - .NET](https://learn.microsoft.com/en-us/dotnet/)

- **Angular Documentation**: [Angular.io](https://angular.io/docs)

- **Azure Functions**: [Azure Functions Documentation](https://learn.microsoft.com/en-us/azure/azure-functions/)

### Development Workflow

1. **Create a Feature Branch**:
   ```powershell
   cd edi-admin-portal-backend
   git checkout -b feature/my-new-feature
   ```

2. **Make Changes and Test Locally**

3. **Commit and Push**:
   ```powershell
   git add .
   git commit -m "Add new feature"
   git push origin feature/my-new-feature
   ```

4. **Create Pull Request**: Go to GitHub and create a PR

5. **Code Review**: Wait for approval from team members

6. **Merge**: Once approved, merge to main branch

### Branching Strategy

- **main**: Production-ready code
- **develop**: Integration branch (if used)
- **feature/***: Feature development branches
- **hotfix/***: Production hotfixes
- **bugfix/***: Bug fixes

### Getting Help

- **Team Chat**: Contact the EDI Platform team in Microsoft Teams
- **GitHub Issues**: Create issues in the respective repository
- **Documentation**: Check `edi-documentation/` for detailed guides
- **Copilot Instructions**: Review `.github/copilot-instructions-admin-portal.md` for AI assistance

### Contributing

1. Follow the branching strategy
2. Write meaningful commit messages
3. Add unit tests for new features
4. Update documentation as needed
5. Request code reviews before merging
6. Keep dependencies up to date

---

## GitHub Resources

### Key Repositories

All repositories are under the [PointCHealth](https://github.com/PointCHealth) GitHub organization:

| Repository | URL | Purpose |
|------------|-----|---------|
| **edi-platform** | [github.com/PointCHealth/edi-platform](https://github.com/PointCHealth/edi-platform) | Main workspace |
| **edi-admin-portal-backend** | [github.com/PointCHealth/edi-admin-portal-backend](https://github.com/PointCHealth/edi-admin-portal-backend) | Admin Portal API |
| **edi-admin-portal-frontend** | [github.com/PointCHealth/edi-admin-portal-frontend](https://github.com/PointCHealth/edi-admin-portal-frontend) | Admin Portal UI |
| **edi-azure-infrastructure** | [github.com/PointCHealth/edi-azure-infrastructure](https://github.com/PointCHealth/edi-azure-infrastructure) | Infrastructure as Code |
| **edi-documentation** | [github.com/PointCHealth/edi-documentation](https://github.com/PointCHealth/edi-documentation) | Platform documentation |
| **edi-sftp-connector** | [github.com/PointCHealth/edi-sftp-connector](https://github.com/PointCHealth/edi-sftp-connector) | SFTP Connector function |
| **edi-mappers** | [github.com/PointCHealth/edi-mappers](https://github.com/PointCHealth/edi-mappers) | EDI mapper functions |

### GitHub Actions (CI/CD)

Each repository may have CI/CD pipelines configured in `.github/workflows/`. Check for:

- **Build workflows**: Automated builds on every push/PR
- **Test workflows**: Automated unit and integration tests
- **Deployment workflows**: Automated deployment to Azure

### Branch Protection Rules

Main branches are protected with rules:
- Require pull request reviews before merging
- Require status checks to pass before merging
- Prevent force pushes and deletions

### Pull Request Template

When creating PRs, follow the template (if configured):
- Description of changes
- Related issues/tickets
- Testing performed
- Checklist for reviewers

---

## Summary

You now have a fully configured EDI Platform development environment! You should be able to:

✅ Clone the repository with all submodules  
✅ Open the multi-root workspace in VS Code  
✅ Build and run the Admin Portal backend and frontend  
✅ Build Azure Functions (SFTP Connector, Mappers)  
✅ Connect to Azure resources  
✅ Start developing new features  

**Happy coding! 🚀**

---

## Document Information

- **Version**: 1.0
- **Last Updated**: October 8, 2025
- **Author**: EDI Platform Team
- **Related Documents**: 
  - [24-argus-administration-portal.md](./24-argus-administration-portal.md)
  - [00-executive-overview.md](./00-executive-overview.md)
  - [08-monitoring-operations.md](./08-monitoring-operations.md)
