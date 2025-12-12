# Project Structure and Dependencies

## Solution Structure

```
IdeaDump.sln
├── src/
│   ├── IdeaDump/                    # Blazor Server web application
│   │   ├── Components/
│   │   │   ├── Layout/
│   │   │   │   ├── MainLayout.razor
│   │   │   │   ├── NavMenu.razor
│   │   │   │   └── NavMenu.razor.css
│   │   │   ├── Pages/
│   │   │   │   ├── Dashboard.razor
│   │   │   │   ├── Channels/
│   │   │   │   │   ├── ChannelList.razor
│   │   │   │   │   ├── ChannelDetail.razor
│   │   │   │   │   └── CreateChannel.razor
│   │   │   │   ├── Admin/
│   │   │   │   │   ├── AdminDashboard.razor
│   │   │   │   │   ├── AdminUsersPanel.razor
│   │   │   │   │   ├── AdminLogsPanel.razor
│   │   │   │   │   ├── AdminStatsPanel.razor
│   │   │   │   │   └── AdminConfigPanel.razor
│   │   │   │   ├── Account/
│   │   │   │   │   ├── Login.razor
│   │   │   │   │   ├── Register.razor
│   │   │   │   │   ├── ConfirmEmail.razor
│   │   │   │   │   └── Settings.razor
│   │   │   │   └── Public/
│   │   │   │       ├── ConfirmRecipient.razor
│   │   │   │       └── Unsubscribe.razor
│   │   │   ├── Shared/
│   │   │   │   ├── IdeaCard.razor
│   │   │   │   ├── ChannelCard.razor
│   │   │   │   ├── RecipientManagementComponent.razor
│   │   │   │   └── StatusBadge.razor
│   │   │   ├── App.razor
│   │   │   ├── Routes.razor
│   │   │   └── _Imports.razor
│   │   ├── Hubs/
│   │   │   └── IdeaHub.cs
│   │   ├── wwwroot/
│   │   │   ├── app.css
│   │   │   ├── bootstrap/
│   │   │   └── js/
│   │   ├── appsettings.json
│   │   ├── appsettings.Development.json
│   │   ├── Program.cs
│   │   └── IdeaDump.csproj
│   │
│   └── IdeaDumpCore/               # Business logic library
│       ├── Data/
│       │   ├── ApplicationDbContext.cs
│       │   ├── Entities/
│       │   │   ├── ApplicationUser.cs
│       │   │   ├── Channel.cs
│       │   │   ├── ChannelRecipient.cs
│       │   │   ├── IdeaEntry.cs
│       │   │   ├── IdeaProcessingLog.cs
│       │   │   ├── SystemLog.cs
│       │   │   └── SystemConfiguration.cs
│       │   └── Migrations/
│       ├── Services/
│       │   ├── AI/
│       │   │   ├── IAiService.cs
│       │   │   ├── GitHubModelsAiService.cs
│       │   │   ├── AzureOpenAIService.cs
│       │   │   ├── OpenAIService.cs
│       │   │   ├── ApiKeyResolver.cs
│       │   │   └── ElaborationParser.cs
│       │   ├── Email/
│       │   │   ├── IEmailService.cs
│       │   │   ├── SmtpEmailService.cs
│       │   │   ├── EmailTemplateService.cs
│       │   │   └── EmailResult.cs
│       │   ├── Background/
│       │   │   ├── IdeaElaborationService.cs
│       │   │   ├── EmailSendingService.cs
│       │   │   ├── RetryService.cs
│       │   │   └── DailyLimitResetService.cs
│       │   ├── Admin/
│       │   │   ├── AdminService.cs
│       │   │   ├── AdminNotificationService.cs
│       │   │   └── SystemStatistics.cs
│       │   ├── ChannelService.cs
│       │   ├── IdeaService.cs
│       │   ├── RecipientService.cs
│       │   ├── UserService.cs
│       │   ├── RateLimitService.cs
│       │   ├── EncryptionService.cs
│       │   └── SystemLogService.cs
│       ├── Models/
│       │   ├── DTOs/
│       │   │   ├── ChannelSummary.cs
│       │   │   ├── IdeaSummary.cs
│       │   │   ├── RecipientSummary.cs
│       │   │   └── UserSummary.cs
│       │   ├── Requests/
│       │   │   ├── CreateChannelRequest.cs
│       │   │   ├── UpdateChannelRequest.cs
│       │   │   ├── AddRecipientRequest.cs
│       │   │   └── CreateIdeaRequest.cs
│       │   └── Configuration/
│       │       ├── SmtpConfiguration.cs
│       │       └── AiConfiguration.cs
│       └── IdeaDumpCore.csproj
│
└── tests/
    └── IdeaDumpTests/
        ├── Services/
        │   ├── AiServiceTests.cs
        │   ├── EmailServiceTests.cs
        │   ├── ChannelServiceTests.cs
        │   └── RateLimitServiceTests.cs
        ├── Mocks/
        │   ├── MockAiService.cs
        │   └── MockEmailService.cs
        └── IdeaDumpTests.csproj
```

