# IMAP Operations and UID Tracking

## Overview

This document covers:
- MailKit library integration
- IMAP client wrapper implementation
- UID-based message tracking
- Cross-account message operations
- Rate limiting and retry logic

## MailKit Integration

### Why MailKit?

- **Mature**: Active development since 2013
- **OAuth2 Support**: Built-in SASL mechanisms
- **Standards Compliant**: RFC-compliant IMAP4rev1
- **Cross-Platform**: Works on Windows, Linux, macOS
- **Well Documented**: Comprehensive documentation and examples

### Installation

```xml
<PackageReference Include="MailKit" Version="4.3.0" />
```

## IMAP Client Wrapper

### Interface Design

```csharp
public interface IImapClient : IDisposable
{
    /// <summary>Connection state</summary>
    bool IsConnected { get; }

    /// <summary>Authentication state</summary>
    bool IsAuthenticated { get; }

    /// <summary>Connect to IMAP server</summary>
    Task ConnectAsync(
        string host,
        int port,
        bool useSsl,
        CancellationToken ct = default);

    /// <summary>Authenticate with SASL mechanism</summary>
    Task AuthenticateAsync(
        SaslMechanism mechanism,
        CancellationToken ct = default);

    /// <summary>Authenticate with username/password</summary>
    Task AuthenticateAsync(
        string username,
        string password,
        CancellationToken ct = default);

    /// <summary>Get a folder by path</summary>
    Task<IMailFolder> GetFolderAsync(
        string path,
        CancellationToken ct = default);

    /// <summary>Create a new folder</summary>
    Task<IMailFolder> CreateFolderAsync(
        string path,
        CancellationToken ct = default);

    /// <summary>List all folders</summary>
    Task<IReadOnlyList<IMailFolder>> GetFoldersAsync(
        CancellationToken ct = default);

    /// <summary>Disconnect from server</summary>
    Task DisconnectAsync(
        bool quit = true,
        CancellationToken ct = default);
}
```

### Implementation

```csharp
public class ImapClientWrapper : IImapClient
{
    private readonly ImapClient _client;
    private readonly ILogger<ImapClientWrapper> _logger;

    public bool IsConnected => _client.IsConnected;
    public bool IsAuthenticated => _client.IsAuthenticated;

    public ImapClientWrapper(ILogger<ImapClientWrapper> logger)
    {
        _logger = logger;
        _client = new ImapClient();

        // Configure client
        _client.ServerCertificateValidationCallback = (s, c, h, e) =>
        {
            // In production, implement proper certificate validation
            // For now, log and accept
            if (e != System.Net.Security.SslPolicyErrors.None)
            {
                _logger.LogWarning(
                    "Certificate validation error: {Error}. Accepting anyway.",
                    e);
            }
            return true;
        };
    }

    public async Task ConnectAsync(
        string host,
        int port,
        bool useSsl,
        CancellationToken ct = default)
    {
        _logger.LogInformation(
            "Connecting to {Host}:{Port} (SSL: {UseSsl})",
            host,
            port,
            useSsl);

        try
        {
            await _client.ConnectAsync(host, port, useSsl, ct);
            _logger.LogInformation("Connected successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to connect to {Host}:{Port}", host, port);
            throw;
        }
    }

    public async Task AuthenticateAsync(
        SaslMechanism mechanism,
        CancellationToken ct = default)
    {
        _logger.LogInformation("Authenticating with SASL mechanism");

        try
        {
            await _client.AuthenticateAsync(mechanism, ct);
            _logger.LogInformation("Authentication successful");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Authentication failed");
            throw;
        }
    }

    public async Task AuthenticateAsync(
        string username,
        string password,
        CancellationToken ct = default)
    {
        _logger.LogInformation("Authenticating user {Username}", username);

        try
        {
            await _client.AuthenticateAsync(username, password, ct);
            _logger.LogInformation("Authentication successful");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Authentication failed for {Username}", username);
            throw;
        }
    }

    public async Task<IMailFolder> GetFolderAsync(
        string path,
        CancellationToken ct = default)
    {
        _logger.LogDebug("Getting folder: {Path}", path);

        var folder = _client.GetFolder(path, ct);
        await folder.OpenAsync(FolderAccess.ReadWrite, ct);

        return folder;
    }

    public async Task<IMailFolder> CreateFolderAsync(
        string path,
        CancellationToken ct = default)
    {
        _logger.LogInformation("Creating folder: {Path}", path);

        var personalNamespace = _client.PersonalNamespaces.FirstOrDefault();
        if (personalNamespace == null)
            throw new InvalidOperationException("No personal namespace available");

        var folder = await personalNamespace.CreateAsync(path, true, ct);
        await folder.OpenAsync(FolderAccess.ReadWrite, ct);

        return folder;
    }

    public async Task<IReadOnlyList<IMailFolder>> GetFoldersAsync(
        CancellationToken ct = default)
    {
        _logger.LogDebug("Listing all folders");

        var personalNamespace = _client.PersonalNamespaces.FirstOrDefault();
        if (personalNamespace == null)
            return Array.Empty<IMailFolder>();

        return await personalNamespace.GetSubfoldersAsync(false, ct);
    }

    public async Task DisconnectAsync(
        bool quit = true,
        CancellationToken ct = default)
    {
        if (!_client.IsConnected)
            return;

        _logger.LogInformation("Disconnecting from server");

        try
        {
            await _client.DisconnectAsync(quit, ct);
            _logger.LogInformation("Disconnected successfully");
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Error during disconnect");
        }
    }

    public void Dispose()
    {
        _client?.Dispose();
    }
}
```

