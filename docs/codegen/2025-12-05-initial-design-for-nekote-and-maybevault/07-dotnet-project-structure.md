# .NET Project Structure Guide

## Overview

This document provides the exact commands and file contents needed to scaffold both the Nekote and MaybeVault projects.

---

## Nekote Project Setup

### Create Solution and Projects

```powershell
# Navigate to your repos folder
cd C:\Repositories

# Create Nekote directory
mkdir Nekote
cd Nekote

# Create solution
dotnet new sln -n Nekote

# Create main library
mkdir -p src/Nekote
dotnet new classlib -n Nekote -o src/Nekote -f net10.0

# Create test project
mkdir -p tests/NekoteTests
dotnet new xunit -n NekoteTests -o tests/NekoteTests -f net10.0

# Create sandbox console app (for manual testing and experiments)
mkdir -p tests/NekoteSandbox
dotnet new console -n NekoteSandbox -o tests/NekoteSandbox -f net10.0

# Add projects to solution
dotnet sln add src/Nekote/Nekote.csproj
dotnet sln add tests/NekoteTests/NekoteTests.csproj
dotnet sln add tests/NekoteSandbox/NekoteSandbox.csproj

# Add project references
dotnet add tests/NekoteTests/NekoteTests.csproj reference src/Nekote/Nekote.csproj
dotnet add tests/NekoteSandbox/NekoteSandbox.csproj reference src/Nekote/Nekote.csproj

# Add test packages
dotnet add tests/NekoteTests/NekoteTests.csproj package FluentAssertions
```

### Nekote.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <RootNamespace>Nekote</RootNamespace>

    <!-- Package metadata -->
    <PackageId>Nekote</PackageId>
    <Version>0.1.0</Version>
    <Authors>Your Name</Authors>
    <Description>Reusable utilities for file management, tagging, hashing, and storage patterns.</Description>
    <PackageTags>utilities;file-management;tagging;hashing;storage</PackageTags>
  </PropertyGroup>

</Project>
```

### Directory Structure

Create the following directories inside `src/Nekote/`:

```powershell
cd src/Nekote
mkdir IO
mkdir Hashing
mkdir Tagging
mkdir Categorization
mkdir Storage
mkdir Time
```

Final structure:
```
Nekote/
├── src/
│   └── Nekote/
│       ├── Nekote.csproj
│       ├── IO/
│       ├── Hashing/
│       ├── Tagging/
│       ├── Categorization/
│       ├── Storage/
│       └── Time/
├── tests/
│   ├── NekoteTests/
│   │   └── NekoteTests.csproj
│   └── NekoteSandbox/
│       └── NekoteSandbox.csproj
└── Nekote.sln
```

---

## MaybeVault Project Setup

### Create Solution and Projects

```powershell
# Navigate to your repos folder
cd C:\Repositories

# Create MaybeVault directory
mkdir MaybeVault
cd MaybeVault

# Create solution
dotnet new sln -n MaybeVault

# Create Avalonia app
mkdir -p src/MaybeVault
dotnet new avalonia.mvvm -n MaybeVault -o src/MaybeVault -f net10.0

# Create core library
mkdir -p src/MaybeVaultCore
dotnet new classlib -n MaybeVaultCore -o src/MaybeVaultCore -f net10.0

# Create test project
mkdir -p tests/MaybeVaultTests
dotnet new xunit -n MaybeVaultTests -o tests/MaybeVaultTests -f net10.0

# Add projects to solution
dotnet sln add src/MaybeVault/MaybeVault.csproj
dotnet sln add src/MaybeVaultCore/MaybeVaultCore.csproj
dotnet sln add tests/MaybeVaultTests/MaybeVaultTests.csproj

# Add project references
dotnet add src/MaybeVault/MaybeVault.csproj reference src/MaybeVaultCore/MaybeVaultCore.csproj
dotnet add tests/MaybeVaultTests/MaybeVaultTests.csproj reference src/MaybeVaultCore/MaybeVaultCore.csproj

# Add packages to MaybeVaultCore
dotnet add src/MaybeVaultCore/MaybeVaultCore.csproj package CommunityToolkit.Mvvm

