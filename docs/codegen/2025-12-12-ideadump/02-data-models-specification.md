# Data Models and Database Specification

## Database Strategy

### Storage Approach
- **SQLite** for all data storage (suitable for personal/family scale)
- **Single database file** for users, channels, recipients, and system metadata
- **Separate tables** for idea entries (can be partitioned by channel if needed)
- **Entity Framework Core** as ORM
- **ASP.NET Core Identity** for user management

### Why SQLite?
- Expected user base: 10-100 users
- Simple deployment (no separate database server)
- Version-controllable backup strategy
- Sufficient performance for expected load
- Easy migration to PostgreSQL/SQL Server if scale requires

## Entity Models

### User

Represents a registered user of the system. Extends ASP.NET Core Identity.

```csharp
public class ApplicationUser : IdentityUser<Guid>
{
    // Inherited from IdentityUser:
    // - Id (Guid)
    // - UserName (string)
    // - Email (string)
    // - EmailConfirmed (bool)
    // - PasswordHash (string)
    // - SecurityStamp (string)
    // - etc.

    /// <summary>
    /// Display name for the user (optional)
    /// </summary>
    public string? DisplayName { get; set; }

    /// <summary>
    /// User-wide API key for AI services (optional, overrides system key)
    /// </summary>
    public string? ApiKey { get; set; }

    /// <summary>
    /// Maximum ideas that can be elaborated per day (set by admin)
    /// </summary>
    public int DailyIdeaLimit { get; set; } = 10;

    /// <summary>
    /// Number of ideas elaborated today (resets at midnight UTC)
    /// </summary>
    public int IdeasElaboratedToday { get; set; } = 0;

    /// <summary>
    /// Last date when the daily counter was reset
    /// </summary>
    public DateTime LastLimitReset { get; set; } = DateTime.UtcNow.Date;

    /// <summary>
    /// Is this user an admin?
    /// </summary>
    public bool IsAdmin { get; set; } = false;

    /// <summary>
    /// Account creation timestamp
    /// </summary>
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    /// <summary>
    /// Last login timestamp
    /// </summary>
    public DateTime? LastLoginAt { get; set; }

    // Navigation properties
    public ICollection<Channel> Channels { get; set; } = new List<Channel>();
}
```

### Channel

Represents a channel (formerly "profile") that organizes ideas and recipients.

```csharp
public class Channel
{
    /// <summary>
    /// Unique identifier
    /// </summary>
    public Guid Id { get; set; } = Guid.NewGuid();

    /// <summary>
    /// User who owns this channel
    /// </summary>
    public Guid UserId { get; set; }
    public ApplicationUser User { get; set; } = null!;

    /// <summary>
    /// Display name for the channel
    /// </summary>
    public string Name { get; set; } = string.Empty;

    /// <summary>
    /// Optional description
    /// </summary>
    public string? Description { get; set; }

    /// <summary>
    /// Custom AI instructions for this channel (optional, uses system default if null)
    /// </summary>
    public string? AiInstructions { get; set; }

    /// <summary>
    /// Channel-specific API key (optional, overrides user key and system key)
    /// </summary>
    public string? ApiKey { get; set; }

    /// <summary>
    /// SMTP server for sending emails from this channel
    /// </summary>
    public string SmtpServer { get; set; } = string.Empty;

    /// <summary>
    /// SMTP port (typically 587 for TLS, 465 for SSL)
    /// </summary>
    public int SmtpPort { get; set; } = 587;

    /// <summary>
    /// Use SSL/TLS for SMTP connection
    /// </summary>
    public bool SmtpUseSsl { get; set; } = true;

    /// <summary>
    /// SMTP username (email address)
    /// </summary>
    public string SmtpUsername { get; set; } = string.Empty;

    /// <summary>
    /// SMTP password (encrypted at rest)
    /// </summary>
    public string SmtpPassword { get; set; } = string.Empty;

    /// <summary>
    /// From email address for sent emails
    /// </summary>
    public string FromEmail { get; set; } = string.Empty;

    /// <summary>
    /// From name for sent emails
    /// </summary>
    public string FromName { get; set; } = string.Empty;

    /// <summary>
    /// Is this channel active?
    /// </summary>
    public bool IsActive { get; set; } = true;

    /// <summary>
    /// Creation timestamp
    /// </summary>
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    /// <summary>
    /// Last update timestamp
    /// </summary>
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;

    // Navigation properties
    public ICollection<ChannelRecipient> Recipients { get; set; } = new List<ChannelRecipient>();
    public ICollection<IdeaEntry> Ideas { get; set; } = new List<IdeaEntry>();
}
```