## Message Fetching

### Fetch Strategy

```csharp
public class MessageFetcher
{
    private readonly ILogger<MessageFetcher> _logger;

    public async Task<List<uint>> GetUnprocessedUidsAsync(
        IMailFolder folder,
        IProcessedMessageStore uidStore,
        string accountName,
        CancellationToken ct = default)
    {
        // Search for messages
        var query = SearchQuery.NotDeleted;
        var uids = await folder.SearchAsync(query, ct);

        _logger.LogInformation(
            "Found {Count} messages in folder {Folder}",
            uids.Count,
            folder.FullName);

        // Filter out already processed
        var unprocessed = new List<uint>();
        foreach (var uid in uids)
        {
            if (!await uidStore.IsProcessedAsync(accountName, uid, ct))
            {
                unprocessed.Add(uid);
            }
        }

        _logger.LogInformation(
            "{UnprocessedCount} unprocessed messages",
            unprocessed.Count);

        return unprocessed;
    }

    public async Task<MimeMessage> FetchMessageAsync(
        IMailFolder folder,
        uint uid,
        CancellationToken ct = default)
    {
        _logger.LogDebug("Fetching message {UID}", uid);

        // Fetch full message
        var message = await folder.GetMessageAsync(uid, ct);

        return message;
    }

    public async Task<MimeMessage> FetchMessageHeadersAsync(
        IMailFolder folder,
        uint uid,
        CancellationToken ct = default)
    {
        _logger.LogDebug("Fetching headers for message {UID}", uid);

        // Fetch only headers (faster)
        var message = await folder.GetMessageAsync(
            uid,
            ct,
            MessageSummaryItems.Headers | MessageSummaryItems.Envelope);

        return message;
    }
}
```

## UID Tracking System

### Why UIDs?

- **Persistent**: UIDs don't change (unlike sequence numbers)
- **Unique**: Each message has a unique UID in a folder
- **Efficient**: Avoid reprocessing already-handled messages

### Database Schema

```sql
-- Processed messages
CREATE TABLE ProcessedMessages (
    AccountName TEXT NOT NULL,
    UID INTEGER NOT NULL,
    ProcessedAt TEXT NOT NULL,  -- ISO 8601 datetime
    RulesApplied TEXT,          -- JSON array of rule names
    MessageId TEXT,             -- Email Message-ID header
    Subject TEXT,               -- For reference
    PRIMARY KEY (AccountName, UID)
);

CREATE INDEX idx_processed_at ON ProcessedMessages(ProcessedAt);
CREATE INDEX idx_message_id ON ProcessedMessages(MessageId);

-- Cross-account moves
CREATE TABLE CrossAccountMoves (
    Id INTEGER PRIMARY KEY AUTOINCREMENT,
    SourceAccount TEXT NOT NULL,
    SourceUID INTEGER NOT NULL,
    TargetAccount TEXT NOT NULL,
    TargetUID INTEGER NOT NULL,
    MovedAt TEXT NOT NULL,      -- ISO 8601 datetime
    UNIQUE(SourceAccount, SourceUID)
);

CREATE INDEX idx_target_lookup ON CrossAccountMoves(TargetAccount, TargetUID);
```