# Add test packages
dotnet add tests/MaybeVaultTests/MaybeVaultTests.csproj package FluentAssertions
```

**Note:** If `avalonia.mvvm` template is not available, install it first:
```powershell
dotnet new install Avalonia.Templates
```

### Alternative: Manual Avalonia Setup

If the template doesn't work, create manually:

```powershell
# Create console app and convert to Avalonia
mkdir -p src/MaybeVault
dotnet new console -n MaybeVault -o src/MaybeVault -f net10.0

# Add Avalonia packages
dotnet add src/MaybeVault/MaybeVault.csproj package Avalonia
dotnet add src/MaybeVault/MaybeVault.csproj package Avalonia.Desktop
dotnet add src/MaybeVault/MaybeVault.csproj package Avalonia.Themes.Fluent
dotnet add src/MaybeVault/MaybeVault.csproj package Avalonia.Fonts.Inter
dotnet add src/MaybeVault/MaybeVault.csproj package Avalonia.Diagnostics --version "*-*"
```

### MaybeVaultCore.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <RootNamespace>MaybeVault.Core</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="CommunityToolkit.Mvvm" Version="8.4.*" />
  </ItemGroup>

  <!-- Reference Nekote when available -->
  <!-- Option 1: Project reference (during development) -->
  <!--
  <ItemGroup>
    <ProjectReference Include="..\..\..\Nekote\src\Nekote\Nekote.csproj" />
  </ItemGroup>
  -->

  <!-- Option 2: NuGet package (after publishing) -->
  <!--
  <ItemGroup>
    <PackageReference Include="Nekote" Version="0.1.0" />
  </ItemGroup>
  -->

</Project>
```

### MaybeVault.csproj (Avalonia App)

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <RootNamespace>MaybeVault</RootNamespace>
    <ApplicationIcon>Assets\app-icon.ico</ApplicationIcon>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Avalonia" Version="11.3.*" />
    <PackageReference Include="Avalonia.Desktop" Version="11.3.*" />
    <PackageReference Include="Avalonia.Themes.Fluent" Version="11.3.*" />
    <PackageReference Include="Avalonia.Fonts.Inter" Version="11.3.*" />
    <PackageReference Include="Avalonia.Diagnostics" Version="11.3.*" Condition="'$(Configuration)' == 'Debug'" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\MaybeVaultCore\MaybeVaultCore.csproj" />
  </ItemGroup>

</Project>
```

### Directory Structure for MaybeVaultCore

```powershell
cd src/MaybeVaultCore
mkdir Models
mkdir Services
mkdir Storage
mkdir ViewModels
mkdir Queries
```

### Directory Structure for MaybeVault (UI)

```powershell
cd src/MaybeVault
mkdir Views
mkdir Views/Dialogs
mkdir Controls
mkdir Converters
mkdir Assets
mkdir Assets/Icons
```

Final structure:
```
MaybeVault/
├── src/
│   ├── MaybeVault/
│   │   ├── MaybeVault.csproj
│   │   ├── App.axaml
│   │   ├── App.axaml.cs
│   │   ├── Program.cs
│   │   ├── Views/
│   │   │   └── Dialogs/
│   │   ├── Controls/
│   │   ├── Converters/
│   │   └── Assets/
│   │       └── Icons/
│   └── MaybeVaultCore/
│       ├── MaybeVaultCore.csproj
│       ├── Models/
│       ├── Services/
│       ├── Storage/
│       ├── ViewModels/
│       └── Queries/
├── tests/
│   └── MaybeVaultTests/
│       └── MaybeVaultTests.csproj
└── MaybeVault.sln
```

---

## Linking Nekote to MaybeVault

### Option 1: Project Reference (Development)

Add to `MaybeVaultCore.csproj`:
```xml
<ItemGroup>
  <ProjectReference Include="..\..\..\Nekote\src\Nekote\Nekote.csproj" />
</ItemGroup>
```

### Option 2: Local NuGet Package

```powershell
# In Nekote directory
dotnet pack src/Nekote/Nekote.csproj -o ./nupkg