### ChannelRecipient

Represents a recipient subscribed to a channel. Requires explicit consent.

```csharp
public class ChannelRecipient
{
    /// <summary>
    /// Unique identifier
    /// </summary>
    public Guid Id { get; set; } = Guid.NewGuid();

    /// <summary>
    /// Channel this recipient belongs to
    /// </summary>
    public Guid ChannelId { get; set; }
    public Channel Channel { get; set; } = null!;

    /// <summary>
    /// Recipient email address
    /// </summary>
    public string Email { get; set; } = string.Empty;

    /// <summary>
    /// Optional display name for recipient
    /// </summary>
    public string? DisplayName { get; set; }

    /// <summary>
    /// Has the recipient confirmed subscription to this channel?
    /// </summary>
    public bool IsConfirmed { get; set; } = false;

    /// <summary>
    /// Confirmation token (sent via email)
    /// </summary>
    public string? ConfirmationToken { get; set; }

    /// <summary>
    /// When confirmation token expires
    /// </summary>
    public DateTime? ConfirmationTokenExpiry { get; set; }

    /// <summary>
    /// When recipient confirmed subscription
    /// </summary>
    public DateTime? ConfirmedAt { get; set; }

    /// <summary>
    /// When recipient was added to channel
    /// </summary>
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    /// <summary>
    /// Unsubscribe token for one-click opt-out
    /// </summary>
    public string UnsubscribeToken { get; set; } = Guid.NewGuid().ToString();

    /// <summary>
    /// Has recipient unsubscribed?
    /// </summary>
    public bool IsUnsubscribed { get; set; } = false;

    /// <summary>
    /// When recipient unsubscribed
    /// </summary>
    public DateTime? UnsubscribedAt { get; set; }
}
```

### IdeaEntry

Represents a single idea submission with its elaboration and email status.

```csharp
public class IdeaEntry
{
    /// <summary>
    /// Unique identifier
    /// </summary>
    public Guid Id { get; set; } = Guid.NewGuid();

    /// <summary>
    /// Channel this idea belongs to
    /// </summary>
    public Guid ChannelId { get; set; }
    public Channel Channel { get; set; } = null!;

    /// <summary>
    /// User who submitted this idea
    /// </summary>
    public Guid UserId { get; set; }
    public ApplicationUser User { get; set; } = null!;

    /// <summary>
    /// Raw idea text submitted by user (immutable)
    /// </summary>
    public string RawText { get; set; } = string.Empty;

    /// <summary>
    /// AI-generated title for this idea
    /// </summary>
    public string? GeneratedTitle { get; set; }

    /// <summary>
    /// AI-elaborated content (research, analysis, tasks, etc.)
    /// </summary>
    public string? ElaboratedContent { get; set; }

    /// <summary>
    /// Email subject line (AI-generated)
    /// </summary>
    public string? EmailSubject { get; set; }

    /// <summary>
    /// Email body (formatted from elaborated content)
    /// </summary>
    public string? EmailBody { get; set; }

    /// <summary>
    /// Current processing status
    /// </summary>
    public IdeaEntryStatus Status { get; set; } = IdeaEntryStatus.New;

    /// <summary>
    /// When idea was submitted
    /// </summary>
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    /// <summary>
    /// When elaboration started
    /// </summary>
    public DateTime? ElaborationStartedAt { get; set; }

    /// <summary>
    /// When elaboration completed
    /// </summary>
    public DateTime? ElaboratedAt { get; set; }

    /// <summary>
    /// When email was successfully sent
    /// </summary>
    public DateTime? EmailSentAt { get; set; }

    /// <summary>
    /// If failed, when will next retry occur
    /// </summary>
    public DateTime? NextRetryAt { get; set; }

    /// <summary>
    /// Number of retry attempts so far
    /// </summary>
    public int RetryCount { get; set; } = 0;

    /// <summary>
    /// Maximum retries before giving up
    /// </summary>
    public int MaxRetries { get; set; } = 5;

    /// <summary>
    /// Error message if failed
    /// </summary>
    public string? ErrorMessage { get; set; }

    /// <summary>
    /// Which API key tier was used (System/User/Channel)
    /// </summary>
    public string? ApiKeySource { get; set; }

    // Navigation properties
    public ICollection<IdeaProcessingLog> ProcessingLogs { get; set; } = new List<IdeaProcessingLog>();
}

public enum IdeaEntryStatus
{
    New = 0,           // Just created, not yet picked up for processing
    Elaborating = 1,   // AI elaboration in progress
    Elaborated = 2,    // Elaboration complete, ready to send
    Sending = 3,       // Email send in progress
    Sent = 4,          // Successfully sent to all recipients
    Failed = 5,        // Failed (AI or email), will retry
    Abandoned = 6      // Max retries exceeded, given up
}
```