## Dependencies

### IdeaDump.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <BlazorDisableThrowNavigationException>true</BlazorDisableThrowNavigationException>
    <RootNamespace>IdeaDump</RootNamespace>
    <Version>0.1.0</Version>
  </PropertyGroup>

  <ItemGroup>
    <!-- Blazor and ASP.NET Core -->
    <PackageReference Include="Microsoft.AspNetCore.Components.Web" Version="10.0.0" />
    <PackageReference Include="Microsoft.AspNetCore.SignalR.Client" Version="10.0.0" />

    <!-- Identity and Authentication -->
    <PackageReference Include="Microsoft.AspNetCore.Identity.EntityFrameworkCore" Version="10.0.0" />

    <!-- Entity Framework Core -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="10.0.0">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>

    <!-- Data Protection for encryption -->
    <PackageReference Include="Microsoft.AspNetCore.DataProtection" Version="10.0.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\IdeaDumpCore\IdeaDumpCore.csproj" />
  </ItemGroup>

</Project>
```

### IdeaDumpCore.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <RootNamespace>IdeaDump.Core</RootNamespace>
    <Version>0.1.0</Version>
  </PropertyGroup>

  <ItemGroup>
    <!-- Entity Framework Core -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="10.0.0">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>

    <!-- ASP.NET Core Identity -->
    <PackageReference Include="Microsoft.AspNetCore.Identity.EntityFrameworkCore" Version="10.0.0" />

    <!-- HTTP Client for AI API calls -->
    <PackageReference Include="Microsoft.Extensions.Http" Version="10.0.0" />

    <!-- JSON serialization -->
    <PackageReference Include="System.Text.Json" Version="10.0.0" />

    <!-- Markdown processing -->
    <PackageReference Include="Markdig" Version="0.37.0" />

    <!-- Hosted services -->
    <PackageReference Include="Microsoft.Extensions.Hosting.Abstractions" Version="10.0.0" />

    <!-- Data Protection -->
    <PackageReference Include="Microsoft.AspNetCore.DataProtection.Abstractions" Version="10.0.0" />

    <!-- Logging -->
    <PackageReference Include="Microsoft.Extensions.Logging.Abstractions" Version="10.0.0" />
  </ItemGroup>

</Project>
```

### IdeaDumpTests.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
    <RootNamespace>IdeaDump.Tests</RootNamespace>
    <Version>0.1.0</Version>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="coverlet.collector" Version="6.0.4" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="18.0.1" />
    <PackageReference Include="xunit" Version="2.9.3" />
    <PackageReference Include="xunit.runner.visualstudio" Version="3.1.5">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>

    <!-- Mocking -->
    <PackageReference Include="Moq" Version="4.20.72" />

    <!-- In-memory database for testing -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="10.0.0" />
  </ItemGroup>

  <ItemGroup>
    <Using Include="Xunit" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\IdeaDumpCore\IdeaDumpCore.csproj" />
    <ProjectReference Include="..\..\src\IdeaDump\IdeaDump.csproj" />
  </ItemGroup>

</Project>
```

## Program.cs Configuration

```csharp
using IdeaDump.Core.Data;
using IdeaDump.Core.Services;
using IdeaDump.Core.Services.AI;
using IdeaDump.Core.Services.Email;
using IdeaDump.Core.Services.Background;
using IdeaDump.Core.Services.Admin;
using IdeaDump.Hubs;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Database
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));

// Identity
builder.Services.AddDefaultIdentity<ApplicationUser>(options =>
{
    options.SignIn.RequireConfirmedEmail = true;
    options.Password.RequireDigit = true;
    options.Password.RequiredLength = 8;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireLowercase = true;
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
})
.AddEntityFrameworkStores<ApplicationDbContext>()
.AddDefaultTokenProviders();