# In MaybeVault directory, add local source
dotnet nuget add source C:\Repositories\Nekote\nupkg -n LocalNekote

# Add package reference
dotnet add src/MaybeVaultCore/MaybeVaultCore.csproj package Nekote
```

### Option 3: Published NuGet Package

Once Nekote is published to NuGet.org:
```powershell
dotnet add src/MaybeVaultCore/MaybeVaultCore.csproj package Nekote
```

---

## Git Setup

### Nekote .gitignore

```gitignore
# Build outputs
bin/
obj/
out/

# NuGet
*.nupkg
*.snupkg
nupkg/

# IDE
.vs/
.vscode/
.idea/
*.user
*.suo
*.userprefs

# macOS
.DS_Store

# Windows
Thumbs.db

# Test results
TestResults/
coverage/
```

### MaybeVault .gitignore

```gitignore
# Build outputs
bin/
obj/
out/

# IDE
.vs/
.vscode/
.idea/
*.user
*.suo
*.userprefs

# macOS
.DS_Store

# Windows
Thumbs.db

# Test results
TestResults/
coverage/

# Publish output
publish/
```

### MaybeVault Storage .gitattributes

For the actual vault storage (if stored in git):
```gitattributes
# Treat JSON as mergeable text
*.json text eol=lf

# Binary files
files/**/*.jpg binary
files/**/*.jpeg binary
files/**/*.png binary
files/**/*.gif binary
files/**/*.pdf binary
files/**/*.doc binary
files/**/*.docx binary
files/**/*.xls binary
files/**/*.xlsx binary
files/**/*.ppt binary
files/**/*.pptx binary
files/**/*.zip binary
files/**/*.rar binary
files/**/*.7z binary
```

---

## Build and Run Commands

### Nekote

```powershell
# Build
dotnet build

# Run tests
dotnet test

# Run sandbox
dotnet run --project tests/NekoteSandbox

# Create NuGet package
dotnet pack src/Nekote -c Release -o ./nupkg
```

### MaybeVault

```powershell
# Build
dotnet build

# Run app
dotnet run --project src/MaybeVault

# Run tests
dotnet test

# Publish (Windows)
dotnet publish src/MaybeVault -c Release -r win-x64 --self-contained

# Publish (macOS)
dotnet publish src/MaybeVault -c Release -r osx-x64 --self-contained

# Publish (Linux)
dotnet publish src/MaybeVault -c Release -r linux-x64 --self-contained
```

---

## Initial Files to Create

### Nekote: Start with these files

1. `src/Nekote/Hashing/IFileHasher.cs`
2. `src/Nekote/Hashing/Sha256FileHasher.cs`
3. `src/Nekote/IO/FileNameResolver.cs`
4. `src/Nekote/Storage/JsonDocumentStore.cs`
5. `src/Nekote/Time/TimestampInfo.cs`

### MaybeVault: Start with these files

1. `src/MaybeVaultCore/Models/VaultFile.cs`
2. `src/MaybeVaultCore/Models/VaultSettings.cs`
3. `src/MaybeVaultCore/Services/IVaultService.cs`
4. `src/MaybeVaultCore/Storage/IVaultStorage.cs`
5. `src/MaybeVault/Views/MainWindow.axaml`
6. `src/MaybeVault/ViewModels/MainWindowViewModel.cs`

---

## Development Order Recommendation

### Phase 1: Nekote Foundation
1. Set up Nekote repository and solution
2. Implement `Sha256FileHasher`
3. Implement `FileNameResolver`
4. Implement `JsonDocumentStore<T>`
5. Write tests for each component

### Phase 2: MaybeVault Core
1. Set up MaybeVault repository and solution
2. Reference Nekote (project reference)
3. Implement data models
4. Implement storage services
5. Implement `VaultService`

### Phase 3: MaybeVault UI
1. Create basic Avalonia window
2. Implement Add Files view
3. Implement Browse view
4. Implement Settings view
5. Add dialogs

### Phase 4: Polish
1. Error handling
2. Logging
3. Settings persistence
4. Platform-specific testing