### Implementation

```csharp
public interface IProcessedMessageStore
{
    /// <summary>Check if message was already processed</summary>
    Task<bool> IsProcessedAsync(
        string accountName,
        uint uid,
        CancellationToken ct = default);

    /// <summary>Mark message as processed</summary>
    Task MarkProcessedAsync(
        string accountName,
        uint uid,
        string[] rulesApplied,
        string? messageId = null,
        string? subject = null,
        CancellationToken ct = default);

    /// <summary>Track cross-account move</summary>
    Task MarkCrossAccountMoveAsync(
        string sourceAccount,
        uint sourceUID,
        string targetAccount,
        uint targetUID,
        CancellationToken ct = default);

    /// <summary>Check if message is from cross-account move</summary>
    Task<bool> IsFromCrossAccountMoveAsync(
        string accountName,
        uint uid,
        CancellationToken ct = default);

    /// <summary>Clean up old entries</summary>
    Task CleanupOldAsync(
        int daysToKeep = 90,
        CancellationToken ct = default);
}

public class ProcessedMessageStore : IProcessedMessageStore
{
    private readonly string _connectionString;
    private readonly ILogger<ProcessedMessageStore> _logger;

    public ProcessedMessageStore(
        string databasePath,
        ILogger<ProcessedMessageStore> logger)
    {
        _connectionString = $"Data Source={databasePath}";
        _logger = logger;

        InitializeDatabase();
    }

    private void InitializeDatabase()
    {
        using var connection = new SqliteConnection(_connectionString);
        connection.Open();

        using var cmd = connection.CreateCommand();
        cmd.CommandText = @"
            CREATE TABLE IF NOT EXISTS ProcessedMessages (
                AccountName TEXT NOT NULL,
                UID INTEGER NOT NULL,
                ProcessedAt TEXT NOT NULL,
                RulesApplied TEXT,
                MessageId TEXT,
                Subject TEXT,
                PRIMARY KEY (AccountName, UID)
            );

            CREATE INDEX IF NOT EXISTS idx_processed_at
                ON ProcessedMessages(ProcessedAt);

            CREATE INDEX IF NOT EXISTS idx_message_id
                ON ProcessedMessages(MessageId);

            CREATE TABLE IF NOT EXISTS CrossAccountMoves (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                SourceAccount TEXT NOT NULL,
                SourceUID INTEGER NOT NULL,
                TargetAccount TEXT NOT NULL,
                TargetUID INTEGER NOT NULL,
                MovedAt TEXT NOT NULL,
                UNIQUE(SourceAccount, SourceUID)
            );

            CREATE INDEX IF NOT EXISTS idx_target_lookup
                ON CrossAccountMoves(TargetAccount, TargetUID);
        ";

        cmd.ExecuteNonQuery();
    }

    public async Task<bool> IsProcessedAsync(
        string accountName,
        uint uid,
        CancellationToken ct = default)
    {
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(ct);

        using var cmd = connection.CreateCommand();
        cmd.CommandText = @"
            SELECT COUNT(*)
            FROM ProcessedMessages
            WHERE AccountName = @AccountName AND UID = @UID
        ";
        cmd.Parameters.AddWithValue("@AccountName", accountName);
        cmd.Parameters.AddWithValue("@UID", uid);

        var count = (long)(await cmd.ExecuteScalarAsync(ct) ?? 0L);
        return count > 0;
    }

    public async Task MarkProcessedAsync(
        string accountName,
        uint uid,
        string[] rulesApplied,
        string? messageId = null,
        string? subject = null,
        CancellationToken ct = default)
    {
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(ct);

        using var cmd = connection.CreateCommand();
        cmd.CommandText = @"
            INSERT OR REPLACE INTO ProcessedMessages
            (AccountName, UID, ProcessedAt, RulesApplied, MessageId, Subject)
            VALUES (@AccountName, @UID, @ProcessedAt, @RulesApplied, @MessageId, @Subject)
        ";

        cmd.Parameters.AddWithValue("@AccountName", accountName);
        cmd.Parameters.AddWithValue("@UID", uid);
        cmd.Parameters.AddWithValue("@ProcessedAt", DateTime.UtcNow.ToString("O"));
        cmd.Parameters.AddWithValue("@RulesApplied", JsonSerializer.Serialize(rulesApplied));
        cmd.Parameters.AddWithValue("@MessageId", messageId ?? (object)DBNull.Value);
        cmd.Parameters.AddWithValue("@Subject", subject ?? (object)DBNull.Value);

        await cmd.ExecuteNonQueryAsync(ct);

        _logger.LogDebug(
            "Marked message {UID} as processed in account {Account}",
            uid,
            accountName);
    }

    public async Task MarkCrossAccountMoveAsync(
        string sourceAccount,
        uint sourceUID,
        string targetAccount,
        uint targetUID,
        CancellationToken ct = default)
    {
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(ct);

        using var cmd = connection.CreateCommand();
        cmd.CommandText = @"
            INSERT OR REPLACE INTO CrossAccountMoves
            (SourceAccount, SourceUID, TargetAccount, TargetUID, MovedAt)
            VALUES (@SourceAccount, @SourceUID, @TargetAccount, @TargetUID, @MovedAt)
        ";

        cmd.Parameters.AddWithValue("@SourceAccount", sourceAccount);
        cmd.Parameters.AddWithValue("@SourceUID", sourceUID);
        cmd.Parameters.AddWithValue("@TargetAccount", targetAccount);
        cmd.Parameters.AddWithValue("@TargetUID", targetUID);
        cmd.Parameters.AddWithValue("@MovedAt", DateTime.UtcNow.ToString("O"));

        await cmd.ExecuteNonQueryAsync(ct);

        _logger.LogInformation(
            "Tracked cross-account move: {SourceAccount}:{SourceUID} -> {TargetAccount}:{TargetUID}",
            sourceAccount,
            sourceUID,
            targetAccount,
            targetUID);
    }

    public async Task<bool> IsFromCrossAccountMoveAsync(
        string accountName,
        uint uid,
        CancellationToken ct = default)
    {
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(ct);

        using var cmd = connection.CreateCommand();
        cmd.CommandText = @"
            SELECT COUNT(*)
            FROM CrossAccountMoves
            WHERE TargetAccount = @Account AND TargetUID = @UID
        ";
        cmd.Parameters.AddWithValue("@Account", accountName);
        cmd.Parameters.AddWithValue("@UID", uid);

        var count = (long)(await cmd.ExecuteScalarAsync(ct) ?? 0L);
        return count > 0;
    }

    public async Task CleanupOldAsync(
        int daysToKeep = 90,
        CancellationToken ct = default)
    {
        var cutoffDate = DateTime.UtcNow.AddDays(-daysToKeep).ToString("O");

        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(ct);

        using var transaction = connection.BeginTransaction();

        try
        {
            // Clean processed messages
            using (var cmd = connection.CreateCommand())
            {
                cmd.CommandText = @"
                    DELETE FROM ProcessedMessages
                    WHERE ProcessedAt < @CutoffDate
                ";
                cmd.Parameters.AddWithValue("@CutoffDate", cutoffDate);

                var deleted = await cmd.ExecuteNonQueryAsync(ct);
                _logger.LogInformation(
                    "Deleted {Count} old processed messages",
                    deleted);
            }

            // Clean cross-account moves
            using (var cmd = connection.CreateCommand())
            {
                cmd.CommandText = @"
                    DELETE FROM CrossAccountMoves
                    WHERE MovedAt < @CutoffDate
                ";
                cmd.Parameters.AddWithValue("@CutoffDate", cutoffDate);

                var deleted = await cmd.ExecuteNonQueryAsync(ct);
                _logger.LogInformation(
                    "Deleted {Count} old cross-account move records",
                    deleted);
            }

            transaction.Commit();
        }
        catch
        {
            transaction.Rollback();
            throw;
        }
    }
}
```

