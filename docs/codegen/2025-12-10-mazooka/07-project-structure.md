# Project Structure

## Solution Layout

```
Mazooka.sln
├── Mazooka/                            # Console application
│   ├── Commands/                       # CLI command implementations
│   │   ├── RunCommand.cs
│   │   ├── SetupOAuth2Command.cs
│   │   ├── TestAuthCommand.cs
│   │   └── AuditCommand.cs
│   ├── Program.cs                      # Entry point
│   ├── ConfigManager.cs                # Configuration loading
│   ├── appsettings.json                # Default configuration
│   └── Mazooka.csproj
│
├── Mazooka.Core/                       # Core library
│   ├── Authentication/
│   │   ├── IAuthenticator.cs
│   │   ├── PasswordAuthenticator.cs
│   │   ├── OAuth2Authenticator.cs
│   │   ├── IOAuth2Provider.cs
│   │   ├── GmailOAuth2Provider.cs
│   │   └── AuthenticatorFactory.cs
│   │
│   ├── Imap/
│   │   ├── IImapClient.cs
│   │   ├── ImapClientWrapper.cs
│   │   ├── MessageFetcher.cs
│   │   └── ImapClientFactory.cs
│   │
│   ├── Rules/
│   │   ├── IRuleEngine.cs
│   │   ├── RuleEngine.cs
│   │   ├── IConditionEvaluator.cs
│   │   ├── ConditionEvaluator.cs
│   │   ├── TextConditionEvaluator.cs
│   │   ├── DateConditionEvaluator.cs
│   │   ├── IActionExecutor.cs
│   │   ├── ActionExecutor.cs
│   │   └── Actions/
│   │       ├── IActionHandler.cs
│   │       ├── MoveToFolderAction.cs
│   │       ├── MoveToAccountAction.cs
│   │       ├── DeleteMessageAction.cs
│   │       ├── MarkAsReadAction.cs
│   │       ├── MarkAsUnreadAction.cs
│   │       ├── AddFlagAction.cs
│   │       └── RemoveFlagAction.cs
│   │
│   ├── Storage/
│   │   ├── IProcessedMessageStore.cs
│   │   ├── ProcessedMessageStore.cs
│   │   ├── IAuditLogger.cs
│   │   ├── SqliteAuditLogger.cs
│   │   ├── IBackupService.cs
│   │   └── BackupService.cs
│   │
│   ├── Processing/
│   │   ├── IAccountProcessor.cs
│   │   ├── AccountProcessor.cs
│   │   ├── ProcessingContext.cs
│   │   ├── RateLimiter.cs
│   │   └── RetryPolicy.cs
│   │
│   ├── Models/
│   │   ├── AccountConfig.cs
│   │   ├── RuleConfig.cs
│   │   ├── RuleCondition.cs
│   │   ├── TextCondition.cs
│   │   ├── DateCondition.cs
│   │   ├── RuleAction.cs
│   │   ├── AppConfig.cs
│   │   ├── LoggingConfig.cs
│   │   ├── StorageConfig.cs
│   │   ├── RateLimitConfig.cs
│   │   ├── BackupConfig.cs
│   │   ├── TokenResponse.cs
│   │   ├── AuditEntry.cs
│   │   ├── AuditQuery.cs
│   │   ├── AuditSummary.cs
│   │   ├── RuleExecutionResult.cs
│   │   └── ActionResult.cs
│   │
│   └── Mazooka.Core.csproj
│
└── Mazooka.Tests/                      # Unit tests
    ├── Authentication/
    │   ├── PasswordAuthenticatorTests.cs
    │   └── OAuth2AuthenticatorTests.cs
    │
    ├── Rules/
    │   ├── ConditionEvaluatorTests.cs
    │   └── RuleEngineTests.cs
    │
    ├── Storage/
    │   ├── ProcessedMessageStoreTests.cs
│   └── AuditLoggerTests.cs
│
└── Mazooka.Tests.csproj
```## Project Files

### Mazooka.csproj (Console App)

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <!-- Core library reference -->
    <ProjectReference Include="..\Mazooka.Core\Mazooka.Core.csproj" />
  </ItemGroup>

  <ItemGroup>
    <!-- Dependency injection -->
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="8.0.0" />

    <!-- Configuration -->
    <PackageReference Include="Microsoft.Extensions.Configuration" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.EnvironmentVariables" Version="8.0.0" />

    <!-- Logging -->
    <PackageReference Include="Serilog" Version="3.1.1" />
    <PackageReference Include="Serilog.Sinks.Console" Version="5.0.1" />
    <PackageReference Include="Serilog.Sinks.File" Version="5.0.0" />
    <PackageReference Include="Serilog.Extensions.Logging" Version="8.0.0" />

    <!-- Command line parsing -->
    <PackageReference Include="System.CommandLine" Version="2.0.0-beta4.22272.1" />
  </ItemGroup>

  <ItemGroup>
    <None Update="appsettings.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
  </ItemGroup>

