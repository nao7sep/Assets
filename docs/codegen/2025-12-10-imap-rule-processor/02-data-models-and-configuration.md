# Data Models and Configuration

## Configuration File Structure

### Primary Configuration File: `appsettings.json`

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
  "accounts": [
    {
      "name": "work",
      "enabled": true,
      "server": "imap.gmail.com",
      "port": 993,
      "useSsl": true,
      "username": "user@work.com",
      "authType": "oauth2",
      "clientId": "${GMAIL_CLIENT_ID}",
      "clientSecret": "${GMAIL_CLIENT_SECRET}",
      "refreshToken": "${WORK_REFRESH_TOKEN}",
      "checkFolders": ["INBOX"],
      "rateLimits": {
        "maxMessagesPerSession": 100,
        "delayBetweenMessagesMs": 100,
        "maxActionsPerMinute": 50,
        "connectionTimeoutSeconds": 30,
        "retryDelaySeconds": 60,
        "maxRetries": 3
      },
      "backup": {
        "mode": "MovedOrDeleted",
        "folder": "Backups",
        "retentionDays": 30,
        "compress": false
      }
    },
    {
      "name": "personal",
      "enabled": true,
      "server": "imap.example.com",
      "port": 993,
      "useSsl": true,
      "username": "user@example.com",
      "authType": "password",
      "password": "${PERSONAL_PASSWORD}",
      "checkFolders": ["INBOX"],
      "rateLimits": {
        "maxMessagesPerSession": 50,
        "delayBetweenMessagesMs": 200
      },
      "backup": {
        "mode": "None"
      }
    }
  ],
  "rules": [
    {
      "name": "Newsletter Organization",
      "enabled": true,
      "applyToAccounts": ["work", "personal"],
      "priority": 10,
      "conditions": {
        "matchType": "all",
        "from": { "contains": "newsletter", "caseInsensitive": true },
        "subject": { "contains": "weekly", "caseInsensitive": true }
      },
      "actions": [
        { "type": "moveToFolder", "folder": "Newsletters" },
        { "type": "markAsRead" }
      ]
    },
    {
      "name": "Important Emails",
      "enabled": true,
      "applyToAccounts": ["ALL"],
      "priority": 1,
      "conditions": {
        "matchType": "any",
        "from": { "contains": "boss@company.com" },
        "subject": { "startsWith": "[URGENT]" }
      },
      "actions": [
        { "type": "addFlag", "flag": "Flagged" },
        { "type": "moveToFolder", "folder": "Important" }
      ]
    },
    {
      "name": "Cross-Account Newsletter Move",
      "enabled": true,
      "applyToAccounts": ["work"],
      "priority": 5,
      "conditions": {
        "matchType": "all",
        "from": { "contains": "marketing" }
      },
      "actions": [
        {
          "type": "moveToAccount",
          "targetAccountName": "personal",
          "targetFolder": "INBOX",
          "markAsProcessed": true
        }
      ]
    },
    {
      "name": "Organize Newsletters in Personal",
      "enabled": true,
      "applyToAccounts": ["personal"],
      "priority": 5,
      "conditions": {
        "matchType": "all",
        "from": { "contains": "marketing" }
      },
      "actions": [
        { "type": "moveToFolder", "folder": "Marketing" }
      ]
    }
  ]
}
```

## Data Models

### Account Configuration

```csharp
public class AccountConfig
{
    /// <summary>
    /// Unique identifier for the account. Used in rule targeting.
    /// Reserved names: "ALL", "NONE", "ANY" (case-sensitive)
    /// </summary>
    public string Name { get; set; } = "";

    /// <summary>
    /// Whether this account should be processed
    /// </summary>
    public bool Enabled { get; set; } = true;

    // Connection settings
    public string Server { get; set; } = "";
    public int Port { get; set; } = 993;
    public bool UseSsl { get; set; } = true;
    public string Username { get; set; } = "";

    /// <summary>
    /// Authentication type: "password" | "oauth2"
    /// </summary>
    public string AuthType { get; set; } = "password";

    // Password authentication
    public string? Password { get; set; }

    // OAuth2 authentication
    public string? ClientId { get; set; }
    public string? ClientSecret { get; set; }
    public string? RefreshToken { get; set; }

    // Runtime OAuth2 state (not persisted in config)
    [JsonIgnore]
    public string? AccessToken { get; set; }

    [JsonIgnore]
    public DateTime? TokenExpiry { get; set; }

    /// <summary>
    /// Folders to check for new messages. Defaults to ["INBOX"]
    /// </summary>
    public string[] CheckFolders { get; set; } = new[] { "INBOX" };

    /// <summary>
    /// Rate limiting configuration
    /// </summary>
    public RateLimitConfig RateLimits { get; set; } = new();

