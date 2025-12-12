# UI and Blazor Design

## Page Structure

### Application Layout

```
/_Layout.razor
├── NavMenu (sidebar)
└── @Body
    ├── /dashboard (HomePage)
    ├── /channels (ChannelListPage)
    │   ├── /channels/new (CreateChannelPage)
    │   └── /channels/{id} (ChannelDetailPage)
    ├── /settings (UserSettingsPage)
    ├── /admin (AdminDashboardPage)
    ├── /login (LoginPage)
    ├── /register (RegisterPage)
    ├── /confirm-email (EmailConfirmationPage)
    ├── /confirm-recipient (RecipientConfirmationPage)
    └── /unsubscribe (UnsubscribePage)
```

## Core Pages

### Dashboard (/dashboard)

**Purpose**: User homepage showing overview of channels and recent ideas

```razor
@page "/dashboard"
@attribute [Authorize]

<PageTitle>Dashboard - IdeaDump</PageTitle>

<div class="dashboard">
    <div class="welcome">
        <h1>Welcome, @UserDisplayName</h1>
        <p>Rate limit: @RateLimit.Remaining / @RateLimit.Limit ideas remaining today</p>
    </div>

    <div class="stats">
        <div class="stat-card">
            <h3>@ChannelCount</h3>
            <p>Channels</p>
        </div>
        <div class="stat-card">
            <h3>@TotalIdeas</h3>
            <p>Ideas Submitted</p>
        </div>
        <div class="stat-card">
            <h3>@ElaboratedToday</h3>
            <p>Elaborated Today</p>
        </div>
    </div>

    <div class="recent-ideas">
        <h2>Recent Ideas</h2>
        @foreach (var idea in RecentIdeas)
        {
            <IdeaCard Idea="@idea" />
        }
    </div>
</div>

@code {
    private string UserDisplayName = "";
    private int ChannelCount;
    private int TotalIdeas;
    private int ElaboratedToday;
    private RateLimitStatus RateLimit = new();
    private List<IdeaSummary> RecentIdeas = new();

    protected override async Task OnInitializedAsync()
    {
        var userId = await GetCurrentUserIdAsync();

        UserDisplayName = await UserService.GetDisplayNameAsync(userId);
        ChannelCount = await ChannelService.GetChannelCountAsync(userId);
        TotalIdeas = await IdeaService.GetTotalIdeaCountAsync(userId);
        ElaboratedToday = await IdeaService.GetElaboratedTodayCountAsync(userId);
        RateLimit = await RateLimitService.CheckRateLimitAsync(userId);
        RecentIdeas = await IdeaService.GetRecentIdeasAsync(userId, 10);
    }
}
```

### Channel List (/channels)

```razor
@page "/channels"
@attribute [Authorize]

<PageTitle>My Channels - IdeaDump</PageTitle>

<div class="page-header">
    <h1>My Channels</h1>
    <button class="btn btn-primary" @onclick="NavigateToNewChannel">
        <span class="icon">➕</span> New Channel
    </button>
</div>

@if (channels == null)
{
    <div class="loading">Loading channels...</div>
}
else if (!channels.Any())
{
    <div class="empty-state">
        <h3>No channels yet</h3>
        <p>Create your first channel to start capturing and elaborating ideas!</p>
        <button class="btn btn-primary" @onclick="NavigateToNewChannel">Create Channel</button>
    </div>
}
else
{
    <div class="channel-grid">
        @foreach (var channel in channels)
        {
            <ChannelCard Channel="@channel" OnDelete="HandleChannelDeleted" />
        }
    </div>
}
```

### Channel Detail (/channels/{id})