</Project>
```

### Mazooka.Core.csproj (Library)

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <!-- IMAP -->
    <PackageReference Include="MailKit" Version="4.3.0" />

    <!-- Database -->
    <PackageReference Include="Microsoft.Data.Sqlite" Version="8.0.0" />

    <!-- Logging -->
    <PackageReference Include="Microsoft.Extensions.Logging.Abstractions" Version="8.0.0" />
    <PackageReference Include="Serilog" Version="3.1.1" />

    <!-- JSON serialization -->
    <PackageReference Include="System.Text.Json" Version="8.0.0" />

    <!-- HTTP client for OAuth2 -->
    <PackageReference Include="System.Net.Http" Version="4.3.4" />
  </ItemGroup>

</Project>
```

### Mazooka.Tests.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <!-- Project reference -->
    <ProjectReference Include="..\Mazooka.Core\Mazooka.Core.csproj" />
  </ItemGroup>

  <ItemGroup>
    <!-- Testing frameworks -->
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    <PackageReference Include="xUnit" Version="2.6.4" />
    <PackageReference Include="xUnit.runner.visualstudio" Version="2.5.5" />

    <!-- Mocking -->
    <PackageReference Include="Moq" Version="4.20.70" />

    <!-- Assertions -->
    <PackageReference Include="FluentAssertions" Version="6.12.0" />
  </ItemGroup>

</Project>
```

## Namespace Organization

```csharp
// Authentication
namespace Mazooka.Core.Authentication;

// IMAP operations
namespace Mazooka.Core.Imap;

// Rule engine
namespace Mazooka.Core.Rules;
namespace Mazooka.Core.Rules.Actions;

// Storage
namespace Mazooka.Core.Storage;

// Processing
namespace Mazooka.Core.Processing;

// Models
namespace Mazooka.Core.Models;

// Console app
namespace Mazooka;
namespace Mazooka.Commands;
```

## Dependency Injection Setup

### Program.cs

```csharp
using System.CommandLine;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Serilog;
using Mazooka.Core;

namespace Mazooka;

public class Program
{
    public static async Task<int> Main(string[] args)
    {
        // Create host builder
        var host = Host.CreateDefaultBuilder(args)
            .ConfigureServices((context, services) =>
            {
                // Load configuration
                var config = ConfigManager.Load("appsettings.json");
                services.AddSingleton(config);

                // Setup logging
                var logger = LoggingSetup.ConfigureLogging(config.Logging);
                services.AddSingleton(logger);
                services.AddLogging(builder =>
                {
                    builder.ClearProviders();
                    builder.AddSerilog(logger, dispose: true);
                });

                // Register core services
                services.AddCoreServices(config);

                // Register commands
                services.AddScoped<RunCommand>();
                services.AddScoped<SetupOAuth2Command>();
                services.AddScoped<TestAuthCommand>();
                services.AddScoped<AuditCommand>();
            })
            .Build();

        // Create command-line interface
        var rootCommand = new RootCommand("Mazooka - Thunderbird-style email filtering");

        var runCommand = new Command("run", "Process all configured accounts");
        var daemonOption = new Option<bool>("--daemon", "Run in daemon mode");
        var dryRunOption = new Option<bool>("--dry-run", "Preview changes without executing");
        runCommand.AddOption(daemonOption);
        runCommand.AddOption(dryRunOption);
        runCommand.SetHandler(async (daemon, dryRun) =>
        {
            var cmd = host.Services.GetRequiredService<RunCommand>();
            await cmd.ExecuteAsync(daemon, dryRun);
        }, daemonOption, dryRunOption);

        var setupCommand = new Command("setup-oauth", "Setup OAuth2 authentication");
        var providerArg = new Argument<string>("provider", "OAuth provider (gmail, outlook)");
        var accountArg = new Argument<string>("account", "Account name");
        setupCommand.AddArgument(providerArg);
        setupCommand.AddArgument(accountArg);
        setupCommand.SetHandler(async (provider, account) =>
        {
            var cmd = host.Services.GetRequiredService<SetupOAuth2Command>();
            await cmd.ExecuteAsync(provider, account);
        }, providerArg, accountArg);

        rootCommand.AddCommand(runCommand);
        rootCommand.AddCommand(setupCommand);

        // Execute command
        return await rootCommand.InvokeAsync(args);
    }
}
```

### Service Registration

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddCoreServices(
        this IServiceCollection services,
        AppConfig config)
    {
        // Configuration
        services.AddSingleton(config.Storage);
        services.AddSingleton(config.Logging);

        // Authentication
        services.AddScoped<PasswordAuthenticator>();
        services.AddScoped<IAuthenticatorFactory, AuthenticatorFactory>();

        // IMAP
        services.AddScoped<IImapClientFactory, ImapClientFactory>();
        services.AddScoped<MessageFetcher>();

        // Rules
        services.AddScoped<IConditionEvaluator, ConditionEvaluator>();
        services.AddScoped<IActionExecutor, ActionExecutor>();
        services.AddScoped<IRuleEngine, RuleEngine>();

        // Action handlers
        services.AddScoped<MoveToFolderAction>();
        services.AddScoped<MoveToAccountAction>();
        services.AddScoped<DeleteMessageAction>();
        services.AddScoped<MarkAsReadAction>();
        services.AddScoped<MarkAsUnreadAction>();
        services.AddScoped<AddFlagAction>();
        services.AddScoped<RemoveFlagAction>();

        // Storage
        services.AddSingleton<IProcessedMessageStore>(sp =>
            new ProcessedMessageStore(
                config.Storage.UidDatabasePath,
                sp.GetRequiredService<ILogger<ProcessedMessageStore>>()));

        services.AddSingleton<IAuditLogger>(sp =>
            new SqliteAuditLogger(
                config.Logging.SqlitePath,
                sp.GetRequiredService<ILogger<SqliteAuditLogger>>()));

        services.AddSingleton<IBackupService, BackupService>();

        // Processing
        services.AddScoped<IAccountProcessor, AccountProcessor>();

        return services;
    }
}
```