### IdeaProcessingLog

Detailed log entries for each idea's processing lifecycle.

```csharp
public class IdeaProcessingLog
{
    /// <summary>
    /// Unique identifier
    /// </summary>
    public Guid Id { get; set; } = Guid.NewGuid();

    /// <summary>
    /// Associated idea entry
    /// </summary>
    public Guid IdeaEntryId { get; set; }
    public IdeaEntry IdeaEntry { get; set; } = null!;

    /// <summary>
    /// Log timestamp
    /// </summary>
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;

    /// <summary>
    /// Log level
    /// </summary>
    public LogLevel Level { get; set; }

    /// <summary>
    /// Event type (e.g., "ElaborationStarted", "EmailSent", "RetryScheduled")
    /// </summary>
    public string EventType { get; set; } = string.Empty;

    /// <summary>
    /// Detailed log message
    /// </summary>
    public string Message { get; set; } = string.Empty;

    /// <summary>
    /// Additional structured data (JSON)
    /// </summary>
    public string? MetadataJson { get; set; }
}

public enum LogLevel
{
    Debug = 0,
    Information = 1,
    Warning = 2,
    Error = 3,
    Critical = 4
}
```

### SystemLog

System-wide logging for admin monitoring.

```csharp
public class SystemLog
{
    /// <summary>
    /// Unique identifier
    /// </summary>
    public Guid Id { get; set; } = Guid.NewGuid();

    /// <summary>
    /// Log timestamp
    /// </summary>
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;

    /// <summary>
    /// Log level
    /// </summary>
    public LogLevel Level { get; set; }

    /// <summary>
    /// Log category (e.g., "AI", "Email", "Authentication", "System")
    /// </summary>
    public string Category { get; set; } = string.Empty;

    /// <summary>
    /// Log message
    /// </summary>
    public string Message { get; set; } = string.Empty;

    /// <summary>
    /// Exception details if applicable
    /// </summary>
    public string? ExceptionDetails { get; set; }

    /// <summary>
    /// User ID if action was user-specific
    /// </summary>
    public Guid? UserId { get; set; }

    /// <summary>
    /// Channel ID if action was channel-specific
    /// </summary>
    public Guid? ChannelId { get; set; }

    /// <summary>
    /// Idea entry ID if action was idea-specific
    /// </summary>
    public Guid? IdeaEntryId { get; set; }

    /// <summary>
    /// Additional structured data (JSON)
    /// </summary>
    public string? MetadataJson { get; set; }
}
```

### SystemConfiguration

System-wide configuration stored in database.

```csharp
public class SystemConfiguration
{
    /// <summary>
    /// Configuration key
    /// </summary>
    public string Key { get; set; } = string.Empty;

    /// <summary>
    /// Configuration value
    /// </summary>
    public string Value { get; set; } = string.Empty;

    /// <summary>
    /// Description of what this configuration does
    /// </summary>
    public string? Description { get; set; }

    /// <summary>
    /// Last update timestamp
    /// </summary>
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
}
```

**Common Configuration Keys:**
- `System.ApiKey` - System-wide AI API key
- `System.DefaultAiInstructions` - Default AI elaboration instructions
- `System.DefaultDailyLimit` - Default daily idea limit for new users
- `System.SmtpServer` - System default SMTP server
- `System.SmtpPort` - System default SMTP port
- `System.SmtpUsername` - System default SMTP username
- `System.SmtpPassword` - System default SMTP password (encrypted)
- `System.AdminNotificationEmail` - Email for critical system alerts
- `System.NotificationThrottleMinutes` - Minimum minutes between admin notifications

## Database Schema Diagram