```razor
@page "/channels/{ChannelId:guid}"
@attribute [Authorize]
@implements IDisposable

<PageTitle>@channel?.Name - IdeaDump</PageTitle>

<div class="channel-detail">
    <div class="channel-header">
        <h1>@channel?.Name</h1>
        <button class="btn btn-secondary" @onclick="EditChannel">Edit</button>
    </div>

    <div class="channel-info">
        <p>@channel?.Description</p>
        <div class="meta">
            <span>@recipientCount confirmed recipients</span>
            <span>@ideaCount ideas</span>
        </div>
    </div>

    <div class="idea-submission">
        <h2>Add New Idea</h2>

        @if (!canSubmit)
        {
            <div class="alert alert-warning">
                Daily rate limit reached. Resets at @rateLimit.ResetsAt.ToLocalTime()
            </div>
        }
        else
        {
            <EditForm Model="@newIdea" OnValidSubmit="SubmitIdea">
                <DataAnnotationsValidator />
                <ValidationSummary />

                <div class="form-group">
                    <label>Your Idea</label>
                    <InputTextArea @bind-Value="newIdea.RawText"
                                   rows="6"
                                   placeholder="Write your fragmented idea here. AI will elaborate it into a comprehensive report." />
                </div>

                <button type="submit" class="btn btn-primary" disabled="@isSubmitting">
                    @(isSubmitting ? "Submitting..." : "Submit Idea")
                </button>
            </EditForm>
        }
    </div>

    <div class="ideas-list">
        <h2>Ideas</h2>

        @foreach (var idea in ideas)
        {
            <div class="idea-item @GetStatusClass(idea.Status)">
                <div class="idea-header">
                    <strong>@GetStatusDisplay(idea.Status)</strong>
                    <span class="timestamp">@idea.CreatedAt.ToLocalTime()</span>
                </div>

                <div class="idea-text">@idea.RawText</div>

                @if (idea.Status >= IdeaEntryStatus.Elaborated)
                {
                    <button @onclick="() => ShowElaboration(idea.Id)">View Elaboration</button>
                }

                @if (idea.Status == IdeaEntryStatus.Failed)
                {
                    <div class="error">@idea.ErrorMessage</div>
                }
            </div>
        }
    </div>

    <div class="recipients">
        <RecipientManagementComponent ChannelId="@ChannelId" />
    </div>
</div>

@code {
    [Parameter] public Guid ChannelId { get; set; }

    private Channel? channel;
    private List<IdeaSummary> ideas = new();
    private int recipientCount;
    private int ideaCount;
    private RateLimitStatus rateLimit = new();
    private bool canSubmit;
    private bool isSubmitting;
    private NewIdeaModel newIdea = new();

    // Real-time updates via SignalR
    private HubConnection? hubConnection;

    protected override async Task OnInitializedAsync()
    {
        await LoadChannelDataAsync();
        await InitializeSignalRAsync();
    }

    private async Task LoadChannelDataAsync()
    {
        var userId = await GetCurrentUserIdAsync();

        channel = await ChannelService.GetChannelAsync(ChannelId, userId);
        ideas = await IdeaService.GetChannelIdeasAsync(ChannelId, userId);
        recipientCount = await RecipientService.GetConfirmedCountAsync(ChannelId, userId);
        ideaCount = ideas.Count;

        rateLimit = await RateLimitService.CheckRateLimitAsync(userId);
        canSubmit = !rateLimit.IsExceeded;
    }

    private async Task InitializeSignalRAsync()
    {
        hubConnection = new HubConnectionBuilder()
            .WithUrl(NavigationManager.ToAbsoluteUri("/ideahub"))
            .Build();

        hubConnection.On<Guid, string>("IdeaStatusChanged", async (ideaId, status) =>
        {
            // Refresh idea list when status changes
            await LoadChannelDataAsync();
            StateHasChanged();
        });

        await hubConnection.StartAsync();
    }

    private async Task SubmitIdea()
    {
        if (!canSubmit) return;

        isSubmitting = true;

        try
        {
            var userId = await GetCurrentUserIdAsync();
            await IdeaService.CreateIdeaAsync(ChannelId, newIdea.RawText, userId);

            newIdea = new();
            await LoadChannelDataAsync();
        }
        finally
        {
            isSubmitting = false;
        }
    }

    public void Dispose()
    {
        hubConnection?.DisposeAsync();
    }
}
```

### Admin Dashboard (/admin)

```razor
@page "/admin"
@attribute [Authorize(Policy = "AdminOnly")]

<PageTitle>Admin Dashboard - IdeaDump</PageTitle>

<div class="admin-dashboard">
    <h1>Admin Dashboard</h1>

    <div class="admin-tabs">
        <button class="@(activeTab == "users" ? "active" : "")" @onclick='() => activeTab = "users"'>
            Users
        </button>
        <button class="@(activeTab == "logs" ? "active" : "")" @onclick='() => activeTab = "logs"'>
            System Logs
        </button>
        <button class="@(activeTab == "stats" ? "active" : "")" @onclick='() => activeTab = "stats"'>
            Statistics
        </button>
        <button class="@(activeTab == "config" ? "active" : "")" @onclick='() => activeTab = "config"'>
            Configuration
        </button>
    </div>

    @if (activeTab == "users")
    {
        <AdminUsersPanel />
    }
    else if (activeTab == "logs")
    {
        <AdminLogsPanel />
    }
    else if (activeTab == "stats")
    {
        <AdminStatsPanel />
    }
    else if (activeTab == "config")
    {
        <AdminConfigPanel />
    }
</div>

@code {
    private string activeTab = "users";
}
```

## Reusable Components

### IdeaCard Component

```razor
<div class="idea-card status-@Idea.Status.ToString().ToLower()">
    <div class="idea-card-header">
        <div class="channel-badge">@Idea.ChannelName</div>
        <div class="status-badge">@GetStatusDisplay(Idea.Status)</div>
    </div>

    <div class="idea-card-body">
        <p class="idea-text">@TruncateText(Idea.RawText, 200)</p>

        @if (Idea.GeneratedTitle != null)
        {
            <h4 class="generated-title">@Idea.GeneratedTitle</h4>
        }
    </div>

    <div class="idea-card-footer">
        <span class="timestamp">@Idea.CreatedAt.ToLocalTime()</span>

        @if (Idea.Status >= IdeaEntryStatus.Elaborated)
        {
            <button class="btn btn-sm btn-primary" @onclick="ViewElaboration">
                View Elaboration
            </button>
        }
    </div>
</div>

@code {
    [Parameter] public IdeaSummary Idea { get; set; } = null!;
    [Parameter] public EventCallback<Guid> OnViewElaboration { get; set; }

    private async Task ViewElaboration()
    {
        await OnViewElaboration.InvokeAsync(Idea.Id);
    }
}
```

### RecipientManagementComponent

