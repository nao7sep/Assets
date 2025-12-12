# Architecture and Design

## High-Level Architecture

IdeaDump follows a layered architecture using Blazor Server for real-time UI updates and background services for async processing.

```
┌─────────────────────────────────────────────────────────┐
│                    Browser Client                        │
│              (Blazor Server via SignalR)                │
└────────────────────┬────────────────────────────────────┘
                     │ WebSocket (SignalR)
┌────────────────────┴────────────────────────────────────┐
│                  Blazor Server Host                      │
│  ┌───────────────────────────────────────────────────┐  │
│  │              UI Components (Razor)                │  │
│  │  • Channel Management                             │  │
│  │  • Idea Submission                                │  │
│  │  • Real-time Status Updates                       │  │
│  │  • Admin Dashboard                                │  │
│  └───────────────────┬───────────────────────────────┘  │
│                      │                                   │
│  ┌───────────────────┴───────────────────────────────┐  │
│  │            Application Services                   │  │
│  │  • ChannelService                                 │  │
│  │  • IdeaService                                    │  │
│  │  • UserService                                    │  │
│  │  • RecipientService                               │  │
│  └───────────────────┬───────────────────────────────┘  │
│                      │                                   │
│  ┌───────────────────┴───────────────────────────────┐  │
│  │         Background Hosted Services                │  │
│  │  • IdeaElaborationService (AI)                    │  │
│  │  • EmailSendingService (SMTP)                     │  │
│  │  • RetryService (Failed tasks)                    │  │
│  │  • DailyLimitResetService (Cron)                  │  │
│  └───────────────────┬───────────────────────────────┘  │
│                      │                                   │
│  ┌───────────────────┴───────────────────────────────┐  │
│  │              Data Access Layer                    │  │
│  │  • EF Core DbContext                              │  │
│  │  • Repositories (if needed)                       │  │
│  └───────────────────┬───────────────────────────────┘  │
└──────────────────────┼───────────────────────────────────┘
                       │
┌──────────────────────┴───────────────────────────────────┐
│                  SQLite Database                         │
│  • Users & Identity                                      │
│  • Channels & Recipients                                 │
│  • Idea Entries                                          │
│  • System Logs                                           │
└──────────────────────┬───────────────────────────────────┘
                       │
         ┌─────────────┴──────────────┐
         │                            │
┌────────┴────────┐          ┌────────┴────────┐
│   AI Service    │          │  SMTP Server    │
│ (GitHub Models  │          │   (SendGrid,    │
│  Azure OpenAI)  │          │   Office365)    │
└─────────────────┘          └─────────────────┘
```

## Technology Stack

### Frontend
- **Blazor Server**: Real-time UI via SignalR
- **Bootstrap 5**: Responsive styling
- **Blazor Components**: Reusable UI elements

### Backend
- **.NET 10.0**: Runtime and SDK
- **ASP.NET Core**: Web framework
- **Entity Framework Core**: ORM
- **ASP.NET Core Identity**: Authentication
- **Hosted Services**: Background processing

### Data Storage
- **SQLite**: Primary database
- **Data Protection API**: Encryption

### External Services
- **AI Services**: GitHub Models, Azure OpenAI, or compatible APIs
- **SMTP**: Email delivery (SendGrid, Office 365, Gmail, etc.)

### Testing
- **xUnit**: Unit testing framework
- **Moq**: Mocking framework
- **EF Core InMemory**: Testing database

## Blazor Server Design

### Why Blazor Server?

1. **Real-time Updates**: SignalR enables live status updates as AI elaboration progresses
2. **No API Layer Needed**: Direct server-side component interaction
3. **Simplified Architecture**: Single deployment, no separate frontend/backend
4. **Small User Base**: Blazor Server's connection limits are acceptable for 10-100 users
5. **Reduced Client Load**: Heavy processing happens server-side

### Blazor Server Considerations

**Pros:**
- Instant UI updates without polling
- No JavaScript framework complexity
- Full .NET ecosystem access
- Secure by default (no client-side secrets)

**Cons:**
- Requires persistent WebSocket connection
- Higher server memory usage per user
- Latency for geographically distant users (mitigated by Azure region selection)

**Mitigation Strategies:**
- Circuit timeout: 30 minutes idle
- Reconnection UI for dropped connections
- Graceful degradation if SignalR unavailable

## Background Processing Architecture

### Hosted Services

IdeaDump uses `IHostedService` implementations for background tasks:

```csharp
public class IdeaElaborationService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcessPendingIdeasAsync();
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}
```

### Service Responsibilities

