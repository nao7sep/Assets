# Project Structure

## Solution Overview

```
BrainLog/
├── BrainLog.sln
├── LICENSE
├── README.md
├── src/
│   ├── BrainLog/              # Avalonia UI application
│   └── BrainLogCore/          # Business logic library
└── tests/
    └── BrainLogTests/         # Unit tests
```

## Project: BrainLogCore

**Type**: Class Library (.NET 9.0)
**Purpose**: Domain models, services, and business logic

```
src/BrainLogCore/
├── BrainLogCore.csproj
├── Models/
│   ├── Entry.cs
│   ├── EntryState.cs
│   ├── Prompt.cs
│   └── AiResponse.cs
├── Dtos/
│   ├── EntryDto.cs
│   └── PromptDto.cs
├── Mappers/
│   ├── EntryMapper.cs
│   └── PromptMapper.cs
├── Services/
│   ├── IEntryService.cs
│   ├── EntryService.cs
│   ├── IFileWatchService.cs
│   ├── FileWatchService.cs
│   ├── IAutoSaveService.cs
│   ├── AutoSaveService.cs
│   ├── IPromptService.cs
│   ├── PromptService.cs
│   ├── IArchiveCooldownService.cs
│   ├── ArchiveCooldownService.cs
│   ├── IConfigurationService.cs
│   └── ConfigurationService.cs
├── Ai/
│   ├── IAiService.cs
│   ├── IAiServiceFactory.cs
│   ├── AiServiceFactory.cs
│   ├── OpenAiService.cs
│   └── GeminiService.cs
├── Configuration/
│   ├── AppSettings.cs
│   ├── StorageSettings.cs
│   ├── AutoSaveSettings.cs
│   ├── WorkflowSettings.cs
│   ├── NotificationSettings.cs
│   ├── AiSettings.cs
│   ├── OpenAiSettings.cs
│   ├── GeminiSettings.cs
│   ├── DefaultPromptSettings.cs
│   └── UserSettings.cs
├── Platform/
│   ├── ITrashService.cs
│   ├── WindowsTrashService.cs
│   └── MacOsTrashService.cs
└── Storage/
    ├── IStorageLocationResolver.cs
    └── StorageLocationResolver.cs
```

### BrainLogCore.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <RootNamespace>BrainLogCore</RootNamespace>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Configuration" Version="9.0.*" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="9.0.*" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Binder" Version="9.0.*" />
    <PackageReference Include="Microsoft.Extensions.Options" Version="9.0.*" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" Version="9.0.*" />
    <PackageReference Include="System.Text.Json" Version="9.0.*" />
  </ItemGroup>

  <!-- Windows-specific for trash functionality -->
  <ItemGroup Condition="'$(TargetFramework)' == 'net9.0'">
    <PackageReference Include="Microsoft.VisualBasic" Version="10.3.0" />
  </ItemGroup>
</Project>
```

## Project: BrainLog (UI)

**Type**: Avalonia MVVM Application (.NET 9.0)
**Purpose**: Cross-platform desktop UI

```
src/BrainLog/
├── BrainLog.csproj
├── App.axaml
├── App.axaml.cs
├── Program.cs
├── appsettings.json
├── ViewModels/
│   ├── ViewModelBase.cs
│   ├── MainViewModel.cs
│   ├── EntryListItemViewModel.cs
│   ├── EntryEditViewModel.cs
│   └── PromptManagementViewModel.cs
├── Views/
│   ├── MainWindow.axaml
│   ├── MainWindow.axaml.cs
│   ├── PromptManagementWindow.axaml
│   ├── PromptManagementWindow.axaml.cs
│   └── Dialogs/
│       ├── UnsavedChangesDialog.axaml
│       ├── SaveConfirmationDialog.axaml
│       ├── AiProgressDialog.axaml
│       ├── ErrorDialog.axaml
│       └── CorruptFileDialog.axaml
├── Converters/
│   ├── StateToBrushConverter.cs
│   ├── BoolToVisibilityConverter.cs
│   └── NullToVisibilityConverter.cs
├── Services/
│   └── DialogService.cs
└── Assets/
    └── (icons, etc.)