## Rate Limiting

### Implementation

```csharp
public class RateLimiter
{
    private readonly SemaphoreSlim _semaphore;
    private readonly int _delayMs;
    private readonly int _maxPerMinute;
    private readonly Queue<DateTime> _actionTimestamps;
    private readonly ILogger<RateLimiter> _logger;

    public RateLimiter(
        RateLimitConfig config,
        ILogger<RateLimiter> logger)
    {
        _semaphore = new SemaphoreSlim(1, 1);
        _delayMs = config.DelayBetweenMessagesMs;
        _maxPerMinute = config.MaxActionsPerMinute;
        _actionTimestamps = new Queue<DateTime>();
        _logger = logger;
    }

    public async Task WaitAsync(CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);

        try
        {
            // Check per-minute rate limit
            var now = DateTime.UtcNow;
            var oneMinuteAgo = now.AddMinutes(-1);

            // Remove old timestamps
            while (_actionTimestamps.Count > 0 && _actionTimestamps.Peek() < oneMinuteAgo)
            {
                _actionTimestamps.Dequeue();
            }

            // Check if we've hit the limit
            if (_actionTimestamps.Count >= _maxPerMinute)
            {
                var oldestTimestamp = _actionTimestamps.Peek();
                var waitTime = oldestTimestamp.AddMinutes(1) - now;

                if (waitTime > TimeSpan.Zero)
                {
                    _logger.LogWarning(
                        "Rate limit reached, waiting {Seconds}s",
                        waitTime.TotalSeconds);

                    await Task.Delay(waitTime, ct);
                }
            }

            // Record this action
            _actionTimestamps.Enqueue(now);

            // Fixed delay between messages
            if (_delayMs > 0)
            {
                await Task.Delay(_delayMs, ct);
            }
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

## Retry Logic

```csharp
public class RetryPolicy
{
    private readonly int _maxRetries;
    private readonly int _retryDelaySeconds;
    private readonly ILogger _logger;