#### 1. IdeaElaborationService
- **Purpose**: Process ideas with status `New` → `Elaborating` → `Elaborated`
- **Polling Interval**: Every 5 seconds
- **Process**:
  1. Query ideas with `Status == New`
  2. Check user rate limits
  3. Determine API key (system → user → channel)
  4. Call AI service with instructions
  5. Parse and store elaboration result
  6. Update status to `Elaborated`
  7. Log all steps

#### 2. EmailSendingService
- **Purpose**: Send elaborated ideas to recipients via SMTP
- **Polling Interval**: Every 5 seconds
- **Process**:
  1. Query ideas with `Status == Elaborated`
  2. Fetch confirmed recipients for channel
  3. Build email with elaboration content
  4. Send to all recipients
  5. Update status to `Sent` on success
  6. Update status to `Failed` and schedule retry on error
  7. Log all steps

#### 3. RetryService
- **Purpose**: Retry failed ideas after exponential backoff
- **Polling Interval**: Every 30 seconds
- **Process**:
  1. Query ideas with `Status == Failed` and `NextRetryAt <= now`
  2. Check `RetryCount < MaxRetries`
  3. Reset status to `New` or `Elaborated` (depending on failure point)
  4. Increment `RetryCount`
  5. Schedule next retry: $2^{RetryCount} \times 60$ seconds
  6. If `RetryCount >= MaxRetries`, set status to `Abandoned` and notify admin

#### 4. DailyLimitResetService
- **Purpose**: Reset daily idea counters at midnight UTC
- **Schedule**: Runs daily at 00:00 UTC
- **Process**:
  1. Query all users where `LastLimitReset < today`
  2. Set `IdeasElaboratedToday = 0`
  3. Set `LastLimitReset = today`

### Concurrency & Thread Safety

- **Single Instance**: Each background service runs once per application instance
- **Database Locking**: Use EF Core transactions to prevent race conditions
- **Idempotency**: Status checks ensure ideas aren't processed twice
- **Cancellation Tokens**: Graceful shutdown on application stop

### Error Handling

```csharp
try
{
    // Process idea
}
catch (Exception ex)
{
    await LogErrorAsync(ex, ideaId);
    await UpdateIdeaStatusAsync(ideaId, IdeaEntryStatus.Failed);
    await ScheduleRetryAsync(ideaId);
}
```

## AI Integration Architecture

### API Key Hierarchy

```csharp
public string GetEffectiveApiKey(Channel channel, ApplicationUser user)
{
    // Priority: Channel → User → System
    return channel.ApiKey
        ?? user.ApiKey
        ?? _systemConfig["System.ApiKey"];
}
```

### AI Service Interface

```csharp
public interface IAiService
{
    Task<ElaborationResult> ElaborateIdeaAsync(
        string rawText,
        string instructions,
        string apiKey,
        CancellationToken ct);
}

public class ElaborationResult
{
    public string Title { get; set; }
    public string Content { get; set; }
    public string EmailSubject { get; set; }
    public string EmailBody { get; set; }
    public bool Success { get; set; }
    public string? ErrorMessage { get; set; }
}
```

### Supported AI Providers

1. **GitHub Models** (Primary)
   - Endpoint: `https://models.inference.ai.azure.com`
   - Models: `gpt-4o`, `gpt-4o-mini`
   - Authentication: Bearer token

2. **Azure OpenAI** (Fallback)
   - Endpoint: `https://<resource>.openai.azure.com`
   - Models: Configurable deployment names
   - Authentication: API key

3. **OpenAI** (Alternative)
   - Endpoint: `https://api.openai.com/v1`
   - Models: `gpt-4o`, `gpt-4-turbo`
   - Authentication: API key

### Rate Limiting Logic

```csharp
public async Task<bool> CanUserSubmitIdeaAsync(Guid userId)
{
    var user = await _context.Users.FindAsync(userId);

    // Reset counter if new day
    if (user.LastLimitReset < DateTime.UtcNow.Date)
    {
        user.IdeasElaboratedToday = 0;
        user.LastLimitReset = DateTime.UtcNow.Date;
        await _context.SaveChangesAsync();
    }

    return user.IdeasElaboratedToday < user.DailyIdeaLimit;
}
```

## Email Architecture

### SMTP Configuration Hierarchy

1. **Channel-level**: Custom SMTP per channel (isolated sender identities)
2. **System-level**: Default SMTP for channels without custom config

### Email Templates

Emails are generated from AI elaboration with:
- **Subject**: AI-generated concise title
- **Body**: Formatted elaboration content (HTML)
- **Footer**: Unsubscribe link, system branding

