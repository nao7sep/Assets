# Admin Features Specification

## Admin Role

Admins have elevated privileges to:
- View all users and their usage
- Adjust rate limits per user
- View system-wide logs
- Monitor system health
- View statistics and metrics
- Configure system-wide settings

## Admin Dashboard Components

### Users Panel

**Purpose**: Manage users and their rate limits

```razor
<div class="admin-users-panel">
    <h2>User Management</h2>

    <div class="search-bar">
        <input type="text" @bind="searchTerm" @bind:event="oninput" placeholder="Search users..." />
    </div>

    <table class="table table-striped">
        <thead>
            <tr>
                <th>Email</th>
                <th>Display Name</th>
                <th>Channels</th>
                <th>Daily Limit</th>
                <th>Used Today</th>
                <th>Joined</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var user in filteredUsers)
            {
                <tr>
                    <td>@user.Email</td>
                    <td>@user.DisplayName</td>
                    <td>@user.ChannelCount</td>
                    <td>
                        <input type="number"
                               value="@user.DailyIdeaLimit"
                               @onchange="(e) => UpdateUserLimit(user.Id, int.Parse(e.Value.ToString()))"
                               style="width: 80px;" />
                    </td>
                    <td>@user.IdeasElaboratedToday / @user.DailyIdeaLimit</td>
                    <td>@user.CreatedAt.ToShortDateString()</td>
                    <td>
                        <button class="btn btn-sm btn-primary" @onclick="() => ViewUserDetails(user.Id)">
                            View
                        </button>
                        <button class="btn btn-sm btn-danger" @onclick="() => DeleteUser(user.Id)">
                            Delete
                        </button>
                    </td>
                </tr>
            }
        </tbody>
    </table>

    <div class="pagination">
        <button @onclick="PreviousPage" disabled="@(currentPage == 1)">Previous</button>
        <span>Page @currentPage of @totalPages</span>
        <button @onclick="NextPage" disabled="@(currentPage == totalPages)">Next</button>
    </div>
</div>

@code {
    private List<UserSummary> users = new();
    private List<UserSummary> filteredUsers = new();
    private string searchTerm = "";
    private int currentPage = 1;
    private int pageSize = 20;
    private int totalPages = 1;

    protected override async Task OnInitializedAsync()
    {
        await LoadUsersAsync();
    }

    private async Task LoadUsersAsync()
    {
        users = await AdminService.GetAllUsersAsync();
        FilterUsers();
    }

    private void FilterUsers()
    {
        filteredUsers = users
            .Where(u => string.IsNullOrEmpty(searchTerm)
                     || u.Email.Contains(searchTerm, StringComparison.OrdinalIgnoreCase))
            .Skip((currentPage - 1) * pageSize)
            .Take(pageSize)
            .ToList();

        totalPages = (int)Math.Ceiling(users.Count / (double)pageSize);
    }

    private async Task UpdateUserLimit(Guid userId, int newLimit)
    {
        await AdminService.SetUserDailyLimitAsync(userId, newLimit);
        await LoadUsersAsync();
    }
}
```

### System Logs Panel

**Purpose**: View and filter system logs