    public RetryPolicy(RateLimitConfig config, ILogger logger)
    {
        _maxRetries = config.MaxRetries;
        _retryDelaySeconds = config.RetryDelaySeconds;
        _logger = logger;
    }

    public async Task<T> ExecuteAsync<T>(
        Func<Task<T>> action,
        string operationName,
        CancellationToken ct = default)
    {
        for (int attempt = 1; attempt <= _maxRetries; attempt++)
        {
            try
            {
                return await action();
            }
            catch (Exception ex) when (attempt < _maxRetries && IsRetryable(ex))
            {
                _logger.LogWarning(
                    ex,
                    "Operation {Operation} failed (attempt {Attempt}/{MaxRetries}), retrying in {Delay}s",
                    operationName,
                    attempt,
                    _maxRetries,
                    _retryDelaySeconds);

                await Task.Delay(TimeSpan.FromSeconds(_retryDelaySeconds), ct);
            }
        }

        // Final attempt (or throw)
        return await action();
    }

    private bool IsRetryable(Exception ex)
    {
        return ex is IOException ||
               ex is TimeoutException ||
               ex is ImapProtocolException;
    }
}
```

## Account Processor

### Main Processing Loop

```csharp
public class AccountProcessor : IAccountProcessor
{
    private readonly IImapClientFactory _clientFactory;
    private readonly IAuthenticatorFactory _authFactory;
    private readonly IRuleEngine _ruleEngine;
    private readonly IProcessedMessageStore _uidStore;
    private readonly MessageFetcher _messageFetcher;
    private readonly RateLimiter _rateLimiter;
    private readonly RetryPolicy _retryPolicy;
    private readonly ILogger<AccountProcessor> _logger;

    public async Task ProcessAccountAsync(
        AccountConfig account,
        IEnumerable<RuleConfig> rules,
        ProcessingOptions options,
        CancellationToken ct = default)
    {
        _logger.LogInformation("Processing account: {Account}", account.Name);

        using var client = _clientFactory.Create();

        // Connect and authenticate
        await _retryPolicy.ExecuteAsync(
            async () =>
            {
                await client.ConnectAsync(
                    account.Server,
                    account.Port,
                    account.UseSsl,
                    ct);

                var authenticator = _authFactory.CreateAuthenticator(account);
                await authenticator.AuthenticateAsync(client, account, ct);

                return true;
            },
            $"Connect and authenticate {account.Name}",
            ct);

        try
        {
            // Process each configured folder
            foreach (var folderPath in account.CheckFolders)
            {
                await ProcessFolderAsync(
                    client,
                    folderPath,
                    account,
                    rules,
                    options,
                    ct);
            }
        }
        finally
        {
            await client.DisconnectAsync(true, ct);
        }

        _logger.LogInformation("Finished processing account: {Account}", account.Name);
    }

