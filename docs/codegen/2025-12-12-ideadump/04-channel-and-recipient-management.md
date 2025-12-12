# Channel and Recipient Management

## Channel Operations

### Create Channel

**User Story**: As a user, I want to create a new channel to organize my ideas and specify recipients.

**Flow**:
1. User navigates to "Channels" page
2. Clicks "New Channel"
3. Fills out channel form
4. Submits form
5. Channel created, user redirected to channel detail page

**Required Fields**:
- Name (3-100 characters)
- SMTP configuration (or use system default)

**Optional Fields**:
- Description
- Custom AI instructions
- Channel-specific API key

**Code Example**:
```csharp
public async Task<Channel> CreateChannelAsync(CreateChannelRequest request, Guid userId)
{
    var channel = new Channel
    {
        Id = Guid.NewGuid(),
        UserId = userId,
        Name = request.Name,
        Description = request.Description,
        AiInstructions = request.AiInstructions,
        ApiKey = request.ApiKey != null ? _encryption.Encrypt(request.ApiKey) : null,
        SmtpServer = request.SmtpServer ?? _config["System.SmtpServer"],
        SmtpPort = request.SmtpPort ?? int.Parse(_config["System.SmtpPort"]),
        SmtpUseSsl = request.SmtpUseSsl ?? true,
        SmtpUsername = request.SmtpUsername ?? _config["System.SmtpUsername"],
        SmtpPassword = request.SmtpPassword != null
            ? _encryption.Encrypt(request.SmtpPassword)
            : _encryption.Encrypt(_config["System.SmtpPassword"]),
        FromEmail = request.FromEmail ?? _config["System.SmtpUsername"],
        FromName = request.FromName ?? "IdeaDump",
        IsActive = true,
        CreatedAt = DateTime.UtcNow,
        UpdatedAt = DateTime.UtcNow
    };

    _context.Channels.Add(channel);
    await _context.SaveChangesAsync();

    await _systemLog.LogAsync(LogLevel.Information, "Channel",
        $"Channel '{channel.Name}' created by user {userId}", userId: userId, channelId: channel.Id);

    return channel;
}
```

### Read Channels