```

### BrainLog.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <RootNamespace>BrainLog</RootNamespace>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <BuiltInComInteropSupport>true</BuiltInComInteropSupport>
    <ApplicationManifest>app.manifest</ApplicationManifest>
    <AvaloniaUseCompiledBindingsByDefault>true</AvaloniaUseCompiledBindingsByDefault>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Avalonia" Version="11.*" />
    <PackageReference Include="Avalonia.Desktop" Version="11.*" />
    <PackageReference Include="Avalonia.Themes.Fluent" Version="11.*" />
    <PackageReference Include="Avalonia.Fonts.Inter" Version="11.*" />
    <PackageReference Include="Avalonia.ReactiveUI" Version="11.*" />
    <PackageReference Include="CommunityToolkit.Mvvm" Version="8.*" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="9.0.*" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\BrainLogCore\BrainLogCore.csproj" />
  </ItemGroup>

  <ItemGroup>
    <None Update="appsettings.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
  </ItemGroup>
</Project>
```

## Project: BrainLogTests

**Type**: xUnit Test Project (.NET 9.0)
**Purpose**: Unit tests for BrainLogCore

```
tests/BrainLogTests/
├── BrainLogTests.csproj
├── Mappers/
│   ├── EntryMapperTests.cs
│   └── PromptMapperTests.cs
├── Services/
│   ├── EntryServiceTests.cs
│   └── PromptServiceTests.cs
└── Ai/
    └── AiResponseParsingTests.cs
```

### BrainLogTests.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <RootNamespace>BrainLogTests</RootNamespace>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />
    <PackageReference Include="xunit" Version="2.*" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.*" />
    <PackageReference Include="Moq" Version="4.*" />
    <PackageReference Include="FluentAssertions" Version="6.*" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\BrainLogCore\BrainLogCore.csproj" />
  </ItemGroup>
</Project>
```

## BrainLog.sln

```
Microsoft Visual Studio Solution File, Format Version 12.00
# Visual Studio Version 17
VisualStudioVersion = 17.0.31903.59
MinimumVisualStudioVersion = 10.0.40219.1

Project("{FAE04EC0-301F-11D3-BF4B-00C04F79EFBC}") = "BrainLog", "src\BrainLog\BrainLog.csproj", "{GUID1}"
EndProject
Project("{FAE04EC0-301F-11D3-BF4B-00C04F79EFBC}") = "BrainLogCore", "src\BrainLogCore\BrainLogCore.csproj", "{GUID2}"
EndProject
Project("{FAE04EC0-301F-11D3-BF4B-00C04F79EFBC}") = "BrainLogTests", "tests\BrainLogTests\BrainLogTests.csproj", "{GUID3}"
EndProject

Global
  GlobalSection(SolutionConfigurationPlatforms) = preSolution
    Debug|Any CPU = Debug|Any CPU
    Release|Any CPU = Release|Any CPU
  EndGlobalSection
  GlobalSection(ProjectConfigurationPlatforms) = postSolution
    {GUID1}.Debug|Any CPU.ActiveCfg = Debug|Any CPU
    {GUID1}.Debug|Any CPU.Build.0 = Debug|Any CPU
    {GUID1}.Release|Any CPU.ActiveCfg = Release|Any CPU
    {GUID1}.Release|Any CPU.Build.0 = Release|Any CPU
    {GUID2}.Debug|Any CPU.ActiveCfg = Debug|Any CPU
    {GUID2}.Debug|Any CPU.Build.0 = Debug|Any CPU
    {GUID2}.Release|Any CPU.ActiveCfg = Release|Any CPU
    {GUID2}.Release|Any CPU.Build.0 = Release|Any CPU
    {GUID3}.Debug|Any CPU.ActiveCfg = Debug|Any CPU
    {GUID3}.Debug|Any CPU.Build.0 = Debug|Any CPU
    {GUID3}.Release|Any CPU.ActiveCfg = Release|Any CPU
    {GUID3}.Release|Any CPU.Build.0 = Release|Any CPU
  EndGlobalSection
  GlobalSection(SolutionProperties) = preSolution
    HideSolutionNode = FALSE
  EndGlobalSection
  GlobalSection(NestedProjects) = preSolution
  EndGlobalSection