    private async Task ProcessFolderAsync(
        IImapClient client,
        string folderPath,
        AccountConfig account,
        IEnumerable<RuleConfig> rules,
        ProcessingOptions options,
        CancellationToken ct)
    {
        _logger.LogInformation(
            "Processing folder {Folder} in account {Account}",
            folderPath,
            account.Name);

        var folder = await client.GetFolderAsync(folderPath, ct);

        // Get unprocessed UIDs
        var uids = await _messageFetcher.GetUnprocessedUidsAsync(
            folder,
            _uidStore,
            account.Name,
            ct);

        // Limit by maxMessagesPerSession
        var maxMessages = account.RateLimits.MaxMessagesPerSession;
        if (uids.Count > maxMessages)
        {
            _logger.LogWarning(
                "Found {Count} messages but limiting to {Max}",
                uids.Count,
                maxMessages);
            uids = uids.Take(maxMessages).ToList();
        }

        var context = new ProcessingContext
        {
            Account = account,
            Folder = folder,
            DryRun = options.DryRun,
            ChainDepth = 0
        };

        // Process each message
        foreach (var uid in uids)
        {
            if (ct.IsCancellationRequested)
                break;

            // Apply rate limiting
            await _rateLimiter.WaitAsync(ct);

            try
            {
                await ProcessMessageAsync(uid, context, rules, ct);
            }
            catch (Exception ex)
            {
                _logger.LogError(
                    ex,
                    "Failed to process message {UID} in {Account}",
                    uid,
                    account.Name);

                // Continue to next message
            }
        }
    }

    private async Task ProcessMessageAsync(
        uint uid,
        ProcessingContext context,
        IEnumerable<RuleConfig> rules,
        CancellationToken ct)
    {
        _logger.LogDebug("Processing message {UID}", uid);

        // Fetch message
        var message = await _messageFetcher.FetchMessageAsync(context.Folder, uid, ct);

        // Run rule engine
        var results = await _ruleEngine.ProcessMessageAsync(message, uid, context, ct);

        // Mark as processed
        var appliedRules = results
            .Where(r => r.RuleMatched)
            .Select(r => r.RuleName ?? "unknown")
            .ToArray();

        await _uidStore.MarkProcessedAsync(
            context.Account.Name,
            uid,
            appliedRules,
            message.MessageId,
            message.Subject,
            ct);

        _logger.LogInformation(
            "Processed message {UID}, applied {RuleCount} rules",
            uid,
            appliedRules.Length);
    }
}
```

## Testing IMAP Operations

```csharp
[Fact]
public async Task ProcessAccountAsync_ProcessesUnreadMessages()
{
    // Arrange
    var mockClient = new Mock<IImapClient>();
    var mockFolder = new Mock<IMailFolder>();
    var mockUidStore = new Mock<IProcessedMessageStore>();

    mockFolder
        .Setup(f => f.SearchAsync(It.IsAny<SearchQuery>(), It.IsAny<CancellationToken>()))
        .ReturnsAsync(new List<uint> { 1, 2, 3 });

    mockUidStore
        .Setup(s => s.IsProcessedAsync(It.IsAny<string>(), It.IsAny<uint>(), It.IsAny<CancellationToken>()))
        .ReturnsAsync(false);

    var processor = new AccountProcessor(/* dependencies */);

    // Act
    await processor.ProcessAccountAsync(account, rules, options, CancellationToken.None);

    // Assert
    mockUidStore.Verify(
        s => s.MarkProcessedAsync(
            It.IsAny<string>(),
            It.IsAny<uint>(),
            It.IsAny<string[]>(),
            It.IsAny<string>(),
            It.IsAny<string>(),
            It.IsAny<CancellationToken>()),
        Times.Exactly(3));
}
```

## Best Practices

1. **Always use UIDs**: Never rely on sequence numbers
2. **Open folders ReadWrite**: Required for moving/flagging messages
3. **Handle folder not found**: Create folders if they don't exist
4. **Expunge after delete**: Call `ExpungeAsync()` to permanently delete
5. **Disconnect gracefully**: Use `DisconnectAsync(true)` to send QUIT
6. **Rate limit aggressively**: Prevent server throttling/bans
7. **Log all operations**: Essential for debugging and auditing
8. **Test with small batches**: Use `maxMessagesPerSession` during testing