```razor
<div class="recipient-management">
    <h3>Recipients</h3>

    <div class="add-recipient-form">
        <EditForm Model="@newRecipient" OnValidSubmit="AddRecipient">
            <div class="form-inline">
                <InputText @bind-Value="newRecipient.Email" placeholder="Email address" />
                <InputText @bind-Value="newRecipient.DisplayName" placeholder="Name (optional)" />
                <button type="submit" class="btn btn-primary">Add</button>
            </div>
        </EditForm>
    </div>

    <div class="recipient-list">
        @if (!recipients.Any())
        {
            <p class="text-muted">No recipients yet. Add one to start sharing ideas!</p>
        }
        else
        {
            <table class="table">
                <thead>
                    <tr>
                        <th>Email</th>
                        <th>Status</th>
                        <th>Confirmed</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    @foreach (var recipient in recipients)
                    {
                        <tr>
                            <td>
                                @recipient.Email
                                @if (recipient.DisplayName != null)
                                {
                                    <br /><small class="text-muted">@recipient.DisplayName</small>
                                }
                            </td>
                            <td>
                                @if (recipient.IsUnsubscribed)
                                {
                                    <span class="badge badge-secondary">Unsubscribed</span>
                                }
                                else if (recipient.IsConfirmed)
                                {
                                    <span class="badge badge-success">Confirmed</span>
                                }
                                else
                                {
                                    <span class="badge badge-warning">Pending</span>
                                }
                            </td>
                            <td>@recipient.ConfirmedAt?.ToShortDateString()</td>
                            <td>
                                @if (!recipient.IsConfirmed && !recipient.IsUnsubscribed)
                                {
                                    <button class="btn btn-sm btn-secondary" @onclick="() => ResendConfirmation(recipient.Id)">
                                        Resend
                                    </button>
                                }
                                <button class="btn btn-sm btn-danger" @onclick="() => RemoveRecipient(recipient.Id)">
                                    Remove
                                </button>
                            </td>
                        </tr>
                    }
                </tbody>
            </table>
        }
    </div>
</div>

@code {
    [Parameter] public Guid ChannelId { get; set; }

    private List<RecipientSummary> recipients = new();
    private AddRecipientRequest newRecipient = new();

    protected override async Task OnInitializedAsync()
    {
        await LoadRecipientsAsync();
    }
}
```

## Real-Time Updates (SignalR)

### IdeaHub

```csharp
public class IdeaHub : Hub
{
    public async Task SubscribeToIdea(Guid ideaId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"idea-{ideaId}");
    }

    public async Task SubscribeToChannel(Guid channelId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"channel-{channelId}");
    }
}

// Service to push updates
public class IdeaUpdateNotifier
{
    private readonly IHubContext<IdeaHub> _hubContext;

    public async Task NotifyIdeaStatusChangedAsync(Guid ideaId, Guid channelId, string status)
    {
        await _hubContext.Clients
            .Group($"idea-{ideaId}")
            .SendAsync("IdeaStatusChanged", ideaId, status);

        await _hubContext.Clients
            .Group($"channel-{channelId}")
            .SendAsync("ChannelIdeaUpdated", ideaId, status);
    }
}
```

## CSS/Styling

### Global Styles (app.css)

```css
:root {
    --primary-color: #007bff;
    --success-color: #28a745;
    --warning-color: #ffc107;
    --danger-color: #dc3545;
    --secondary-color: #6c757d;
    --light-bg: #f8f9fa;
}

.idea-card {
    border: 1px solid #dee2e6;
    border-radius: 8px;
    padding: 16px;
    margin-bottom: 16px;
    transition: box-shadow 0.2s;
}

.idea-card:hover {
    box-shadow: 0 4px 12px rgba(0,0,0,0.1);
}

.idea-card.status-new {
    border-left: 4px solid var(--secondary-color);
}

.idea-card.status-elaborating {
    border-left: 4px solid var(--warning-color);
    animation: pulse 2s infinite;
}

.idea-card.status-elaborated {
    border-left: 4px solid var(--success-color);
}

.idea-card.status-sent {
    border-left: 4px solid var(--primary-color);
}

.idea-card.status-failed {
    border-left: 4px solid var(--danger-color);
}

@keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.7; }
}
```

### Create Channel Page (/channels/new)

**Purpose**: Form to create a new channel

