# Architecture and Design

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│             ImapRuleProcessor (Console App)             │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   CLI        │  │ Config       │  │ DI Container │ │
│  │   Commands   │  │ Manager      │  │              │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│          ImapRuleProcessor.Core (Library)               │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ IMAP Client  │  │ Rule Engine  │  │ Auth         │ │
│  │ Wrapper      │  │              │  │ Providers    │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ UID Tracker  │  │ Audit Logger │  │ Action       │ │
│  │ (SQLite)     │  │ (SQLite)     │  │ Executor     │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│             External Dependencies                       │
│                                                         │
│  • MailKit (IMAP, OAuth2)                              │
│  • Serilog (Logging)                                   │
│  • Microsoft.Data.Sqlite (Storage)                     │
│  • System.Text.Json (Configuration)                    │
└─────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Console Application Layer

**Purpose**: Entry point, CLI commands, dependency injection setup

**Responsibilities**:
- Parse command-line arguments
- Load configuration from JSON
- Set up dependency injection container
- Provide CLI commands (setup, run, test, dry-run)
- Handle graceful shutdown

**Key Classes**:
```csharp
// Program.cs - Entry point
public class Program
{
    public static async Task<int> Main(string[] args)
    {
        // Setup logging, DI, configuration
        // Parse commands and execute
    }
}

// Commands/SetupCommand.cs
public class SetupCommand
{
    public async Task ExecuteAsync(string provider, string accountName);
}

// Commands/RunCommand.cs
public class RunCommand
{
    public async Task ExecuteAsync(bool daemon, bool dryRun);
}

// ConfigManager.cs
public class ConfigManager
{
    public Task<AppConfig> LoadAsync(string path);
    public Task SaveAsync(AppConfig config, string path);
}
```

### 2. Core Library Components

#### A. IMAP Client Wrapper

**Purpose**: Abstract MailKit's ImapClient with a testable interface

**Responsibilities**:
- Connect/disconnect from IMAP servers
- Authenticate (password or OAuth2)
- List folders
- Search and fetch messages
- Move/copy/delete messages
- Apply flags

**Key Interfaces**:
```csharp
public interface IImapClient : IDisposable
{
    Task ConnectAsync(string host, int port, bool useSsl, CancellationToken ct);
    Task AuthenticateAsync(IAuthenticator authenticator, CancellationToken ct);
    Task<IReadOnlyList<IMailFolder>> GetFoldersAsync(CancellationToken ct);
    Task<IMailFolder> GetFolderAsync(string path, CancellationToken ct);
    Task DisconnectAsync(CancellationToken ct);
}

public interface IMailFolder
{
    Task<IList<uint>> SearchAsync(SearchQuery query, CancellationToken ct);
    Task<MimeMessage> GetMessageAsync(uint uid, CancellationToken ct);
    Task MoveToAsync(uint uid, IMailFolder destination, CancellationToken ct);
    Task AddFlagsAsync(uint uid, MessageFlags flags, CancellationToken ct);
    Task DeleteAsync(uint uid, CancellationToken ct);
}
```

#### B. Authentication System

**Purpose**: Handle password and OAuth2 authentication

**Responsibilities**:
- Authenticate with password
- Perform OAuth2 flow (initial setup)
- Refresh OAuth2 tokens
- Store and encrypt credentials

**Key Interfaces**:
```csharp
public interface IAuthenticator
{
    Task AuthenticateAsync(IImapClient client, AccountConfig account, CancellationToken ct);
}

public interface IOAuth2Provider
{
    Task<string> GetAuthorizationUrlAsync(string redirectUri);
    Task<TokenResponse> ExchangeCodeForTokensAsync(string code);
    Task<TokenResponse> RefreshTokenAsync(string refreshToken);
}

// Implementations
public class PasswordAuthenticator : IAuthenticator { }
public class OAuth2Authenticator : IAuthenticator { }
public class GmailOAuth2Provider : IOAuth2Provider { }
```