// Authorization
builder.Services.AddAuthorizationCore(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireAssertion(context =>
        {
            var dbContext = context.User.FindFirst(System.Security.Claims.ClaimTypes.NameIdentifier);
            // Implementation in middleware
            return true; // Placeholder
        }));
});

// Data Protection
builder.Services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo("./keys"))
    .SetApplicationName("IdeaDump");

// Application Services
builder.Services.AddScoped<ChannelService>();
builder.Services.AddScoped<IdeaService>();
builder.Services.AddScoped<RecipientService>();
builder.Services.AddScoped<UserService>();
builder.Services.AddScoped<RateLimitService>();
builder.Services.AddScoped<EncryptionService>();
builder.Services.AddScoped<SystemLogService>();
builder.Services.AddScoped<AdminService>();
builder.Services.AddScoped<EmailTemplateService>();
builder.Services.AddScoped<ApiKeyResolver>();

// AI Services
builder.Services.AddHttpClient<IAiService, GitHubModelsAiService>();

// Email Services
builder.Services.AddScoped<IEmailService, SmtpEmailService>();

// Background Services
builder.Services.AddHostedService<IdeaElaborationService>();
builder.Services.AddHostedService<EmailSendingService>();
builder.Services.AddHostedService<RetryService>();
builder.Services.AddHostedService<DailyLimitResetService>();

// Blazor
builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents();

// SignalR
builder.Services.AddSignalR();

// Cookie configuration
builder.Services.ConfigureApplicationCookie(options =>
{
    options.Cookie.HttpOnly = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.ExpireTimeSpan = TimeSpan.FromDays(14);
    options.SlidingExpiration = true;
    options.LoginPath = "/login";
    options.LogoutPath = "/logout";
});

var app = builder.Build();

// Middleware
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseAntiforgery();

app.UseAuthentication();
app.UseAuthorization();

// Blazor
app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode();

// SignalR Hub
app.MapHub<IdeaHub>("/ideahub");

// Database migration
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    db.Database.Migrate();
}

app.Run();
```

## Configuration Files

### appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=ideadump.db"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "Ai": {
    "Provider": "GitHub",
    "Endpoint": "https://models.inference.ai.azure.com",
    "Model": "gpt-4o",
    "MaxTokens": 4000,
    "Temperature": 0.7
  },
  "App": {
    "BaseUrl": "https://ideadump.example.com"
  }
}
```

### appsettings.Development.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Information"
    }
  },
  "App": {
    "BaseUrl": "https://localhost:7001"
  }
}
```

## Database Migrations

### Initial Migration

```bash
cd src/IdeaDumpCore
dotnet ef migrations add InitialCreate --startup-project ../IdeaDump

cd ../IdeaDump
dotnet ef database update
```

## Build and Run

```bash
# Restore dependencies
dotnet restore

# Build solution
dotnet build

# Run application
cd src/IdeaDump
dotnet run

# Run tests
cd tests/IdeaDumpTests
dotnet test
```

## Deployment

### Azure App Service

1. **Create App Service**:
   ```bash
   az webapp create --name ideadump --resource-group rg-ideadump --plan asp-ideadump --runtime "DOTNET:10.0"
   ```

2. **Configure Environment Variables**:
   ```bash
   az webapp config appsettings set --name ideadump --settings \
     "ConnectionStrings__DefaultConnection=Data Source=/home/site/data/ideadump.db" \
     "System__ApiKey=<encrypted-api-key>" \
     "System__AdminNotificationEmail=admin@example.com"
   ```

3. **Deploy**:
   ```bash
   dotnet publish -c Release
   az webapp deployment source config-zip --src ./src/IdeaDump/bin/Release/net10.0/publish.zip
   ```

## Development Workflow

1. **Create feature branch**
2. **Implement feature**
3. **Write tests**
4. **Run tests**: `dotnet test`
5. **Check for errors**: `dotnet build`
6. **Commit and push**
7. **Create pull request**
8. **Merge to main**
9. **Deploy to Azure**

---

**End of Documentation Set**

All specification documents have been created in `Assets/docs/codegen/2025-12-12-ideadump/`. These documents provide comprehensive guidance for AI-assisted implementation of the IdeaDump application.