```razor
@page "/channels/new"
@attribute [Authorize]
@inject IChannelService ChannelService
@inject NavigationManager Nav

<PageTitle>Create Channel - IdeaDump</PageTitle>

<div class="create-channel-page">
    <h1>Create New Channel</h1>

    <EditForm Model="@model" OnValidSubmit="@CreateChannel">
        <DataAnnotationsValidator />
        <ValidationSummary />

        <div class="form-section">
            <h3>Basic Information</h3>

            <div class="form-group">
                <label for="name">Channel Name *</label>
                <InputText id="name" @bind-Value="model.Name" class="form-control"
                          placeholder="e.g., Work Ideas, Personal Projects" />
                <ValidationMessage For="@(() => model.Name)" />
            </div>

            <div class="form-group">
                <label for="description">Description</label>
                <InputTextArea id="description" @bind-Value="model.Description"
                              class="form-control" rows="3"
                              placeholder="Brief description of this channel's purpose (optional)" />
            </div>
        </div>

        <div class="form-section">
            <h3>AI Configuration</h3>

            <div class="form-group">
                <label for="aiInstructions">Custom AI Instructions</label>
                <InputTextArea id="aiInstructions" @bind-Value="model.AiInstructions"
                              class="form-control" rows="6"
                              placeholder="Leave empty to use system defaults, or provide custom instructions for AI elaboration" />
                <small class="form-text text-muted">
                    Example: "Focus on technical feasibility and implementation details. Assume the user is a software developer."
                </small>
            </div>

            <div class="form-group">
                <label for="apiKey">Channel-Specific API Key</label>
                <InputText id="apiKey" type="password" @bind-Value="model.ApiKey"
                          class="form-control"
                          placeholder="Leave empty to use your user-level or system-wide key" />
                <small class="form-text text-muted">
                    Optional: Provide a channel-specific AI API key to override defaults
                </small>
            </div>
        </div>

        <div class="form-section">
            <h3>Email Configuration</h3>

            <div class="form-check mb-3">
                <input type="checkbox" class="form-check-input" id="useCustomSmtp"
                       @bind="useCustomSmtp" />
                <label class="form-check-label" for="useCustomSmtp">
                    Use custom SMTP settings for this channel
                </label>
            </div>

            @if (useCustomSmtp)
            {
                <div class="smtp-config">
                    <div class="row">
                        <div class="col-md-8">
                            <div class="form-group">
                                <label for="smtpServer">SMTP Server *</label>
                                <InputText id="smtpServer" @bind-Value="model.SmtpServer"
                                          class="form-control"
                                          placeholder="smtp.example.com" />
                            </div>
                        </div>
                        <div class="col-md-4">
                            <div class="form-group">
                                <label for="smtpPort">Port *</label>
                                <InputNumber id="smtpPort" @bind-Value="model.SmtpPort"
                                            class="form-control"
                                            placeholder="587" />
                            </div>
                        </div>
                    </div>

                    <div class="form-check mb-3">
                        <input type="checkbox" class="form-check-input" id="smtpUseSsl"
                               @bind="model.SmtpUseSsl" />
                        <label class="form-check-label" for="smtpUseSsl">
                            Use SSL/TLS (recommended)
                        </label>
                    </div>

                    <div class="form-group">
                        <label for="smtpUsername">SMTP Username *</label>
                        <InputText id="smtpUsername" @bind-Value="model.SmtpUsername"
                                  class="form-control"
                                  placeholder="your-email@example.com" />
                    </div>

                    <div class="form-group">
                        <label for="smtpPassword">SMTP Password *</label>
                        <InputText id="smtpPassword" type="password"
                                  @bind-Value="model.SmtpPassword"
                                  class="form-control" />
                    </div>

                    <div class="row">
                        <div class="col-md-6">
                            <div class="form-group">
                                <label for="fromEmail">From Email *</label>
                                <InputText id="fromEmail" type="email"
                                          @bind-Value="model.FromEmail"
                                          class="form-control"
                                          placeholder="ideas@example.com" />
                            </div>
                        </div>
                        <div class="col-md-6">
                            <div class="form-group">
                                <label for="fromName">From Name *</label>
                                <InputText id="fromName" @bind-Value="model.FromName"
                                          class="form-control"
                                          placeholder="My Ideas Channel" />
                            </div>
                        </div>
                    </div>
                </div>
            }
        </div>

        <div class="form-actions">
            <button type="submit" class="btn btn-primary" disabled="@isSubmitting">
                @(isSubmitting ? "Creating..." : "Create Channel")
            </button>
            <button type="button" class="btn btn-secondary" @onclick="Cancel">
                Cancel
            </button>
        </div>
    </EditForm>
</div>

@code {
    private CreateChannelRequest model = new();
    private bool useCustomSmtp = false;
    private bool isSubmitting = false;

    protected override void OnInitialized()
    {
        // Set defaults
        model.SmtpPort = 587;
        model.SmtpUseSsl = true;
    }

    private async Task CreateChannel()
    {
        isSubmitting = true;

        try
        {
            var userId = await GetCurrentUserIdAsync();

            if (!useCustomSmtp)
            {
                // Clear SMTP fields if not using custom settings
                model.SmtpServer = null;
                model.SmtpUsername = null;
                model.SmtpPassword = null;
                model.FromEmail = null;
                model.FromName = null;
            }

            var channel = await ChannelService.CreateChannelAsync(model, userId);
            Nav.NavigateTo($"/channels/{channel.Id}");
        }
        catch (Exception ex)
        {
            // Show error message
            await JS.InvokeVoidAsync("alert", $"Error creating channel: {ex.Message}");
        }
        finally
        {
            isSubmitting = false;
        }
    }

    private void Cancel()
    {
        Nav.NavigateTo("/channels");
    }
}
```

### User Settings Page (/settings)

**Purpose**: User profile and configuration management