```
┌─────────────────────┐
│  ApplicationUser    │
├─────────────────────┤
│ Id (PK)             │──┐
│ Email               │  │
│ ApiKey              │  │
│ DailyIdeaLimit      │  │
│ IsAdmin             │  │
└─────────────────────┘  │
                         │
                         │ 1:N
                         │
┌─────────────────────┐  │
│      Channel        │◄─┘
├─────────────────────┤
│ Id (PK)             │──┐──┐
│ UserId (FK)         │  │  │
│ Name                │  │  │
│ AiInstructions      │  │  │
│ ApiKey              │  │  │
│ SmtpServer          │  │  │
└─────────────────────┘  │  │
                         │  │
         ┌───────────────┘  │
         │ 1:N              │ 1:N
         │                  │
┌─────────────────────┐    │
│ ChannelRecipient    │    │
├─────────────────────┤    │
│ Id (PK)             │    │
│ ChannelId (FK)      │    │
│ Email               │    │
│ IsConfirmed         │    │
└─────────────────────┘    │
                           │
┌─────────────────────┐    │
│    IdeaEntry        │◄───┘
├─────────────────────┤
│ Id (PK)             │──┐
│ ChannelId (FK)      │  │
│ UserId (FK)         │  │
│ RawText             │  │
│ ElaboratedContent   │  │
│ Status              │  │
└─────────────────────┘  │
                         │ 1:N
                         │
┌─────────────────────┐  │
│ IdeaProcessingLog   │◄─┘
├─────────────────────┤
│ Id (PK)             │
│ IdeaEntryId (FK)    │
│ Timestamp           │
│ EventType           │
└─────────────────────┘

┌─────────────────────┐
│    SystemLog        │
├─────────────────────┤
│ Id (PK)             │
│ Timestamp           │
│ Level               │
│ Category            │
│ Message             │
└─────────────────────┘

┌─────────────────────┐
│ SystemConfiguration │
├─────────────────────┤
│ Key (PK)            │
│ Value               │
│ Description         │
└─────────────────────┘
```

## Indexes for Performance

```sql
-- User lookups
CREATE INDEX IX_Users_Email ON AspNetUsers(Email);

-- Channel ownership
CREATE INDEX IX_Channels_UserId ON Channels(UserId);
CREATE INDEX IX_Channels_IsActive ON Channels(IsActive);

-- Recipient lookups
CREATE INDEX IX_ChannelRecipients_ChannelId ON ChannelRecipients(ChannelId);
CREATE INDEX IX_ChannelRecipients_Email ON ChannelRecipients(Email);
CREATE INDEX IX_ChannelRecipients_IsConfirmed ON ChannelRecipients(IsConfirmed);

-- Idea entry queries
CREATE INDEX IX_IdeaEntries_ChannelId ON IdeaEntries(ChannelId);
CREATE INDEX IX_IdeaEntries_UserId ON IdeaEntries(UserId);
CREATE INDEX IX_IdeaEntries_Status ON IdeaEntries(Status);
CREATE INDEX IX_IdeaEntries_CreatedAt ON IdeaEntries(CreatedAt);
CREATE INDEX IX_IdeaEntries_NextRetryAt ON IdeaEntries(NextRetryAt) WHERE NextRetryAt IS NOT NULL;

-- Processing logs
CREATE INDEX IX_IdeaProcessingLogs_IdeaEntryId ON IdeaProcessingLogs(IdeaEntryId);
CREATE INDEX IX_IdeaProcessingLogs_Timestamp ON IdeaProcessingLogs(Timestamp);

-- System logs
CREATE INDEX IX_SystemLogs_Timestamp ON SystemLogs(Timestamp);
CREATE INDEX IX_SystemLogs_Level ON SystemLogs(Level);
CREATE INDEX IX_SystemLogs_Category ON SystemLogs(Category);
```

## Data Validation Rules

### User
- Email: Required, valid email format, unique
- Password: Minimum 8 characters, requires uppercase, lowercase, digit, special character
- DailyIdeaLimit: Minimum 0, maximum 1000

### Channel
- Name: Required, 3-100 characters
- SmtpServer: Required if sending emails
- SmtpPort: 1-65535
- FromEmail: Valid email format

### ChannelRecipient
- Email: Required, valid email format
- ConfirmationToken: 32-character random string
- ConfirmationTokenExpiry: 7 days from creation

### IdeaEntry
- RawText: Required, 1-10,000 characters
- ElaboratedContent: Maximum 50,000 characters
- Status: Must be valid enum value
- RetryCount: 0-MaxRetries

## Encryption & Security

### Encrypted Fields
- `Channel.SmtpPassword` - Encrypted at rest using Data Protection API
- `ApplicationUser.ApiKey` - Encrypted at rest
- `Channel.ApiKey` - Encrypted at rest
- `SystemConfiguration.Value` - Encrypted for sensitive keys (passwords, API keys)

### Hashed Fields
- User passwords - Hashed via ASP.NET Core Identity (PBKDF2)

### Token Generation
- Confirmation tokens: 32-byte cryptographically random strings
- Unsubscribe tokens: GUID format

## Migration Strategy

Initial migration will create all tables with proper foreign keys and indexes. Subsequent migrations will be additive.

```bash
dotnet ef migrations add InitialCreate --project src/IdeaDumpCore
dotnet ef database update --project src/IdeaDump
```

---

**Next Document**: [03-authentication-and-authorization.md](03-authentication-and-authorization.md)