EndGlobal
```

## Namespace Structure

| Folder | Namespace |
|--------|-----------|
| BrainLogCore/ | `BrainLogCore` |
| BrainLogCore/Models | `BrainLogCore.Models` |
| BrainLogCore/Dtos | `BrainLogCore.Dtos` |
| BrainLogCore/Mappers | `BrainLogCore.Mappers` |
| BrainLogCore/Services | `BrainLogCore.Services` |
| BrainLogCore/Ai | `BrainLogCore.Ai` |
| BrainLogCore/Configuration | `BrainLogCore.Configuration` |
| BrainLogCore/Platform | `BrainLogCore.Platform` |
| BrainLogCore/Storage | `BrainLogCore.Storage` |
| BrainLog/ | `BrainLog` |
| BrainLog/ViewModels | `BrainLog.ViewModels` |
| BrainLog/Views | `BrainLog.Views` |
| BrainLog/Views/Dialogs | `BrainLog.Views.Dialogs` |
| BrainLog/Converters | `BrainLog.Converters` |
| BrainLog/Services | `BrainLog.Services` |
| BrainLogTests/ | `BrainLogTests` |

## Dependency Graph

```
BrainLog (UI)
    │
    ├──► BrainLogCore (Library)
    │        │
    │        └──► Microsoft.Extensions.*
    │             System.Text.Json
    │
    ├──► Avalonia
    │    Avalonia.Desktop
    │    Avalonia.Themes.Fluent
    │    Avalonia.ReactiveUI
    │
    ├──► CommunityToolkit.Mvvm
    │
    └──► Microsoft.Extensions.DependencyInjection

BrainLogTests
    │
    ├──► BrainLogCore
    │
    ├──► xunit
    │
    ├──► Moq
    │
    └──► FluentAssertions
```

## DI Registration (Program.cs)

```csharp
public static class Program
{
    public static void Main(string[] args)
    {
        BuildAvaloniaApp()
            .StartWithClassicDesktopLifetime(args);
    }

    public static AppBuilder BuildAvaloniaApp()
    {
        var services = new ServiceCollection();
        ConfigureServices(services);
        var serviceProvider = services.BuildServiceProvider();

        return AppBuilder.Configure(() => new App(serviceProvider))
            .UsePlatformDetect()
            .WithInterFont()
            .LogToTrace()
            .UseReactiveUI();
    }

    private static void ConfigureServices(IServiceCollection services)
    {
        // Configuration
        var configuration = new ConfigurationBuilder()
            .SetBasePath(AppContext.BaseDirectory)
            .AddJsonFile("appsettings.json", optional: false)
            .Build();

        services.Configure<AppSettings>(configuration);
        services.Configure<StorageSettings>(configuration.GetSection("Storage"));
        services.Configure<AutoSaveSettings>(configuration.GetSection("AutoSave"));
        services.Configure<WorkflowSettings>(configuration.GetSection("Workflow"));
        services.Configure<AiSettings>(configuration.GetSection("AiServices"));
        services.Configure<DefaultPromptSettings>(configuration.GetSection("DefaultPrompt"));

        // Core services
        services.AddSingleton<IStorageLocationResolver, StorageLocationResolver>();
        services.AddSingleton<IConfigurationService, ConfigurationService>();
        services.AddSingleton<IEntryService, EntryService>();
        services.AddSingleton<IFileWatchService, FileWatchService>();
        services.AddSingleton<IAutoSaveService, AutoSaveService>();
        services.AddSingleton<IPromptService, PromptService>();
        services.AddSingleton<IArchiveCooldownService, ArchiveCooldownService>();

        // AI services
        services.AddSingleton<IAiService, OpenAiService>();
        services.AddSingleton<IAiService, GeminiService>();
        services.AddSingleton<IAiServiceFactory, AiServiceFactory>();

        // Platform services
        if (OperatingSystem.IsWindows())
            services.AddSingleton<ITrashService, WindowsTrashService>();
        else if (OperatingSystem.IsMacOS())
            services.AddSingleton<ITrashService, MacOsTrashService>();

        // ViewModels
        services.AddTransient<MainViewModel>();
        services.AddTransient<PromptManagementViewModel>();
    }
}
```