```razor
@page "/settings"
@attribute [Authorize]
@inject IUserService UserService
@inject SignInManager<ApplicationUser> SignInManager

<PageTitle>Settings - IdeaDump</PageTitle>

<div class="settings-page">
    <h1>Account Settings</h1>

    <div class="settings-tabs">
        <button class="@(activeTab == "profile" ? "active" : "")"
                @onclick='() => activeTab = "profile"'>
            Profile
        </button>
        <button class="@(activeTab == "api" ? "active" : "")"
                @onclick='() => activeTab = "api"'>
            API Configuration
        </button>
        <button class="@(activeTab == "security" ? "active" : "")"
                @onclick='() => activeTab = "security"'>
            Security
        </button>
    </div>

    @if (activeTab == "profile")
    {
        <div class="settings-section">
            <h2>Profile Information</h2>

            <EditForm Model="@profileModel" OnValidSubmit="@UpdateProfile">
                <div class="form-group">
                    <label>Email</label>
                    <input type="email" value="@user?.Email" class="form-control" disabled />
                    <small class="form-text text-muted">Email cannot be changed</small>
                </div>

                <div class="form-group">
                    <label for="displayName">Display Name</label>
                    <InputText id="displayName" @bind-Value="profileModel.DisplayName"
                              class="form-control" />
                </div>

                <div class="info-box">
                    <strong>Rate Limit:</strong> @user?.DailyIdeaLimit ideas per day<br />
                    <strong>Used Today:</strong> @user?.IdeasElaboratedToday<br />
                    <strong>Member Since:</strong> @user?.CreatedAt.ToShortDateString()
                </div>

                <button type="submit" class="btn btn-primary">Update Profile</button>
            </EditForm>
        </div>
    }
    else if (activeTab == "api")
    {
        <div class="settings-section">
            <h2>API Configuration</h2>

            <div class="alert alert-info">
                <strong>API Key Hierarchy:</strong>
                <ul>
                    <li>System-wide key (default for all users)</li>
                    <li>User-wide key (overrides system key for all your channels)</li>
                    <li>Channel-specific key (overrides everything for that channel)</li>
                </ul>
            </div>

            <EditForm Model="@apiModel" OnValidSubmit="@UpdateApiConfig">
                <div class="form-group">
                    <label for="apiKey">Your API Key</label>
                    <InputText id="apiKey" type="password" @bind-Value="apiModel.ApiKey"
                              class="form-control"
                              placeholder="Leave empty to use system default" />
                    <small class="form-text text-muted">
                        This key will be used for all your channels unless they have their own key
                    </small>
                </div>

                @if (hasApiKey)
                {
                    <div class="alert alert-success">
                        You have a user-wide API key configured
                    </div>
                }

                <button type="submit" class="btn btn-primary">Save API Key</button>

                @if (hasApiKey)
                {
                    <button type="button" class="btn btn-danger" @onclick="RemoveApiKey">
                        Remove API Key
                    </button>
                }
            </EditForm>
        </div>
    }
    else if (activeTab == "security")
    {
        <div class="settings-section">
            <h2>Security</h2>

            <div class="security-section">
                <h3>Change Password</h3>
                <EditForm Model="@passwordModel" OnValidSubmit="@ChangePassword">
                    <div class="form-group">
                        <label for="currentPassword">Current Password</label>
                        <InputText id="currentPassword" type="password"
                                  @bind-Value="passwordModel.CurrentPassword"
                                  class="form-control" />
                    </div>

                    <div class="form-group">
                        <label for="newPassword">New Password</label>
                        <InputText id="newPassword" type="password"
                                  @bind-Value="passwordModel.NewPassword"
                                  class="form-control" />
                        <small class="form-text text-muted">
                            Min 8 characters, must include uppercase, lowercase, digit, special character
                        </small>
                    </div>

                    <div class="form-group">
                        <label for="confirmPassword">Confirm New Password</label>
                        <InputText id="confirmPassword" type="password"
                                  @bind-Value="passwordModel.ConfirmPassword"
                                  class="form-control" />
                    </div>

                    <button type="submit" class="btn btn-primary">Change Password</button>
                </EditForm>
            </div>

            <div class="security-section danger-zone">
                <h3>Danger Zone</h3>
                <p>Once you delete your account, there is no going back. This will delete:</p>
                <ul>
                    <li>All your channels</li>
                    <li>All your ideas</li>
                    <li>All recipients associated with your channels</li>
                </ul>
                <button type="button" class="btn btn-danger" @onclick="DeleteAccount">
                    Delete Account
                </button>
            </div>
        </div>
    }
</div>

@code {
    private ApplicationUser? user;
    private string activeTab = "profile";
    private bool hasApiKey;

    private ProfileUpdateModel profileModel = new();
    private ApiConfigModel apiModel = new();
    private PasswordChangeModel passwordModel = new();

    protected override async Task OnInitializedAsync()
    {
        var userId = await GetCurrentUserIdAsync();
        user = await UserService.GetUserAsync(userId);

        profileModel.DisplayName = user.DisplayName;
        hasApiKey = !string.IsNullOrEmpty(user.ApiKey);
    }

    private async Task UpdateProfile()
    {
        var userId = await GetCurrentUserIdAsync();
        await UserService.UpdateProfileAsync(userId, profileModel);
        await ShowToast("Profile updated successfully");
    }

    private async Task UpdateApiConfig()
    {
        var userId = await GetCurrentUserIdAsync();
        await UserService.UpdateApiKeyAsync(userId, apiModel.ApiKey);
        hasApiKey = !string.IsNullOrEmpty(apiModel.ApiKey);
        apiModel.ApiKey = ""; // Clear form
        await ShowToast("API key updated successfully");
    }

    private async Task RemoveApiKey()
    {
        if (await JS.InvokeAsync<bool>("confirm", "Remove your API key? You'll use the system default."))
        {
            var userId = await GetCurrentUserIdAsync();
            await UserService.UpdateApiKeyAsync(userId, null);
            hasApiKey = false;
            await ShowToast("API key removed");
        }
    }

    private async Task ChangePassword()
    {
        if (passwordModel.NewPassword != passwordModel.ConfirmPassword)
        {
            await ShowToast("Passwords don't match", "error");
            return;
        }

        var result = await UserService.ChangePasswordAsync(
            user!.Id,
            passwordModel.CurrentPassword,
            passwordModel.NewPassword);

        if (result.Succeeded)
        {
            passwordModel = new();
            await ShowToast("Password changed successfully");
        }
        else
        {
            await ShowToast(string.Join(", ", result.Errors.Select(e => e.Description)), "error");
        }
    }

    private async Task DeleteAccount()
    {
        var confirmed = await JS.InvokeAsync<bool>("confirm",
            "Are you absolutely sure? This action cannot be undone. Type 'DELETE' to confirm.");

        if (confirmed)
        {
            var userId = await GetCurrentUserIdAsync();
            await UserService.DeleteAccountAsync(userId);
            await SignInManager.SignOutAsync();
            Nav.NavigateTo("/");
        }
    }
}
```