    /// <summary>
    /// Backup configuration
    /// </summary>
    public BackupConfig Backup { get; set; } = new();
}

public class RateLimitConfig
{
    /// <summary>
    /// Maximum number of messages to process per session
    /// </summary>
    public int MaxMessagesPerSession { get; set; } = 100;

    /// <summary>
    /// Delay in milliseconds between processing messages
    /// </summary>
    public int DelayBetweenMessagesMs { get; set; } = 100;

    /// <summary>
    /// Maximum number of actions per minute
    /// </summary>
    public int MaxActionsPerMinute { get; set; } = 50;

    /// <summary>
    /// Connection timeout in seconds
    /// </summary>
    public int ConnectionTimeoutSeconds { get; set; } = 30;

    /// <summary>
    /// Delay before retrying after an error
    /// </summary>
    public int RetryDelaySeconds { get; set; } = 60;

    /// <summary>
    /// Maximum number of retry attempts
    /// </summary>
    public int MaxRetries { get; set; } = 3;
}

public class BackupConfig
{
    /// <summary>
    /// Backup mode
    /// </summary>
    public BackupMode Mode { get; set; } = BackupMode.MovedOrDeleted;

    /// <summary>
    /// Folder name for backups
    /// </summary>
    public string Folder { get; set; } = "Backups";

    /// <summary>
    /// Days to keep backups before cleanup
    /// </summary>
    public int RetentionDays { get; set; } = 30;

    /// <summary>
    /// Whether to compress backups
    /// </summary>
    public bool Compress { get; set; } = false;
}

public enum BackupMode
{
    /// <summary>No backups created</summary>
    None,

    /// <summary>Backup every message processed</summary>
    All,

    /// <summary>Backup only when message is moved or deleted</summary>
    MovedOrDeleted,

    /// <summary>Backup only when message is deleted</summary>
    DeletedOnly
}
```

### Rule Configuration

```csharp
public class RuleConfig
{
    /// <summary>
    /// Unique, descriptive name for the rule
    /// </summary>
    public string Name { get; set; } = "";

    /// <summary>
    /// Whether this rule is active
    /// </summary>
    public bool Enabled { get; set; } = true;

    /// <summary>
    /// Account names this rule applies to.
    /// Special values: "ALL" (all accounts), "NONE" (no accounts)
    /// Default: ["ALL"]
    /// </summary>
    public string[] ApplyToAccounts { get; set; } = new[] { "ALL" };

    /// <summary>
    /// Rule priority (lower number = higher priority)
    /// Rules are evaluated in priority order
    /// </summary>
    public int Priority { get; set; } = 100;

    /// <summary>
    /// Conditions that must be met for this rule to apply
    /// </summary>
    public RuleCondition Conditions { get; set; } = new();

    /// <summary>
    /// Actions to execute when conditions are met
    /// </summary>
    public RuleAction[] Actions { get; set; } = Array.Empty<RuleAction>();

    /// <summary>
    /// Whether to stop processing other rules if this one matches
    /// </summary>
    public bool StopProcessing { get; set; } = false;
}

public class RuleCondition
{
    /// <summary>
    /// How to combine conditions: "all" (AND) or "any" (OR)
    /// </summary>
    public string MatchType { get; set; } = "all"; // "all" | "any"

    // Email header conditions
    /// <summary>From, Sender, or Reply-To addresses</summary>
    public TextCondition? From { get; set; }

    /// <summary>To, Cc, or Bcc addresses</summary>
    public TextCondition? To { get; set; }

    /// <summary>Subject line</summary>
    public TextCondition? Subject { get; set; }

    /// <summary>Message body (plain text or HTML)</summary>
    public TextCondition? Body { get; set; }

    // Meta conditions
    /// <summary>Whether message has attachments</summary>
    public bool? HasAttachment { get; set; }

    /// <summary>Whether message is flagged</summary>
    public bool? IsFlagged { get; set; }

    /// <summary>Whether message is unread</summary>
    public bool? IsUnread { get; set; }

    /// <summary>Date/time conditions</summary>
    public DateCondition? ReceivedDate { get; set; }

    /// <summary>Message size greater than (in KB)</summary>
    public int? SizeGreaterThanKB { get; set; }

    /// <summary>Message size less than (in KB)</summary>
    public int? SizeLessThanKB { get; set; }
}

public class TextCondition
{
    /// <summary>Text contains this substring</summary>
    public string? Contains { get; set; }

    /// <summary>Text starts with this string</summary>
    public string? StartsWith { get; set; }

    /// <summary>Text ends with this string</summary>
    public string? EndsWith { get; set; }

    /// <summary>Text exactly equals this string</summary>
    public string? Equals { get; set; }