## Build and Run

### Build

```bash
# Build solution
dotnet build

# Build specific project
dotnet build Mazooka.Core/Mazooka.Core.csproj

# Build for release
dotnet build -c Release
```

### Run

```bash
# Run from source
dotnet run --project Mazooka

# Run with arguments
dotnet run --project Mazooka -- run --dry-run

# Run published executable
dotnet publish -c Release -o ./publish
./publish/Mazooka run
```

### Test

```bash
# Run all tests
dotnet test

# Run with coverage
dotnet test --collect:"XPlat Code Coverage"

# Run specific test
dotnet test --filter "FullyQualifiedName~ConditionEvaluatorTests"
```

## Deployment

### Publish as Self-Contained

```bash
# Windows
dotnet publish -c Release -r win-x64 --self-contained -o ./publish/win

# Linux
dotnet publish -c Release -r linux-x64 --self-contained -o ./publish/linux

# macOS
dotnet publish -c Release -r osx-x64 --self-contained -o ./publish/osx
```

### Docker (Optional)

```dockerfile
FROM mcr.microsoft.com/dotnet/runtime:8.0 AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["Mazooka/Mazooka.csproj", "Mazooka/"]
COPY ["Mazooka.Core/Mazooka.Core.csproj", "Mazooka.Core/"]
RUN dotnet restore "Mazooka/Mazooka.csproj"
COPY . .
WORKDIR "/src/Mazooka"
RUN dotnet build "Mazooka.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "Mazooka.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Mazooka.dll"]
```

## Configuration Files

### appsettings.json (Template)

```json
{
  "logging": {
    "level": "Information",
    "filePath": "logs/app-.log",
    "sqlitePath": "logs/audit.db"
  },
  "storage": {
    "uidDatabasePath": "data/processed.db",
    "backupPath": "backups"
  },
  "accounts": [],
  "rules": []
}
```

### .gitignore

```gitignore
# Build outputs
bin/
obj/
publish/

# User-specific files
*.user
*.suo
*.userprefs
.vs/

# Application data
data/
logs/
backups/

# Secrets
appsettings.secrets.json
*.secret.json

# OS files
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
*.swp
```

### README.md

```markdown
# Mazooka

Thunderbird-style email filtering for multiple IMAP accounts.

## Features

- Multiple account support (password & OAuth2)
- Flexible rule engine with conditions and actions
- UID-based tracking (no duplicate processing)
- Comprehensive audit logging
- Dry-run mode for testing

## Quick Start

1. Configure accounts and rules in `appsettings.json`
2. Setup OAuth2 (if using Gmail):
   ```bash
   dotnet run -- setup-oauth gmail work
   ```
3. Test authentication:
   ```bash
   dotnet run -- test-auth work
   ```
4. Run processor:
   ```bash
   dotnet run -- run
   ```

## Documentation

See `docs/codegen/2025-12-10-mazooka/` for complete specifications.

## License

MIT
```

## Development Workflow

### Initial Setup

```bash
# Clone repository
git clone https://github.com/yourusername/Mazooka.git
cd Mazooka

# Restore packages
dotnet restore

# Build solution
dotnet build

# Run tests
dotnet test

# Create configuration
cp appsettings.json appsettings.local.json
# Edit appsettings.local.json with your accounts
```

### Adding a New Action Type

1. Create action handler in `Mazooka.Core/Rules/Actions/`
2. Implement `IActionHandler` interface
3. Register in `ActionExecutor` constructor
4. Add tests in `Mazooka.Tests/Rules/`
5. Update documentation

### Adding a New OAuth Provider

1. Create provider in `Mazooka.Core/Authentication/`
2. Implement `IOAuth2Provider` interface
3. Register in `AuthenticatorFactory.DetermineOAuth2Provider()`
4. Add tests
5. Update documentation

## Best Practices

1. **Dependency Injection**: Use DI for all components
2. **Async/Await**: All I/O operations must be async
3. **Cancellation Tokens**: Support graceful shutdown
4. **Logging**: Use structured logging with Serilog
5. **Testing**: Write unit tests for all business logic
6. **Error Handling**: Use try-catch with proper logging
7. **Configuration**: Support environment variables
8. **Documentation**: Keep XML comments up to date