#### C. Rule Engine

**Purpose**: Evaluate conditions and execute actions on messages

**Responsibilities**:
- Match messages against rule conditions
- Determine applicable rules for each account
- Execute actions in order
- Track rule application
- Handle errors gracefully

**Key Classes**:
```csharp
public interface IRuleEngine
{
    Task<RuleExecutionResult> ProcessMessageAsync(
        MimeMessage message,
        uint uid,
        AccountConfig account,
        IMailFolder folder,
        CancellationToken ct);
}

public class RuleEngine : IRuleEngine
{
    private readonly IConditionEvaluator _conditionEvaluator;
    private readonly IActionExecutor _actionExecutor;
    private readonly ILogger<RuleEngine> _logger;

    public async Task<RuleExecutionResult> ProcessMessageAsync(...) { }
}

public interface IConditionEvaluator
{
    bool Evaluate(MimeMessage message, RuleCondition condition);
}

public interface IActionExecutor
{
    Task<ActionResult> ExecuteAsync(
        RuleAction action,
        MimeMessage message,
        uint uid,
        IMailFolder folder,
        CancellationToken ct);
}
```

#### D. UID Tracking System

**Purpose**: Prevent reprocessing of messages

**Responsibilities**:
- Store processed message UIDs per account
- Check if message was already processed
- Track which rules were applied
- Clean up old entries

**Schema**:
```sql
CREATE TABLE ProcessedMessages (
    AccountName TEXT NOT NULL,
    UID INTEGER NOT NULL,
    ProcessedAt TEXT NOT NULL,
    RulesApplied TEXT, -- JSON array
    PRIMARY KEY (AccountName, UID)
);

CREATE INDEX idx_processed_at ON ProcessedMessages(ProcessedAt);
```

**Interface**:
```csharp
public interface IProcessedMessageStore
{
    Task<bool> IsProcessedAsync(string accountName, uint uid, CancellationToken ct);
    Task MarkProcessedAsync(string accountName, uint uid, string[] rulesApplied, CancellationToken ct);
    Task CleanupOldAsync(int daysToKeep, CancellationToken ct);
}
```

#### E. Audit Logger

**Purpose**: Record all actions for accountability and debugging

**Responsibilities**:
- Log every action taken
- Store in SQLite for queryability
- Provide reporting capabilities

**Schema**:
```sql
CREATE TABLE AuditLog (
    Id INTEGER PRIMARY KEY AUTOINCREMENT,
    Timestamp TEXT NOT NULL,
    AccountName TEXT NOT NULL,
    MessageUID INTEGER NOT NULL,
    MessageId TEXT,
    Subject TEXT,
    RuleName TEXT,
    ActionType TEXT NOT NULL,
    ActionDetails TEXT, -- JSON
    Success INTEGER NOT NULL,
    ErrorMessage TEXT
);

CREATE INDEX idx_audit_timestamp ON AuditLog(Timestamp);
CREATE INDEX idx_audit_account ON AuditLog(AccountName);
```

**Interface**:
```csharp
public interface IAuditLogger
{
    Task LogActionAsync(AuditEntry entry, CancellationToken ct);
    Task<IEnumerable<AuditEntry>> QueryAsync(AuditQuery query, CancellationToken ct);
}

public class AuditEntry
{
    public DateTime Timestamp { get; set; }
    public string AccountName { get; set; }
    public uint MessageUID { get; set; }
    public string? MessageId { get; set; }
    public string? Subject { get; set; }
    public string? RuleName { get; set; }
    public string ActionType { get; set; }
    public string? ActionDetails { get; set; }
    public bool Success { get; set; }
    public string? ErrorMessage { get; set; }
}
```

### 3. Account Processor

**Purpose**: Orchestrate processing of a single account