    /// <summary>Text matches this regex pattern</summary>
    public string? Regex { get; set; }

    /// <summary>Case-insensitive matching (default: true)</summary>
    public bool CaseInsensitive { get; set; } = true;
}

public class DateCondition
{
    /// <summary>Date is before this ISO 8601 date</summary>
    public string? Before { get; set; }

    /// <summary>Date is after this ISO 8601 date</summary>
    public string? After { get; set; }

    /// <summary>Date is within last N days</summary>
    public int? WithinLastDays { get; set; }

    /// <summary>Date is older than N days</summary>
    public int? OlderThanDays { get; set; }
}

public class RuleAction
{
    /// <summary>
    /// Action type: "moveToFolder", "moveToAccount", "delete",
    /// "markAsRead", "markAsUnread", "addFlag", "removeFlag"
    /// </summary>
    public string Type { get; set; } = "";

    // moveToFolder
    public string? Folder { get; set; }

    // moveToAccount
    public string? TargetAccountName { get; set; }
    public string? TargetFolder { get; set; }
    public bool MarkAsProcessed { get; set; } = true;

    // addFlag, removeFlag
    /// <summary>
    /// Flag name: "Seen", "Flagged", "Answered", "Draft", "Deleted"
    /// </summary>
    public string? Flag { get; set; }
}
```

### Application Configuration

```csharp
public class AppConfig
{
    /// <summary>
    /// Logging configuration
    /// </summary>
    public LoggingConfig Logging { get; set; } = new();

    /// <summary>
    /// Storage paths
    /// </summary>
    public StorageConfig Storage { get; set; } = new();

    /// <summary>
    /// Email account configurations
    /// </summary>
    public AccountConfig[] Accounts { get; set; } = Array.Empty<AccountConfig>();

    /// <summary>
    /// Rule configurations
    /// </summary>
    public RuleConfig[] Rules { get; set; } = Array.Empty<RuleConfig>();
}

public class LoggingConfig
{
    /// <summary>
    /// Minimum log level: "Verbose", "Debug", "Information", "Warning", "Error", "Fatal"
    /// </summary>
    public string Level { get; set; } = "Information";

    /// <summary>
    /// Path template for log files (supports rolling)
    /// </summary>
    public string FilePath { get; set; } = "logs/app-.log";

    /// <summary>
    /// Path for SQLite audit database
    /// </summary>
    public string SqlitePath { get; set; } = "logs/audit.db";
}

public class StorageConfig
{
    /// <summary>
    /// Path for UID tracking database
    /// </summary>
    public string UidDatabasePath { get; set; } = "data/processed.db";

    /// <summary>
    /// Root path for email backups
    /// </summary>
    public string BackupPath { get; set; } = "backups";
}
```

## Runtime Models

### Processing Context

```csharp
public class ProcessingContext
{
    public AccountConfig Account { get; set; } = null!;
    public IMailFolder Folder { get; set; } = null!;
    public int ChainDepth { get; set; }
    public bool DryRun { get; set; }
    public CancellationToken CancellationToken { get; set; }

    public const int MaxChainDepth = 3;
}
```

### Rule Execution Result

```csharp
public class RuleExecutionResult
{
    public bool RuleMatched { get; set; }
    public string? RuleName { get; set; }
    public List<ActionResult> ActionResults { get; set; } = new();
    public bool StopProcessing { get; set; }
}

public class ActionResult
{
    public string ActionType { get; set; } = "";
    public bool Success { get; set; }
    public string? ErrorMessage { get; set; }
    public Dictionary<string, object> Details { get; set; } = new();
    public bool DryRun { get; set; }
}
```

### OAuth2 Token Response

```csharp
public class TokenResponse
{
    [JsonPropertyName("access_token")]
    public string AccessToken { get; set; } = "";

    [JsonPropertyName("refresh_token")]
    public string? RefreshToken { get; set; }

    [JsonPropertyName("expires_in")]
    public int ExpiresIn { get; set; }