### Login Page (/login)

**Purpose**: User authentication

```razor
@page "/login"
@layout EmptyLayout
@inject SignInManager<ApplicationUser> SignInManager
@inject NavigationManager Nav

<PageTitle>Login - IdeaDump</PageTitle>

<div class="auth-page">
    <div class="auth-container">
        <div class="auth-header">
            <h1>💡 IdeaDump</h1>
            <p>Turn fragmented ideas into actionable plans</p>
        </div>

        <div class="auth-form">
            <h2>Log In</h2>

            @if (!string.IsNullOrEmpty(errorMessage))
            {
                <div class="alert alert-danger">@errorMessage</div>
            }

            <EditForm Model="@model" OnValidSubmit="@HandleLogin">
                <DataAnnotationsValidator />

                <div class="form-group">
                    <label for="email">Email</label>
                    <InputText id="email" type="email" @bind-Value="model.Email"
                              class="form-control" placeholder="your@email.com" />
                    <ValidationMessage For="@(() => model.Email)" />
                </div>

                <div class="form-group">
                    <label for="password">Password</label>
                    <InputText id="password" type="password" @bind-Value="model.Password"
                              class="form-control" />
                    <ValidationMessage For="@(() => model.Password)" />
                </div>

                <div class="form-check">
                    <InputCheckbox id="rememberMe" @bind-Value="model.RememberMe"
                                  class="form-check-input" />
                    <label class="form-check-label" for="rememberMe">
                        Remember me
                    </label>
                </div>

                <button type="submit" class="btn btn-primary btn-block" disabled="@isSubmitting">
                    @(isSubmitting ? "Logging in..." : "Log In")
                </button>
            </EditForm>

            <div class="auth-links">
                <a href="/register">Don't have an account? Register</a>
                <a href="/forgot-password">Forgot password?</a>
            </div>
        </div>
    </div>
</div>

@code {
    private LoginModel model = new();
    private bool isSubmitting = false;
    private string? errorMessage;

    private async Task HandleLogin()
    {
        isSubmitting = true;
        errorMessage = null;

        try
        {
            var result = await SignInManager.PasswordSignInAsync(
                model.Email,
                model.Password,
                model.RememberMe,
                lockoutOnFailure: true
            );

            if (result.Succeeded)
            {
                Nav.NavigateTo("/dashboard");
            }
            else if (result.IsLockedOut)
            {
                errorMessage = "Account locked due to multiple failed login attempts. Please try again later.";
            }
            else if (result.IsNotAllowed)
            {
                errorMessage = "Please confirm your email before logging in.";
            }
            else
            {
                errorMessage = "Invalid email or password.";
            }
        }
        finally
        {
            isSubmitting = false;
        }
    }

    public class LoginModel
    {
        [Required, EmailAddress]
        public string Email { get; set; } = "";

        [Required]
        public string Password { get; set; } = "";

        public bool RememberMe { get; set; }
    }
}
```

### Register Page (/register)

**Purpose**: New user registration