**List All Channels (User's Channels)**:
```csharp
public async Task<List<ChannelSummary>> GetUserChannelsAsync(Guid userId)
{
    return await _context.Channels
        .Where(c => c.UserId == userId)
        .OrderByDescending(c => c.CreatedAt)
        .Select(c => new ChannelSummary
        {
            Id = c.Id,
            Name = c.Name,
            Description = c.Description,
            IsActive = c.IsActive,
            RecipientCount = c.Recipients.Count(r => r.IsConfirmed && !r.IsUnsubscribed),
            IdeaCount = c.Ideas.Count,
            CreatedAt = c.CreatedAt
        })
        .ToListAsync();
}
```

**Get Channel Details**:
```csharp
public async Task<Channel> GetChannelAsync(Guid channelId, Guid userId)
{
    var channel = await _context.Channels
        .Include(c => c.Recipients)
        .Include(c => c.Ideas.OrderByDescending(i => i.CreatedAt).Take(10))
        .FirstOrDefaultAsync(c => c.Id == channelId && c.UserId == userId);

    if (channel == null)
        throw new UnauthorizedAccessException("Channel not found or access denied");

    // Decrypt sensitive fields for display (masked)
    channel.SmtpPassword = "********";
    if (channel.ApiKey != null)
        channel.ApiKey = "********";

    return channel;
}
```

### Update Channel

**User Story**: As a user, I want to update my channel's name, description, AI instructions, or SMTP settings.

**Code Example**:
```csharp
public async Task<Channel> UpdateChannelAsync(Guid channelId, UpdateChannelRequest request, Guid userId)
{
    var channel = await _context.Channels
        .FirstOrDefaultAsync(c => c.Id == channelId && c.UserId == userId);

    if (channel == null)
        throw new UnauthorizedAccessException("Channel not found or access denied");

    // Update fields
    if (request.Name != null) channel.Name = request.Name;
    if (request.Description != null) channel.Description = request.Description;
    if (request.AiInstructions != null) channel.AiInstructions = request.AiInstructions;

    if (request.ApiKey != null)
        channel.ApiKey = _encryption.Encrypt(request.ApiKey);

    if (request.SmtpServer != null) channel.SmtpServer = request.SmtpServer;
    if (request.SmtpPort != null) channel.SmtpPort = request.SmtpPort.Value;
    if (request.SmtpUseSsl != null) channel.SmtpUseSsl = request.SmtpUseSsl.Value;
    if (request.SmtpUsername != null) channel.SmtpUsername = request.SmtpUsername;

    if (request.SmtpPassword != null)
        channel.SmtpPassword = _encryption.Encrypt(request.SmtpPassword);

    if (request.FromEmail != null) channel.FromEmail = request.FromEmail;
    if (request.FromName != null) channel.FromName = request.FromName;

    if (request.IsActive != null) channel.IsActive = request.IsActive.Value;

    channel.UpdatedAt = DateTime.UtcNow;
    await _context.SaveChangesAsync();

    await _systemLog.LogAsync(LogLevel.Information, "Channel",
        $"Channel '{channel.Name}' updated", userId: userId, channelId: channelId);

    return channel;
}
```

### Delete Channel

**User Story**: As a user, I want to delete a channel I no longer need.

**Cascade Behavior**:
- Deletes all recipients
- Deletes all idea entries
- Deletes all processing logs for those ideas

**Code Example**:
```csharp
public async Task DeleteChannelAsync(Guid channelId, Guid userId)
{
    var channel = await _context.Channels
        .Include(c => c.Recipients)
        .Include(c => c.Ideas)
            .ThenInclude(i => i.ProcessingLogs)
        .FirstOrDefaultAsync(c => c.Id == channelId && c.UserId == userId);

    if (channel == null)
        throw new UnauthorizedAccessException("Channel not found or access denied");

    var channelName = channel.Name;
    var recipientCount = channel.Recipients.Count;
    var ideaCount = channel.Ideas.Count;

    _context.Channels.Remove(channel); // Cascade delete configured in EF
    await _context.SaveChangesAsync();

    await _systemLog.LogAsync(LogLevel.Information, "Channel",
        $"Channel '{channelName}' deleted (had {recipientCount} recipients, {ideaCount} ideas)",
        userId: userId);
}
```

## Recipient Management

### Add Recipient to Channel

**User Story**: As a user, I want to add a recipient to my channel so they receive elaborated ideas.

**Flow**:
1. User navigates to channel detail page
2. Clicks "Add Recipient"
3. Enters recipient email and optional display name
4. Submits form
5. System creates unconfirmed recipient
6. System sends confirmation email to recipient
7. Recipient clicks confirmation link
8. Recipient is confirmed and will receive future emails

**Code Example**:
```csharp
public async Task<ChannelRecipient> AddRecipientAsync(Guid channelId, AddRecipientRequest request, Guid userId)
{
    // Verify channel ownership
    var channel = await _context.Channels
        .FirstOrDefaultAsync(c => c.Id == channelId && c.UserId == userId);

    if (channel == null)
        throw new UnauthorizedAccessException("Channel not found or access denied");

    // Check if recipient already exists
    var existing = await _context.ChannelRecipients
        .FirstOrDefaultAsync(r => r.ChannelId == channelId && r.Email == request.Email);

    if (existing != null)
    {
        if (existing.IsUnsubscribed)
        {
            // Re-subscribe: Reset unsubscribe flag and send new confirmation
            existing.IsUnsubscribed = false;
            existing.IsConfirmed = false;
            existing.ConfirmationToken = GenerateSecureToken();
            existing.ConfirmationTokenExpiry = DateTime.UtcNow.AddDays(7);
            await _context.SaveChangesAsync();

            await SendRecipientConfirmationEmailAsync(existing, channel);
            return existing;
        }
        else
        {
            throw new InvalidOperationException("Recipient already exists in this channel");
        }
    }

    // Create new recipient
    var recipient = new ChannelRecipient
    {
        Id = Guid.NewGuid(),
        ChannelId = channelId,
        Email = request.Email,
        DisplayName = request.DisplayName,
        IsConfirmed = false,
        ConfirmationToken = GenerateSecureToken(),
        ConfirmationTokenExpiry = DateTime.UtcNow.AddDays(7),
        CreatedAt = DateTime.UtcNow,
        UnsubscribeToken = Guid.NewGuid().ToString()
    };

    _context.ChannelRecipients.Add(recipient);
    await _context.SaveChangesAsync();

    await SendRecipientConfirmationEmailAsync(recipient, channel);

    await _systemLog.LogAsync(LogLevel.Information, "Recipient",
        $"Recipient {recipient.Email} added to channel '{channel.Name}'",
        userId: userId, channelId: channelId);

    return recipient;
}

private string GenerateSecureToken()
{
    var bytes = new byte[32];
    using var rng = RandomNumberGenerator.Create();
    rng.GetBytes(bytes);
    return Convert.ToBase64String(bytes).Replace("+", "").Replace("/", "").Replace("=", "");
}
```

### Send Recipient Confirmation Email

```csharp
private async Task SendRecipientConfirmationEmailAsync(ChannelRecipient recipient, Channel channel)
{
    var confirmUrl = $"{_config["App:BaseUrl"]}/confirm-recipient?token={Uri.EscapeDataString(recipient.ConfirmationToken)}";

    var emailBody = $@"
<html>
<body>
    <h2>IdeaDump: Confirm Subscription</h2>
    <p>You have been added to the channel <strong>{channel.Name}</strong>.</p>
    <p>By confirming, you will receive elaborated idea reports via email.</p>
    <p><a href=""{confirmUrl}"">Click here to confirm your subscription</a></p>
    <p>This link expires in 7 days.</p>
    <p>If you did not expect this email, you can safely ignore it.</p>
</body>
</html>";

    await _emailService.SendEmailAsync(
        to: recipient.Email,
        subject: $"Confirm subscription to {channel.Name}",
        body: emailBody,
        smtpConfig: GetSystemSmtpConfig() // Use system SMTP for confirmation emails
    );
}
```

### Confirm Recipient

**Public Endpoint** (no authentication required):

```csharp
public async Task ConfirmRecipientAsync(string token)
{
    var recipient = await _context.ChannelRecipients
        .Include(r => r.Channel)
        .FirstOrDefaultAsync(r => r.ConfirmationToken == token);

    if (recipient == null)
        throw new InvalidOperationException("Invalid confirmation token");

    if (recipient.ConfirmationTokenExpiry < DateTime.UtcNow)
        throw new InvalidOperationException("Confirmation token has expired");

    if (recipient.IsConfirmed)
        throw new InvalidOperationException("Recipient already confirmed");

    recipient.IsConfirmed = true;
    recipient.ConfirmedAt = DateTime.UtcNow;
    recipient.ConfirmationToken = null; // Clear token after use
    recipient.ConfirmationTokenExpiry = null;

    await _context.SaveChangesAsync();

    await _systemLog.LogAsync(LogLevel.Information, "Recipient",
        $"Recipient {recipient.Email} confirmed for channel '{recipient.Channel.Name}'",
        channelId: recipient.ChannelId);
}
```

### List Recipients for Channel

```csharp
public async Task<List<RecipientSummary>> GetChannelRecipientsAsync(Guid channelId, Guid userId)
{
    // Verify ownership
    var channel = await _context.Channels
        .FirstOrDefaultAsync(c => c.Id == channelId && c.UserId == userId);

    if (channel == null)
        throw new UnauthorizedAccessException("Channel not found or access denied");

    return await _context.ChannelRecipients
        .Where(r => r.ChannelId == channelId)
        .OrderBy(r => r.Email)
        .Select(r => new RecipientSummary
        {
            Id = r.Id,
            Email = r.Email,
            DisplayName = r.DisplayName,
            IsConfirmed = r.IsConfirmed,
            IsUnsubscribed = r.IsUnsubscribed,
            ConfirmedAt = r.ConfirmedAt,
            CreatedAt = r.CreatedAt
        })
        .ToListAsync();
}
```

### Remove Recipient from Channel

**User Story**: As a user, I want to remove a recipient from my channel.

```csharp
public async Task RemoveRecipientAsync(Guid recipientId, Guid userId)
{
    var recipient = await _context.ChannelRecipients
        .Include(r => r.Channel)
        .FirstOrDefaultAsync(r => r.Id == recipientId);

    if (recipient == null)
        throw new InvalidOperationException("Recipient not found");

    // Verify channel ownership
    if (recipient.Channel.UserId != userId)
        throw new UnauthorizedAccessException("Access denied");

    var email = recipient.Email;
    var channelName = recipient.Channel.Name;

    _context.ChannelRecipients.Remove(recipient);
    await _context.SaveChangesAsync();

    await _systemLog.LogAsync(LogLevel.Information, "Recipient",
        $"Recipient {email} removed from channel '{channelName}'",
        userId: userId, channelId: recipient.ChannelId);
}
```

### Resend Confirmation Email

**User Story**: As a user, I want to resend a confirmation email to a recipient who hasn't confirmed yet.

```csharp
public async Task ResendConfirmationAsync(Guid recipientId, Guid userId)
{
    var recipient = await _context.ChannelRecipients
        .Include(r => r.Channel)
        .FirstOrDefaultAsync(r => r.Id == recipientId);

    if (recipient == null)
        throw new InvalidOperationException("Recipient not found");

    // Verify channel ownership
    if (recipient.Channel.UserId != userId)
        throw new UnauthorizedAccessException("Access denied");

    if (recipient.IsConfirmed)
        throw new InvalidOperationException("Recipient already confirmed");

    // Generate new token with extended expiry
    recipient.ConfirmationToken = GenerateSecureToken();
    recipient.ConfirmationTokenExpiry = DateTime.UtcNow.AddDays(7);
    await _context.SaveChangesAsync();

    await SendRecipientConfirmationEmailAsync(recipient, recipient.Channel);

    await _systemLog.LogAsync(LogLevel.Information, "Recipient",
        $"Confirmation email resent to {recipient.Email}",
        userId: userId, channelId: recipient.ChannelId);
}
```

### Unsubscribe Recipient

**Public Endpoint** (no authentication required):

```csharp
public async Task UnsubscribeRecipientAsync(string unsubscribeToken)
{
    var recipient = await _context.ChannelRecipients
        .Include(r => r.Channel)
        .FirstOrDefaultAsync(r => r.UnsubscribeToken == unsubscribeToken);

    if (recipient == null)
        throw new InvalidOperationException("Invalid unsubscribe token");

    if (recipient.IsUnsubscribed)
        throw new InvalidOperationException("Already unsubscribed");

    recipient.IsUnsubscribed = true;
    recipient.UnsubscribedAt = DateTime.UtcNow;

    await _context.SaveChangesAsync();

    await _systemLog.LogAsync(LogLevel.Information, "Recipient",
        $"Recipient {recipient.Email} unsubscribed from channel '{recipient.Channel.Name}'",
        channelId: recipient.ChannelId);

    // Optional: Send confirmation email
    await _emailService.SendEmailAsync(
        to: recipient.Email,
        subject: "Unsubscribed from IdeaDump",
        body: $"You have been unsubscribed from the channel '{recipient.Channel.Name}'.",
        smtpConfig: GetSystemSmtpConfig()
    );
}
```

## UI Components

### Channel List Page

```razor
@page "/channels"
@attribute [Authorize]
@inject IChannelService ChannelService
@inject NavigationManager Nav

<h1>My Channels</h1>

<button @onclick="CreateNewChannel">New Channel</button>

@if (channels == null)
{
    <p>Loading...</p>
}
else if (!channels.Any())
{
    <p>No channels yet. Create your first channel to get started!</p>
}
else
{
    <table>
        <thead>
            <tr>
                <th>Name</th>
                <th>Recipients</th>
                <th>Ideas</th>
                <th>Status</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var channel in channels)
            {
                <tr>
                    <td>@channel.Name</td>
                    <td>@channel.RecipientCount</td>
                    <td>@channel.IdeaCount</td>
                    <td>@(channel.IsActive ? "Active" : "Inactive")</td>
                    <td>
                        <a href="/channels/@channel.Id">View</a>
                        <button @onclick="() => DeleteChannel(channel.Id)">Delete</button>
                    </td>
                </tr>
            }
        </tbody>
    </table>
}

@code {
    private List<ChannelSummary>? channels;

    protected override async Task OnInitializedAsync()
    {
        var userId = await GetCurrentUserIdAsync();
        channels = await ChannelService.GetUserChannelsAsync(userId);
    }

    private void CreateNewChannel()
    {
        Nav.NavigateTo("/channels/new");
    }

    private async Task DeleteChannel(Guid channelId)
    {
        if (await JS.InvokeAsync<bool>("confirm", "Are you sure you want to delete this channel?"))
        {
            var userId = await GetCurrentUserIdAsync();
            await ChannelService.DeleteChannelAsync(channelId, userId);
            channels = await ChannelService.GetUserChannelsAsync(userId);
            StateHasChanged();
        }
    }
}
```

### Recipient Management Component

```razor
@inject IRecipientService RecipientService

<h3>Recipients</h3>

<EditForm Model="@newRecipient" OnValidSubmit="@AddRecipient">
    <InputText @bind-Value="newRecipient.Email" placeholder="Email" />
    <InputText @bind-Value="newRecipient.DisplayName" placeholder="Display Name (optional)" />
    <button type="submit">Add Recipient</button>
</EditForm>

<table>
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
                <td>@recipient.Email</td>
                <td>
                    @if (recipient.IsUnsubscribed) { <span>Unsubscribed</span> }
                    else if (recipient.IsConfirmed) { <span>Confirmed</span> }
                    else { <span>Pending</span> }
                </td>
                <td>@recipient.ConfirmedAt?.ToString("g")</td>
                <td>
                    @if (!recipient.IsConfirmed && !recipient.IsUnsubscribed)
                    {
                        <button @onclick="() => ResendConfirmation(recipient.Id)">Resend</button>
                    }
                    <button @onclick="() => RemoveRecipient(recipient.Id)">Remove</button>
                </td>
            </tr>
        }
    </tbody>
</table>

@code {
    [Parameter] public Guid ChannelId { get; set; }

    private List<RecipientSummary> recipients = new();
    private AddRecipientRequest newRecipient = new();

    protected override async Task OnInitializedAsync()
    {
        await LoadRecipients();
    }

    private async Task LoadRecipients()
    {
        var userId = await GetCurrentUserIdAsync();
        recipients = await RecipientService.GetChannelRecipientsAsync(ChannelId, userId);
    }

    private async Task AddRecipient()
    {
        var userId = await GetCurrentUserIdAsync();
        await RecipientService.AddRecipientAsync(ChannelId, newRecipient, userId);
        newRecipient = new();
        await LoadRecipients();
    }

    private async Task RemoveRecipient(Guid recipientId)
    {
        var userId = await GetCurrentUserIdAsync();
        await RecipientService.RemoveRecipientAsync(recipientId, userId);
        await LoadRecipients();
    }

    private async Task ResendConfirmation(Guid recipientId)
    {
        var userId = await GetCurrentUserIdAsync();
        await RecipientService.ResendConfirmationAsync(recipientId, userId);
        // Show success message
    }
}
```

## Validation Rules

### Channel Validation
- **Name**: Required, 3-100 characters, unique per user
- **SMTP Server**: Valid hostname or IP
- **SMTP Port**: 1-65535
- **From Email**: Valid email format
- **API Key**: Optional, if provided must be valid format

### Recipient Validation
- **Email**: Required, valid email format
- **Display Name**: Optional, max 100 characters
- **No duplicates**: Cannot add same email twice to same channel (unless unsubscribed)

## Error Handling

### Common Errors
- **Channel not found**: 404
- **Unauthorized access**: 403
- **Recipient already exists**: 409 Conflict
- **Invalid confirmation token**: 400 Bad Request
- **Expired confirmation token**: 410 Gone

### User-Friendly Messages
- "This channel doesn't exist or you don't have permission to access it."
- "This recipient is already in your channel."
- "The confirmation link has expired. Please request a new one."
- "Successfully added recipient. They will receive a confirmation email."

---

**Next Document**: [05-ai-integration-specification.md](05-ai-integration-specification.md)