**Responsibilities**:
- Connect to IMAP account
- Fetch unprocessed messages
- Apply rate limiting
- Execute rule engine for each message
- Handle errors per message
- Create backups if configured

**Flow**:
```
1. Connect & Authenticate
2. Get INBOX (or configured folders)
3. Search for unread/new messages
4. For each message:
   a. Check if already processed (UID store)
   b. Apply rate limiting delay
   c. Fetch full message
   d. Run rule engine
   e. Execute actions
   f. Create backup if needed
   g. Mark as processed
   h. Log audit entry
5. Disconnect
```

**Interface**:
```csharp
public interface IAccountProcessor
{
    Task ProcessAccountAsync(AccountConfig account, IEnumerable<RuleConfig> rules, CancellationToken ct);
}

public class AccountProcessorOptions
{
    public bool DryRun { get; set; }
    public int MaxMessagesPerRun { get; set; } = 100;
}
```

## Design Principles

### 1. Dependency Injection

All components use constructor injection for testability:

```csharp
public class RuleEngine : IRuleEngine
{
    private readonly IConditionEvaluator _evaluator;
    private readonly IActionExecutor _executor;
    private readonly IAuditLogger _auditLogger;
    private readonly ILogger<RuleEngine> _logger;

    public RuleEngine(
        IConditionEvaluator evaluator,
        IActionExecutor executor,
        IAuditLogger auditLogger,
        ILogger<RuleEngine> logger)
    {
        _evaluator = evaluator;
        _executor = executor;
        _auditLogger = auditLogger;
        _logger = logger;
    }
}
```

### 2. Async/Await Throughout

All I/O operations are async:
- IMAP operations
- Database access
- File I/O
- HTTP calls (OAuth)

### 3. Cancellation Token Support

All async methods accept `CancellationToken` for graceful shutdown:

```csharp
public async Task ProcessAccountAsync(
    AccountConfig account,
    CancellationToken ct = default)
{
    ct.ThrowIfCancellationRequested();
    // ...
}
```

### 4. Structured Logging

Use Serilog with structured properties:

```csharp
_logger.LogInformation(
    "Processing message {MessageId} from {From} in account {Account}",
    message.MessageId,
    message.From,
    account.Name);
```

### 5. Error Handling

**Per-Account Isolation**: One account failure doesn't stop others

```csharp
foreach (var account in accounts)
{
    try
    {
        await ProcessAccountAsync(account, ct);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to process account {Account}", account.Name);
        // Continue to next account
    }
}
```

**Per-Message Isolation**: One message failure doesn't stop processing

```csharp
foreach (var uid in uids)
{
    try
    {
        await ProcessMessageAsync(uid, ct);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to process message {UID}", uid);
        // Continue to next message
    }
}
```

### 6. Rate Limiting

Configurable delays to prevent server throttling:

```csharp
public class RateLimiter
{
    private readonly SemaphoreSlim _semaphore;
    private readonly int _delayMs;

    public async Task ExecuteAsync(Func<Task> action, CancellationToken ct)
    {
        await _semaphore.WaitAsync(ct);
        try
        {
            await action();
            await Task.Delay(_delayMs, ct);
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

## Execution Modes

### 1. One-Time Mode (Default)

```bash
dotnet run
```

- Process all accounts once
- Exit when complete
- Suitable for cron jobs or scheduled tasks

### 2. Daemon Mode

```bash
dotnet run -- --daemon --interval 5
```

- Run continuously
- Check accounts on interval (minutes)
- Graceful shutdown on SIGTERM/SIGINT

```csharp
while (!ct.IsCancellationRequested)
{
    await ProcessAllAccountsAsync(ct);
    await Task.Delay(TimeSpan.FromMinutes(interval), ct);
}
```

### 3. Dry-Run Mode

```bash
dotnet run -- --dry-run
```

- Evaluate rules but don't execute actions
- Log what would happen
- Useful for testing new rules

```csharp
if (options.DryRun)
{
    _logger.LogInformation("DRY RUN: Would execute {Action}", action);
    return ActionResult.Success(dryRun: true);
}
```

### 4. Setup Mode

```bash
dotnet run -- setup-account gmail work
```

- Interactive OAuth2 setup
- Test authentication
- Save credentials

## Cross-Account Message Moving

**Challenge**: When moving message from Account A to Account B, need to track both UIDs

**Solution**: Extended UID tracking

```csharp
public class ProcessedMessageStore
{
    // Original tracking
    Task MarkProcessedAsync(string account, uint uid, string[] rules);