```razor
<div class="admin-logs-panel">
    <h2>System Logs</h2>

    <div class="log-filters">
        <select @bind="filterLevel">
            <option value="">All Levels</option>
            <option value="Debug">Debug</option>
            <option value="Information">Information</option>
            <option value="Warning">Warning</option>
            <option value="Error">Error</option>
            <option value="Critical">Critical</option>
        </select>

        <select @bind="filterCategory">
            <option value="">All Categories</option>
            <option value="AI">AI</option>
            <option value="Email">Email</option>
            <option value="Authentication">Authentication</option>
            <option value="Channel">Channel</option>
            <option value="Recipient">Recipient</option>
            <option value="System">System</option>
        </select>

        <input type="date" @bind="filterDate" />

        <button class="btn btn-primary" @onclick="ApplyFilters">Apply Filters</button>
        <button class="btn btn-secondary" @onclick="ClearFilters">Clear</button>
        <button class="btn btn-success" @onclick="ExportLogs">Export CSV</button>
    </div>

    <div class="log-table-container">
        <table class="table table-sm">
            <thead>
                <tr>
                    <th>Timestamp</th>
                    <th>Level</th>
                    <th>Category</th>
                    <th>Message</th>
                    <th>User</th>
                    <th>Details</th>
                </tr>
            </thead>
            <tbody>
                @foreach (var log in logs)
                {
                    <tr class="log-level-@log.Level.ToString().ToLower()">
                        <td>@log.Timestamp.ToLocalTime().ToString("g")</td>
                        <td>
                            <span class="badge badge-@GetLevelBadgeClass(log.Level)">
                                @log.Level
                            </span>
                        </td>
                        <td>@log.Category</td>
                        <td>@log.Message</td>
                        <td>@(log.UserId.HasValue ? GetUserEmail(log.UserId.Value) : "-")</td>
                        <td>
                            @if (!string.IsNullOrEmpty(log.ExceptionDetails))
                            {
                                <button class="btn btn-sm btn-link" @onclick="() => ShowLogDetails(log)">
                                    View Exception
                                </button>
                            }
                        </td>
                    </tr>
                }
            </tbody>
        </table>
    </div>

    <div class="pagination">
        <button @onclick="LoadMoreLogs">Load More</button>
    </div>
</div>

@code {
    private List<SystemLog> logs = new();
    private string filterLevel = "";
    private string filterCategory = "";
    private DateTime? filterDate;
    private int pageSize = 50;
    private int currentOffset = 0;

    protected override async Task OnInitializedAsync()
    {
        await LoadLogsAsync();
    }

    private async Task LoadLogsAsync()
    {
        var filter = new LogFilter
        {
            Level = string.IsNullOrEmpty(filterLevel) ? null : Enum.Parse<LogLevel>(filterLevel),
            Category = string.IsNullOrEmpty(filterCategory) ? null : filterCategory,
            Date = filterDate,
            Offset = currentOffset,
            Limit = pageSize
        };

        var newLogs = await AdminService.GetSystemLogsAsync(filter);
        logs.AddRange(newLogs);
    }

    private async Task ApplyFilters()
    {
        currentOffset = 0;
        logs.Clear();
        await LoadLogsAsync();
    }

    private async Task LoadMoreLogs()
    {
        currentOffset += pageSize;
        await LoadLogsAsync();
    }
}
```

### Statistics Panel

**Purpose**: Display system-wide metrics