```razor
@page "/register"
@layout EmptyLayout
@inject UserManager<ApplicationUser> UserManager
@inject IEmailService EmailService
@inject IConfiguration Config
@inject NavigationManager Nav

<PageTitle>Register - IdeaDump</PageTitle>

<div class="auth-page">
    <div class="auth-container">
        <div class="auth-header">
            <h1>💡 IdeaDump</h1>
            <p>Turn fragmented ideas into actionable plans</p>
        </div>

        <div class="auth-form">
            <h2>Create Account</h2>

            @if (registrationSuccess)
            {
                <div class="alert alert-success">
                    <h4>Registration Successful!</h4>
                    <p>We've sent a confirmation email to <strong>@model.Email</strong></p>
                    <p>Please check your inbox and click the confirmation link to activate your account.</p>
                    <a href="/login" class="btn btn-primary">Go to Login</a>
                </div>
            }
            else
            {
                @if (errors.Any())
                {
                    <div class="alert alert-danger">
                        <ul>
                            @foreach (var error in errors)
                            {
                                <li>@error</li>
                            }
                        </ul>
                    </div>
                }

                <EditForm Model="@model" OnValidSubmit="@HandleRegistration">
                    <DataAnnotationsValidator />

                    <div class="form-group">
                        <label for="email">Email *</label>
                        <InputText id="email" type="email" @bind-Value="model.Email"
                                  class="form-control" placeholder="your@email.com" />
                        <ValidationMessage For="@(() => model.Email)" />
                    </div>

                    <div class="form-group">
                        <label for="displayName">Display Name</label>
                        <InputText id="displayName" @bind-Value="model.DisplayName"
                                  class="form-control" placeholder="Optional" />
                    </div>

                    <div class="form-group">
                        <label for="password">Password *</label>
                        <InputText id="password" type="password" @bind-Value="model.Password"
                                  class="form-control" />
                        <small class="form-text text-muted">
                            Min 8 characters, must include uppercase, lowercase, digit, special character
                        </small>
                        <ValidationMessage For="@(() => model.Password)" />
                    </div>

                    <div class="form-group">
                        <label for="confirmPassword">Confirm Password *</label>
                        <InputText id="confirmPassword" type="password"
                                  @bind-Value="model.ConfirmPassword"
                                  class="form-control" />
                        <ValidationMessage For="@(() => model.ConfirmPassword)" />
                    </div>

                    <button type="submit" class="btn btn-primary btn-block" disabled="@isSubmitting">
                        @(isSubmitting ? "Creating account..." : "Register")
                    </button>
                </EditForm>

                <div class="auth-links">
                    <a href="/login">Already have an account? Log in</a>
                </div>
            }
        </div>
    </div>
</div>

@code {
    private RegisterModel model = new();
    private bool isSubmitting = false;
    private bool registrationSuccess = false;
    private List<string> errors = new();

    private async Task HandleRegistration()
    {
        isSubmitting = true;
        errors.Clear();

        try
        {
            if (model.Password != model.ConfirmPassword)
            {
                errors.Add("Passwords don't match");
                return;
            }

            var user = new ApplicationUser
            {
                Id = Guid.NewGuid(),
                UserName = model.Email,
                Email = model.Email,
                DisplayName = model.DisplayName,
                EmailConfirmed = false,
                DailyIdeaLimit = int.Parse(Config["System:DefaultDailyLimit"] ?? "10"),
                CreatedAt = DateTime.UtcNow
            };

            var result = await UserManager.CreateAsync(user, model.Password);

            if (result.Succeeded)
            {
                // Generate email confirmation token
                var token = await UserManager.GenerateEmailConfirmationTokenAsync(user);
                var baseUrl = Config["App:BaseUrl"];
                var confirmUrl = $"{baseUrl}/confirm-email?userId={user.Id}&token={Uri.EscapeDataString(token)}";

                // Send confirmation email
                await EmailService.SendEmailAsync(
                    to: user.Email,
                    subject: "Confirm your IdeaDump account",
                    body: BuildConfirmationEmailHtml(confirmUrl),
                    smtpConfig: GetSystemSmtpConfig()
                );

                registrationSuccess = true;
            }
            else
            {
                errors.AddRange(result.Errors.Select(e => e.Description));
            }
        }
        catch (Exception ex)
        {
            errors.Add($"Registration failed: {ex.Message}");
        }
        finally
        {
            isSubmitting = false;
        }
    }

    private string BuildConfirmationEmailHtml(string confirmUrl)
    {
        return $@"
<html>
<body style='font-family: Arial, sans-serif;'>
    <h2>Welcome to IdeaDump!</h2>
    <p>Please confirm your email address by clicking the link below:</p>
    <p><a href='{confirmUrl}' style='display: inline-block; padding: 12px 24px; background: #007bff; color: white; text-decoration: none; border-radius: 4px;'>Confirm Email</a></p>
    <p><small>Or copy this link: {confirmUrl}</small></p>
    <p><small>This link will expire in 24 hours.</small></p>
</body>
</html>";
    }

    public class RegisterModel
    {
        [Required, EmailAddress]
        public string Email { get; set; } = "";

        public string? DisplayName { get; set; }

        [Required, MinLength(8)]
        public string Password { get; set; } = "";

        [Required]
        [Compare(nameof(Password), ErrorMessage = "Passwords don't match")]
        public string ConfirmPassword { get; set; } = "";
    }
}
```

### Email Confirmation Page (/confirm-email)

**Purpose**: Activate user account via email link

```razor
@page "/confirm-email"
@layout EmptyLayout
@inject UserManager<ApplicationUser> UserManager
@inject NavigationManager Nav

<PageTitle>Confirm Email - IdeaDump</PageTitle>

<div class="auth-page">
    <div class="auth-container">
        <div class="auth-header">
            <h1>💡 IdeaDump</h1>
        </div>

        <div class="confirmation-result">
            @if (isProcessing)
            {
                <div class="text-center">
                    <div class="spinner-border" role="status">
                        <span class="sr-only">Confirming...</span>
                    </div>
                    <p>Confirming your email...</p>
                </div>
            }
            else if (success)
            {
                <div class="alert alert-success">
                    <h3>✓ Email Confirmed!</h3>
                    <p>Your account has been activated. You can now log in.</p>
                    <a href="/login" class="btn btn-primary">Go to Login</a>
                </div>
            }
            else
            {
                <div class="alert alert-danger">
                    <h3>✗ Confirmation Failed</h3>
                    <p>@errorMessage</p>
                    <a href="/register" class="btn btn-secondary">Register Again</a>
                </div>
            }
        </div>
    </div>
</div>

@code {
    [SupplyParameterFromQuery]
    public string? UserId { get; set; }

    [SupplyParameterFromQuery]
    public string? Token { get; set; }

    private bool isProcessing = true;
    private bool success = false;
    private string? errorMessage;

    protected override async Task OnInitializedAsync()
    {
        if (string.IsNullOrEmpty(UserId) || string.IsNullOrEmpty(Token))
        {
            isProcessing = false;
            errorMessage = "Invalid confirmation link";
            return;
        }

        try
        {
            var user = await UserManager.FindByIdAsync(UserId);

            if (user == null)
            {
                errorMessage = "User not found";
                return;
            }

            if (user.EmailConfirmed)
            {
                success = true;
                return;
            }

            var result = await UserManager.ConfirmEmailAsync(user, Token);

            if (result.Succeeded)
            {
                success = true;
            }
            else
            {
                errorMessage = string.Join(", ", result.Errors.Select(e => e.Description));
            }
        }
        catch (Exception ex)
        {
            errorMessage = $"An error occurred: {ex.Message}";
        }
        finally
        {
            isProcessing = false;
        }
    }
}
```