    // Cross-account tracking
    Task MarkCrossAccountMoveAsync(
        string sourceAccount,
        uint sourceUID,
        string targetAccount,
        uint targetUID);

    Task<bool> IsFromCrossAccountMoveAsync(string account, uint uid);
}
```

**Processing Logic**:
```csharp
// In rule action execution
if (action.Type == "moveToAccount")
{
    var sourceUID = uid;
    var targetUID = await MoveToAccountAsync(message, targetAccount, targetFolder);

    // Mark source as processed
    await _uidStore.MarkProcessedAsync(sourceAccount, sourceUID, new[] { rule.Name });

    // Track cross-account move
    await _uidStore.MarkCrossAccountMoveAsync(
        sourceAccount, sourceUID,
        targetAccount, targetUID);
}

// When processing target account
if (await _uidStore.IsFromCrossAccountMoveAsync(account.Name, uid))
{
    // Allow reprocessing for folder organization
    _logger.LogInformation("Message is from cross-account move, applying rules");
}
```

**Chain Depth Limit**: Prevent infinite loops

```csharp
public class ProcessingContext
{
    public int ChainDepth { get; set; }
    public const int MaxChainDepth = 3;
}

// Before processing
if (context.ChainDepth >= ProcessingContext.MaxChainDepth)
{
    _logger.LogWarning("Max chain depth reached for UID {UID}", uid);
    return;
}
```

## Security Considerations

### 1. Credential Storage

- Passwords encrypted using DPAPI (Windows) or keyring (Linux/macOS)
- OAuth refresh tokens encrypted
- Access tokens stored in memory only
- `.gitignore` configuration files with secrets

### 2. Configuration File Security

**appsettings.json** (committable):
```json
{
  "accounts": [
    {
      "name": "work",
      "server": "imap.gmail.com",
      "username": "user@example.com",
      "authType": "oauth2",
      "clientId": "${GMAIL_CLIENT_ID}"
    }
  ]
}
```

**appsettings.secrets.json** (not committed):
```json
{
  "accounts": [
    {
      "name": "work",
      "clientSecret": "actual_secret",
      "refreshToken": "encrypted_token"
    }
  ]
}
```

### 3. Network Security

- Require SSL/TLS by default
- Certificate validation
- Timeout configuration

## Performance Considerations

### 1. Message Fetching

- Fetch headers first, full body only if needed for rules
- Use IMAP search to filter on server side
- Batch operations when possible

### 2. Database Operations

- Use transactions for consistency
- Batch inserts for audit logs
- Index frequently queried columns

### 3. Memory Management

- Process messages one at a time
- Dispose IMAP connections properly
- Limit concurrent account processing

```csharp
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = 3, // Process 3 accounts concurrently
    CancellationToken = ct
};

await Parallel.ForEachAsync(accounts, options, async (account, ct) =>
{
    await ProcessAccountAsync(account, ct);
});
```

## Testing Strategy

### 1. Unit Tests

- Mock `IImapClient` for rule engine tests
- Test condition evaluation in isolation
- Test action execution logic
- Test UID tracking

### 2. Integration Tests

- Use test IMAP server (e.g., MailHog)
- Test full processing flow
- Test OAuth2 token refresh
- Test database operations

### 3. Manual Testing

- Dry-run mode with real accounts
- Small message batches
- Monitor audit logs