```razor
<div class="admin-stats-panel">
    <h2>System Statistics</h2>

    <div class="stats-grid">
        <div class="stat-card">
            <h3>@stats.TotalUsers</h3>
            <p>Total Users</p>
            <small>+@stats.NewUsersToday today</small>
        </div>

        <div class="stat-card">
            <h3>@stats.TotalChannels</h3>
            <p>Total Channels</p>
            <small>Avg @stats.AvgChannelsPerUser.ToString("F1") per user</small>
        </div>

        <div class="stat-card">
            <h3>@stats.TotalIdeas</h3>
            <p>Total Ideas</p>
            <small>@stats.IdeasToday today</small>
        </div>

        <div class="stat-card">
            <h3>@stats.ElaboratedToday</h3>
            <p>Elaborated Today</p>
            <small>@stats.AvgElaborationTimeSeconds.ToString("F1")s avg time</small>
        </div>

        <div class="stat-card">
            <h3>@stats.EmailsSentToday</h3>
            <p>Emails Sent Today</p>
            <small>@stats.EmailFailureRate.ToString("P1") failure rate</small>
        </div>

        <div class="stat-card">
            <h3>@stats.TotalRecipients</h3>
            <p>Total Recipients</p>
            <small>@stats.ConfirmedRecipientRate.ToString("P1") confirmed</small>
        </div>
    </div>

    <div class="charts">
        <div class="chart-container">
            <h3>Ideas per Day (Last 30 Days)</h3>
            <canvas id="ideasPerDayChart"></canvas>
        </div>

        <div class="chart-container">
            <h3>Status Distribution</h3>
            <canvas id="statusDistributionChart"></canvas>
        </div>

        <div class="chart-container">
            <h3>AI Token Usage</h3>
            <canvas id="tokenUsageChart"></canvas>
        </div>
    </div>

    <div class="health-status">
        <h3>System Health</h3>
        <div class="health-indicators">
            <div class="health-item @(stats.AiServiceHealthy ? "healthy" : "unhealthy")">
                <strong>AI Service:</strong> @(stats.AiServiceHealthy ? "Healthy" : "Down")
            </div>
            <div class="health-item @(stats.SmtpServiceHealthy ? "healthy" : "unhealthy")">
                <strong>SMTP Service:</strong> @(stats.SmtpServiceHealthy ? "Healthy" : "Down")
            </div>
            <div class="health-item @(stats.DatabaseHealthy ? "healthy" : "unhealthy")">
                <strong>Database:</strong> @(stats.DatabaseHealthy ? "Healthy" : "Issues Detected")
            </div>
        </div>
    </div>
</div>

@code {
    private SystemStatistics stats = new();

    protected override async Task OnInitializedAsync()
    {
        stats = await AdminService.GetSystemStatisticsAsync();
        await RenderChartsAsync();
    }

    private async Task RenderChartsAsync()
    {
        // Use Chart.js or similar library via JS interop
        await JS.InvokeVoidAsync("renderCharts", stats);
    }
}
```

### Configuration Panel

**Purpose**: Manage system-wide settings