    [JsonPropertyName("token_type")]
    public string TokenType { get; set; } = "Bearer";
}
```

## Configuration Management

### Environment Variable Substitution

Configuration values can reference environment variables using `${VAR_NAME}` syntax:

```json
{
  "accounts": [
    {
      "name": "work",
      "password": "${WORK_EMAIL_PASSWORD}",
      "clientId": "${GMAIL_CLIENT_ID}"
    }
  ]
}
```

**Implementation**:
```csharp
public class ConfigManager
{
    public async Task<AppConfig> LoadAsync(string path)
    {
        var json = await File.ReadAllTextAsync(path);

        // Replace environment variables
        json = Regex.Replace(json, @"\$\{([^}]+)\}", match =>
        {
            var varName = match.Groups[1].Value;
            return Environment.GetEnvironmentVariable(varName) ?? match.Value;
        });

        var config = JsonSerializer.Deserialize<AppConfig>(json, _options);
        return config ?? throw new InvalidOperationException("Failed to deserialize config");
    }
}
```

### Configuration File Locations

**Priority** (first found is used):
1. Command-line argument: `--config path/to/config.json`
2. Environment variable: `IMAP_PROCESSOR_CONFIG`
3. Current directory: `./appsettings.json`
4. User home: `~/.config/imap-processor/appsettings.json`

### Secret Management

**Option 1: Separate secrets file** (recommended)

```
appsettings.json          # Committable, no secrets
appsettings.secrets.json  # Gitignored, contains secrets
```

Load both and merge:
```csharp
var config = await LoadAsync("appsettings.json");
var secrets = await LoadAsync("appsettings.secrets.json");
MergeConfig(config, secrets); // Secrets override config
```

**Option 2: Environment variables** (best for CI/CD)

```bash
export WORK_PASSWORD="mypassword"
export GMAIL_CLIENT_ID="..."
export GMAIL_CLIENT_SECRET="..."
```

**Option 3: Encrypted fields**

```csharp
public class AccountConfig
{
    private string? _password;

    public string? Password
    {
        get => _password == null ? null : Decrypt(_password);
        set => _password = value == null ? null : Encrypt(value);
    }

    private string Encrypt(string value) =>
        Convert.ToBase64String(ProtectedData.Protect(
            Encoding.UTF8.GetBytes(value),
            null,
            DataProtectionScope.CurrentUser));

    private string Decrypt(string encrypted) =>
        Encoding.UTF8.GetString(ProtectedData.Unprotect(
            Convert.FromBase64String(encrypted),
            null,
            DataProtectionScope.CurrentUser));
}
```

## Validation

### Configuration Validation

```csharp
public class ConfigValidator
{
    public List<ValidationError> Validate(AppConfig config)
    {
        var errors = new List<ValidationError>();

        // Validate accounts
        var accountNames = new HashSet<string>();
        foreach (var account in config.Accounts)
        {
            if (string.IsNullOrWhiteSpace(account.Name))
                errors.Add(new("Account.Name", "Name is required"));

            if (!accountNames.Add(account.Name))
                errors.Add(new("Account.Name", $"Duplicate account name: {account.Name}"));

            if (IsReservedName(account.Name))
                errors.Add(new("Account.Name",
                    $"'{account.Name}' is a reserved name (ALL, NONE, ANY)"));

            if (string.IsNullOrWhiteSpace(account.Server))
                errors.Add(new("Account.Server", "Server is required"));

            if (account.AuthType == "password" && string.IsNullOrWhiteSpace(account.Password))
                errors.Add(new("Account.Password", "Password required for password auth"));

            if (account.AuthType == "oauth2" && string.IsNullOrWhiteSpace(account.ClientId))
                errors.Add(new("Account.ClientId", "ClientId required for OAuth2"));
        }

        // Validate rules
        foreach (var rule in config.Rules)
        {
            if (string.IsNullOrWhiteSpace(rule.Name))
                errors.Add(new("Rule.Name", "Name is required"));

            foreach (var accountName in rule.ApplyToAccounts)
            {
                if (accountName != "ALL" && accountName != "NONE" &&
                    !accountNames.Contains(accountName))
                {
                    errors.Add(new("Rule.ApplyToAccounts",
                        $"Unknown account: {accountName}"));
                }
            }

            if (rule.Actions.Length == 0)
                errors.Add(new("Rule.Actions", "At least one action required"));
        }

        return errors;
    }

    private bool IsReservedName(string name) =>
        name is "ALL" or "NONE" or "ANY";
}

public record ValidationError(string Field, string Message);
```

## JSON Serialization Options

```csharp
var jsonOptions = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = true,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    Converters = { new JsonStringEnumConverter() }
};
```

## Example Configurations

### Minimal Configuration

```json
{
  "accounts": [
    {
      "name": "personal",
      "server": "imap.gmail.com",
      "port": 993,
      "username": "user@gmail.com",
      "authType": "oauth2",
      "clientId": "${GMAIL_CLIENT_ID}",
      "clientSecret": "${GMAIL_CLIENT_SECRET}",
      "refreshToken": "${GMAIL_REFRESH_TOKEN}"
    }
  ],
  "rules": [
    {
      "name": "Archive old emails",
      "conditions": {
        "receivedDate": { "olderThanDays": 365 }
      },
      "actions": [
        { "type": "moveToFolder", "folder": "Archive" }
      ]
    }
  ]
}
```

### Complex Multi-Account Configuration

See the primary configuration example at the top of this document for a comprehensive example with multiple accounts, OAuth2, cross-account moves, and various rule types.