### Recipient Confirmation Page (/confirm-recipient)

**Purpose**: Confirm recipient subscription to a channel

```razor
@page "/confirm-recipient"
@layout EmptyLayout
@inject IRecipientService RecipientService

<PageTitle>Confirm Subscription - IdeaDump</PageTitle>

<div class="auth-page">
    <div class="auth-container">
        <div class="auth-header">
            <h1>💡 IdeaDump</h1>
            <p>Confirm Your Subscription</p>
        </div>

        <div class="confirmation-result">
            @if (isProcessing)
            {
                <div class="text-center">
                    <div class="spinner-border" role="status">
                        <span class="sr-only">Processing...</span>
                    </div>
                    <p>Confirming your subscription...</p>
                </div>
            }
            else if (success)
            {
                <div class="alert alert-success">
                    <h3>✓ Subscription Confirmed!</h3>
                    <p>You've successfully subscribed to the channel: <strong>@channelName</strong></p>
                    <p>You'll now receive elaborated idea reports via email.</p>
                </div>
            }
            else
            {
                <div class="alert alert-danger">
                    <h3>✗ Confirmation Failed</h3>
                    <p>@errorMessage</p>
                </div>
            }
        </div>
    </div>
</div>

@code {
    [SupplyParameterFromQuery]
    public string? Token { get; set; }

    private bool isProcessing = true;
    private bool success = false;
    private string? errorMessage;
    private string? channelName;

    protected override async Task OnInitializedAsync()
    {
        if (string.IsNullOrEmpty(Token))
        {
            isProcessing = false;
            errorMessage = "Invalid confirmation link";
            return;
        }

        try
        {
            var result = await RecipientService.ConfirmRecipientAsync(Token);

            if (result.Success)
            {
                success = true;
                channelName = result.ChannelName;
            }
            else
            {
                errorMessage = result.ErrorMessage;
            }
        }
        catch (Exception ex)
        {
            errorMessage = $"An error occurred: {ex.Message}";
        }
        finally
        {
            isProcessing = false;
        }
    }
}
```

### Unsubscribe Page (/unsubscribe)

**Purpose**: One-click unsubscribe from channel

```razor
@page "/unsubscribe"
@layout EmptyLayout
@inject IRecipientService RecipientService

<PageTitle>Unsubscribe - IdeaDump</PageTitle>

<div class="auth-page">
    <div class="auth-container">
        <div class="auth-header">
            <h1>💡 IdeaDump</h1>
            <p>Unsubscribe</p>
        </div>

        <div class="confirmation-result">
            @if (isProcessing)
            {
                <div class="text-center">
                    <div class="spinner-border" role="status">
                        <span class="sr-only">Processing...</span>
                    </div>
                    <p>Processing your request...</p>
                </div>
            }
            else if (success)
            {
                <div class="alert alert-success">
                    <h3>✓ Unsubscribed</h3>
                    <p>You've been unsubscribed from: <strong>@channelName</strong></p>
                    <p>You will no longer receive emails from this channel.</p>
                    <p><small>If this was a mistake, please contact the channel owner to be re-added.</small></p>
                </div>
            }
            else
            {
                <div class="alert alert-danger">
                    <h3>✗ Unsubscribe Failed</h3>
                    <p>@errorMessage</p>
                </div>
            }
        </div>
    </div>
</div>

@code {
    [SupplyParameterFromQuery]
    public string? Token { get; set; }

    private bool isProcessing = true;
    private bool success = false;
    private string? errorMessage;
    private string? channelName;

    protected override async Task OnInitializedAsync()
    {
        if (string.IsNullOrEmpty(Token))
        {
            isProcessing = false;
            errorMessage = "Invalid unsubscribe link";
            return;
        }

        try
        {
            var result = await RecipientService.UnsubscribeRecipientAsync(Token);

            if (result.Success)
            {
                success = true;
                channelName = result.ChannelName;
            }
            else
            {
                errorMessage = result.ErrorMessage;
            }
        }
        catch (Exception ex)
        {
            errorMessage = $"An error occurred: {ex.Message}";
        }
        finally
        {
            isProcessing = false;
        }
    }
}
```

## Mobile Responsiveness

All pages use Bootstrap 5's responsive grid system:

```html
<div class="container">
    <div class="row">
        <div class="col-12 col-md-8 col-lg-6">
            <!-- Content -->
        </div>
    </div>
</div>
```

### Mobile-Specific Considerations

**Navigation**:
- Collapsed hamburger menu on mobile
- Full sidebar on desktop
- Touch-friendly buttons (min 44px tap targets)

**Forms**:
- Full-width inputs on mobile
- Multi-column layouts on tablet+
- Virtual keyboard-friendly input types

**Tables**:
- Horizontally scrollable on mobile
- Card-based layout for better mobile UX (alternative to tables)
- Stack columns vertically on small screens

**Real-Time Updates**:
- SignalR works seamlessly on mobile
- Reconnection handling for spotty connections
- Lightweight UI updates to minimize data usage

---

**Next Document**: [08-admin-features-specification.md](08-admin-features-specification.md)