```razor
<div class="admin-config-panel">
    <h2>System Configuration</h2>

    <div class="config-section">
        <h3>AI Configuration</h3>
        <EditForm Model="@aiConfig" OnValidSubmit="SaveAiConfig">
            <div class="form-group">
                <label>System API Key</label>
                <InputText @bind-Value="aiConfig.ApiKey" type="password" />
                <small>Used when users don't provide their own key</small>
            </div>

            <div class="form-group">
                <label>Default AI Instructions</label>
                <InputTextArea @bind-Value="aiConfig.DefaultInstructions" rows="10" />
            </div>

            <div class="form-group">
                <label>AI Provider</label>
                <InputSelect @bind-Value="aiConfig.Provider">
                    <option value="GitHub">GitHub Models</option>
                    <option value="AzureOpenAI">Azure OpenAI</option>
                    <option value="OpenAI">OpenAI</option>
                </InputSelect>
            </div>

            <div class="form-group">
                <label>Default Daily Limit</label>
                <InputNumber @bind-Value="aiConfig.DefaultDailyLimit" />
                <small>Applied to new users</small>
            </div>

            <button type="submit" class="btn btn-primary">Save AI Config</button>
        </EditForm>
    </div>

    <div class="config-section">
        <h3>Email Configuration</h3>
        <EditForm Model="@emailConfig" OnValidSubmit="SaveEmailConfig">
            <div class="form-group">
                <label>System SMTP Server</label>
                <InputText @bind-Value="emailConfig.SmtpServer" />
            </div>

            <div class="form-group">
                <label>SMTP Port</label>
                <InputNumber @bind-Value="emailConfig.SmtpPort" />
            </div>

            <div class="form-group">
                <label>SMTP Username</label>
                <InputText @bind-Value="emailConfig.SmtpUsername" />
            </div>

            <div class="form-group">
                <label>SMTP Password</label>
                <InputText @bind-Value="emailConfig.SmtpPassword" type="password" />
            </div>

            <div class="form-group">
                <label>From Email</label>
                <InputText @bind-Value="emailConfig.FromEmail" type="email" />
            </div>

            <div class="form-group">
                <label>Admin Notification Email</label>
                <InputText @bind-Value="emailConfig.AdminNotificationEmail" type="email" />
                <small>Receives critical error notifications</small>
            </div>

            <div class="form-group">
                <label>Notification Throttle (minutes)</label>
                <InputNumber @bind-Value="emailConfig.NotificationThrottleMinutes" />
                <small>Minimum time between admin notifications</small>
            </div>

            <button type="submit" class="btn btn-primary">Save Email Config</button>
            <button type="button" class="btn btn-secondary" @onclick="TestSmtp">Test Connection</button>
        </EditForm>
    </div>

    <div class="config-section">
        <h3>Application Settings</h3>
        <EditForm Model="@appConfig" OnValidSubmit="SaveAppConfig">
            <div class="form-group">
                <label>Application Base URL</label>
                <InputText @bind-Value="appConfig.BaseUrl" />
                <small>Used in confirmation and unsubscribe links</small>
            </div>

            <div class="form-group">
                <label>Max Retry Attempts</label>
                <InputNumber @bind-Value="appConfig.MaxRetries" />
            </div>

            <div class="form-group">
                <label>Enable Registration</label>
                <InputCheckbox @bind-Value="appConfig.AllowRegistration" />
                <small>Allow new user registrations</small>
            </div>

            <button type="submit" class="btn btn-primary">Save App Config</button>
        </EditForm>
    </div>
</div>

@code {
    private AiConfigModel aiConfig = new();
    private EmailConfigModel emailConfig = new();
    private AppConfigModel appConfig = new();

    protected override async Task OnInitializedAsync()
    {
        aiConfig = await AdminService.GetAiConfigAsync();
        emailConfig = await AdminService.GetEmailConfigAsync();
        appConfig = await AdminService.GetAppConfigAsync();
    }

    private async Task SaveAiConfig()
    {
        await AdminService.SaveAiConfigAsync(aiConfig);
        await ShowToast("AI configuration saved");
    }

    private async Task SaveEmailConfig()
    {
        await AdminService.SaveEmailConfigAsync(emailConfig);
        await ShowToast("Email configuration saved");
    }

    private async Task TestSmtp()
    {
        var result = await AdminService.TestSmtpConnectionAsync(emailConfig);
        if (result.Success)
        {
            await ShowToast("SMTP connection successful!", "success");
        }
        else
        {
            await ShowToast($"SMTP connection failed: {result.ErrorMessage}", "error");
        }
    }
}
```

## Admin Service