```html
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: Arial, sans-serif; }
        .content { max-width: 600px; margin: 0 auto; }
        .footer { font-size: 12px; color: #666; margin-top: 30px; }
    </style>
</head>
<body>
    <div class="content">
        <h1>{{EmailSubject}}</h1>
        <div>{{ElaboratedContent}}</div>
        <div class="footer">
            <p>You are receiving this because you subscribed to "{{ChannelName}}".</p>
            <p><a href="{{UnsubscribeUrl}}">Unsubscribe</a></p>
        </div>
    </div>
</body>
</html>
```

### Retry Strategy

- **Exponential Backoff**: $2^{attempt} \times 60$ seconds
- **Max Retries**: 5 attempts
- **Transient Errors**: Network timeouts, SMTP 4xx errors
- **Permanent Errors**: Invalid recipient, auth failure → abandon immediately

## Security Architecture

### Authentication
- **ASP.NET Core Identity**: User registration, login, password management
- **Email Confirmation**: Required before account activation
- **Password Policy**: Strong passwords enforced

### Authorization
- **User-level**: Users can only access their own channels and ideas
- **Admin-level**: Special role for system administration
- **Recipient-level**: No login, but explicit consent required

### Data Protection
- **Encrypted at Rest**: API keys, SMTP passwords via Data Protection API
- **HTTPS Only**: All traffic encrypted in transit
- **SQL Injection**: Prevented by EF Core parameterized queries
- **XSS**: Prevented by Blazor's automatic encoding

### API Key Security
- **Never logged**: API keys redacted in logs
- **Never exposed to client**: Server-side only
- **Encrypted storage**: Database encryption for keys

## Logging Architecture

### Log Levels
- **Debug**: Detailed flow tracing (disabled in production)
- **Information**: Normal operations (idea submitted, email sent)
- **Warning**: Non-critical issues (retry scheduled)
- **Error**: Failures requiring attention (API timeout, SMTP error)
- **Critical**: System-level failures (database unavailable)

### Log Destinations

1. **File Logs**: Rolling file per day
   - Path: `logs/app-{Date}.log`
   - Retention: 30 days

2. **Database Logs**: `SystemLog` table
   - Queryable via admin dashboard
   - Retention: 90 days

3. **Console Logs**: Development only

### Admin Notification Logic

```csharp
public async Task SendAdminNotificationAsync(string message, LogLevel level)
{
    if (level < LogLevel.Error) return;

    var lastNotified = await GetLastNotificationTimeAsync();
    var throttleMinutes = int.Parse(_config["System.NotificationThrottleMinutes"]);

    if (DateTime.UtcNow - lastNotified < TimeSpan.FromMinutes(throttleMinutes))
    {
        // Throttle: Skip notification
        return;
    }

    await SendEmailAsync(_config["System.AdminNotificationEmail"], "Critical Error", message);
    await UpdateLastNotificationTimeAsync();
}
```

## Deployment Architecture

### Azure App Service
- **Hosting Plan**: Basic B1 or Standard S1 (supports WebSockets for SignalR)
- **Region**: Choose closest to primary users
- **Scaling**: Single instance (sufficient for 10-100 users)
- **Always On**: Enabled to keep background services running

### Environment Variables
```bash
ASPNETCORE_ENVIRONMENT=Production
ConnectionStrings__DefaultConnection=Data Source=/home/site/data/ideadump.db
System__ApiKey=<encrypted>
System__AdminNotificationEmail=admin@example.com
```

### File Storage
- **SQLite Database**: Persistent storage in `/home/site/data`
- **Log Files**: Persistent storage in `/home/site/logs`
- **Backup Strategy**: Daily automated Azure backup or manual export

## Performance Considerations

### Expected Load
- **Users**: 10-100 active users
- **Ideas per day**: ~100-1000 submissions
- **Background processing**: Sequential, low CPU usage
- **Database size**: ~100 MB/year (text-heavy)

### Optimization Strategies
- **Caching**: User settings cached in memory
- **Connection Pooling**: EF Core default connection pool
- **Lazy Loading**: Disabled (explicit includes only)
- **Indexes**: See data model specification

### Monitoring
- **Application Insights**: Optional Azure integration
- **Health Checks**: Endpoint for Azure monitoring
- **Custom Metrics**: Ideas processed, emails sent, retry counts

## Scalability Path

If user base exceeds 100 users:

1. **Horizontal Scaling**: Multiple App Service instances with Redis for SignalR backplane
2. **Database Migration**: SQLite → PostgreSQL on Azure
3. **Queue-based Processing**: Replace hosted services with Azure Service Bus
4. **CDN**: Static assets via Azure CDN
5. **Rate Limiting**: Token bucket algorithm per user

---

**Next Document**: [03-authentication-and-authorization.md](03-authentication-and-authorization.md)