```csharp
public class AdminService
{
    private readonly ApplicationDbContext _context;
    private readonly IEncryptionService _encryption;
    private readonly ISystemLogService _systemLog;

    public async Task<List<UserSummary>> GetAllUsersAsync()
    {
        return await _context.Users
            .Select(u => new UserSummary
            {
                Id = u.Id,
                Email = u.Email,
                DisplayName = u.DisplayName,
                DailyIdeaLimit = u.DailyIdeaLimit,
                IdeasElaboratedToday = u.IdeasElaboratedToday,
                ChannelCount = u.Channels.Count,
                CreatedAt = u.CreatedAt,
                LastLoginAt = u.LastLoginAt
            })
            .ToListAsync();
    }

    public async Task SetUserDailyLimitAsync(Guid userId, int newLimit)
    {
        var user = await _context.Users.FindAsync(userId);
        if (user == null)
            throw new InvalidOperationException("User not found");

        user.DailyIdeaLimit = newLimit;
        await _context.SaveChangesAsync();

        await _systemLog.LogAsync(LogLevel.Information, "Admin",
            $"User {user.Email} daily limit changed to {newLimit}");
    }

    public async Task<List<SystemLog>> GetSystemLogsAsync(LogFilter filter)
    {
        var query = _context.SystemLogs.AsQueryable();

        if (filter.Level.HasValue)
            query = query.Where(l => l.Level == filter.Level.Value);

        if (!string.IsNullOrEmpty(filter.Category))
            query = query.Where(l => l.Category == filter.Category);

        if (filter.Date.HasValue)
            query = query.Where(l => l.Timestamp.Date == filter.Date.Value.Date);

        return await query
            .OrderByDescending(l => l.Timestamp)
            .Skip(filter.Offset)
            .Take(filter.Limit)
            .ToListAsync();
    }

    public async Task<SystemStatistics> GetSystemStatisticsAsync()
    {
        var now = DateTime.UtcNow;
        var today = now.Date;

        return new SystemStatistics
        {
            TotalUsers = await _context.Users.CountAsync(),
            NewUsersToday = await _context.Users.CountAsync(u => u.CreatedAt >= today),
            TotalChannels = await _context.Channels.CountAsync(),
            AvgChannelsPerUser = await _context.Users
                .Select(u => u.Channels.Count)
                .AverageAsync(),
            TotalIdeas = await _context.IdeaEntries.CountAsync(),
            IdeasToday = await _context.IdeaEntries.CountAsync(i => i.CreatedAt >= today),
            ElaboratedToday = await _context.IdeaEntries
                .CountAsync(i => i.ElaboratedAt >= today),
            AvgElaborationTimeSeconds = await _context.IdeaEntries
                .Where(i => i.ElaboratedAt.HasValue)
                .Select(i => (i.ElaboratedAt.Value - i.CreatedAt).TotalSeconds)
                .AverageAsync(),
            EmailsSentToday = await _context.IdeaEntries
                .CountAsync(i => i.EmailSentAt >= today),
            TotalRecipients = await _context.ChannelRecipients.CountAsync(),
            ConfirmedRecipientRate = await _context.ChannelRecipients
                .CountAsync(r => r.IsConfirmed) /
                (double)await _context.ChannelRecipients.CountAsync(),
            AiServiceHealthy = await CheckAiServiceHealthAsync(),
            SmtpServiceHealthy = await CheckSmtpServiceHealthAsync(),
            DatabaseHealthy = true // Always true if query succeeds
        };
    }

    private async Task<bool> CheckAiServiceHealthAsync()
    {
        try
        {
            // Try a lightweight API call
            var recentFailures = await _context.SystemLogs
                .Where(l => l.Category == "AI"
                         && l.Level >= LogLevel.Error
                         && l.Timestamp >= DateTime.UtcNow.AddMinutes(-10))
                .CountAsync();

            return recentFailures < 5; // Healthy if fewer than 5 errors in 10 minutes
        }
        catch
        {
            return false;
        }
    }
}
```

## Admin Notification System

```csharp
public class AdminNotificationService
{
    private static DateTime _lastNotificationTime = DateTime.MinValue;
    private readonly IConfiguration _config;
    private readonly IEmailService _emailService;

    public async Task SendCriticalAlertAsync(string subject, string message)
    {
        var throttleMinutes = int.Parse(_config["System.NotificationThrottleMinutes"] ?? "30");

        if (DateTime.UtcNow - _lastNotificationTime < TimeSpan.FromMinutes(throttleMinutes))
        {
            // Throttled
            return;
        }

        var adminEmail = _config["System.AdminNotificationEmail"];
        if (string.IsNullOrEmpty(adminEmail))
            return;

        var emailBody = $@"
<h2>IdeaDump Critical Alert</h2>
<p><strong>{subject}</strong></p>
<p>{message}</p>
<p><small>Time: {DateTime.UtcNow:u}</small></p>
";

        await _emailService.SendEmailAsync(
            adminEmail,
            $"[IdeaDump] {subject}",
            emailBody,
            GetSystemSmtpConfig());

        _lastNotificationTime = DateTime.UtcNow;
    }
}
```

---

**Next Document**: [10-project-structure-and-dependencies.md](10-project-structure-and-dependencies.md)
