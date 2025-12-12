# Logging and Auditing

## Overview

The application uses dual logging:
1. **Serilog** for structured application logs (file-based)
2. **SQLite** for audit trail (queryable database)

This provides both developer-friendly logs and queryable audit records.

## Serilog Configuration

### Installation

```xml
<PackageReference Include="Serilog" Version="4.3.0" />
<PackageReference Include="Serilog.Sinks.File" Version="6.0.0" />
<PackageReference Include="Serilog.Sinks.Console" Version="6.0.0" />
<PackageReference Include="Serilog.Sinks.SQLite" Version="8.0.0" />
<PackageReference Include="Serilog.Extensions.Logging" Version="10.0.0" />
```

### Setup

```csharp
public class LoggingSetup
{
    public static ILogger ConfigureLogging(LoggingConfig config)
    {
        var logConfig = new LoggerConfiguration()
            .MinimumLevel.Is(ParseLogLevel(config.Level))
            .Enrich.FromLogContext()
            .Enrich.WithMachineName()
            .Enrich.WithThreadId();

        // Console sink (for development)
        logConfig.WriteTo.Console(
            outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception}");

        // File sink (rolling daily)
        logConfig.WriteTo.File(
            config.FilePath,
            rollingInterval: RollingInterval.Day,
            outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}",
            retainedFileCountLimit: 30); // Keep 30 days of logs

        return Log.Logger = logConfig.CreateLogger();
    }

    private static LogEventLevel ParseLogLevel(string level) => level.ToLowerInvariant() switch
    {
        "verbose" => LogEventLevel.Verbose,
        "debug" => LogEventLevel.Debug,
        "information" => LogEventLevel.Information,
        "warning" => LogEventLevel.Warning,
        "error" => LogEventLevel.Error,
        "fatal" => LogEventLevel.Fatal,
        _ => LogEventLevel.Information
    };
}
```

### Integration with Microsoft.Extensions.Logging

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddLogging(
        this IServiceCollection services,
        LoggingConfig config)
    {
        // Configure Serilog
        var logger = LoggingSetup.ConfigureLogging(config);

        // Add to DI
        services.AddSingleton(logger);
        services.AddLogging(builder =>
        {
            builder.ClearProviders();
            builder.AddSerilog(logger, dispose: true);
        });

        return services;
    }
}
```

### Structured Logging Examples

```csharp
// With structured properties
_logger.LogInformation(
    "Processing message {MessageId} from {Sender} in account {Account}",
    message.MessageId,
    message.From.Mailboxes.FirstOrDefault()?.Address,
    accountName);

// With exception
_logger.LogError(
    ex,
    "Failed to move message {UID} to folder {Folder}",
    uid,
    targetFolder);

// With context enrichment
using (_logger.BeginScope(new Dictionary<string, object>
{
    ["Account"] = accountName,
    ["Folder"] = folderName
}))
{
    _logger.LogInformation("Processing messages");
    // Subsequent logs will include Account and Folder properties
}
```

## Audit Database

### Schema

```sql
-- Audit log for all actions
CREATE TABLE AuditLog (
    Id INTEGER PRIMARY KEY AUTOINCREMENT,
    Timestamp TEXT NOT NULL,        -- ISO 8601 datetime
    AccountName TEXT NOT NULL,
    MessageUID INTEGER NOT NULL,
    MessageId TEXT,                 -- Email Message-ID header
    Subject TEXT,                   -- Email subject (truncated)
    FromAddress TEXT,               -- Sender email
    RuleName TEXT,                  -- Rule that triggered action
    ActionType TEXT NOT NULL,       -- Action type (moveToFolder, delete, etc.)
    ActionDetails TEXT,             -- JSON with action-specific details
    Success INTEGER NOT NULL,       -- 1 = success, 0 = failure
    ErrorMessage TEXT,              -- Error message if failed
    DryRun INTEGER NOT NULL DEFAULT 0  -- 1 if dry run
);

-- Indexes for efficient querying
CREATE INDEX idx_audit_timestamp ON AuditLog(Timestamp);
CREATE INDEX idx_audit_account ON AuditLog(AccountName);
CREATE INDEX idx_audit_message_id ON AuditLog(MessageId);
CREATE INDEX idx_audit_rule ON AuditLog(RuleName);
CREATE INDEX idx_audit_action ON AuditLog(ActionType);
CREATE INDEX idx_audit_success ON AuditLog(Success);

-- Summary statistics view
CREATE VIEW AuditSummary AS
SELECT
    DATE(Timestamp) AS Date,
    AccountName,
    RuleName,
    ActionType,
    COUNT(*) AS ActionCount,
    SUM(Success) AS SuccessCount,
    SUM(CASE WHEN Success = 0 THEN 1 ELSE 0 END) AS FailureCount
FROM AuditLog
WHERE DryRun = 0
GROUP BY DATE(Timestamp), AccountName, RuleName, ActionType;
```

### Implementation

```csharp
public interface IAuditLogger
{
    /// <summary>Log an action to the audit database</summary>
    Task LogActionAsync(
        AuditEntry entry,
        CancellationToken ct = default);

    /// <summary>Query audit entries</summary>
    Task<List<AuditEntry>> QueryAsync(
        AuditQuery query,
        CancellationToken ct = default);

    /// <summary>Get summary statistics</summary>
    Task<List<AuditSummary>> GetSummaryAsync(
        DateTime? startDate = null,
        DateTime? endDate = null,
        CancellationToken ct = default);
}

public class AuditEntry
{
    public long Id { get; set; }
    public DateTime Timestamp { get; set; }
    public string AccountName { get; set; } = "";
    public uint MessageUID { get; set; }
    public string? MessageId { get; set; }
    public string? Subject { get; set; }
    public string? FromAddress { get; set; }
    public string? RuleName { get; set; }
    public string ActionType { get; set; } = "";
    public string? ActionDetails { get; set; }
    public bool Success { get; set; }
    public string? ErrorMessage { get; set; }
    public bool DryRun { get; set; }
}

public class AuditQuery
{
    public DateTime? StartDate { get; set; }
    public DateTime? EndDate { get; set; }
    public string? AccountName { get; set; }
    public string? RuleName { get; set; }
    public string? ActionType { get; set; }
    public bool? Success { get; set; }
    public bool IncludeDryRun { get; set; } = false;
    public int Limit { get; set; } = 100;
}

public class AuditSummary
{
    public DateTime Date { get; set; }
    public string AccountName { get; set; } = "";
    public string? RuleName { get; set; }
    public string ActionType { get; set; } = "";
    public int ActionCount { get; set; }
    public int SuccessCount { get; set; }
    public int FailureCount { get; set; }
}

public class SqliteAuditLogger : IAuditLogger
{
    private readonly string _connectionString;
    private readonly ILogger<SqliteAuditLogger> _logger;

    public SqliteAuditLogger(
        string databasePath,
        ILogger<SqliteAuditLogger> logger)
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
            CREATE TABLE IF NOT EXISTS AuditLog (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                Timestamp TEXT NOT NULL,
                AccountName TEXT NOT NULL,
                MessageUID INTEGER NOT NULL,
                MessageId TEXT,
                Subject TEXT,
                FromAddress TEXT,
                RuleName TEXT,
                ActionType TEXT NOT NULL,
                ActionDetails TEXT,
                Success INTEGER NOT NULL,
                ErrorMessage TEXT,
                DryRun INTEGER NOT NULL DEFAULT 0
            );

            CREATE INDEX IF NOT EXISTS idx_audit_timestamp ON AuditLog(Timestamp);
            CREATE INDEX IF NOT EXISTS idx_audit_account ON AuditLog(AccountName);
            CREATE INDEX IF NOT EXISTS idx_audit_message_id ON AuditLog(MessageId);
            CREATE INDEX IF NOT EXISTS idx_audit_rule ON AuditLog(RuleName);
            CREATE INDEX IF NOT EXISTS idx_audit_action ON AuditLog(ActionType);
            CREATE INDEX IF NOT EXISTS idx_audit_success ON AuditLog(Success);

            CREATE VIEW IF NOT EXISTS AuditSummary AS
            SELECT
                DATE(Timestamp) AS Date,
                AccountName,
                RuleName,
                ActionType,
                COUNT(*) AS ActionCount,
                SUM(Success) AS SuccessCount,
                SUM(CASE WHEN Success = 0 THEN 1 ELSE 0 END) AS FailureCount
            FROM AuditLog
            WHERE DryRun = 0
            GROUP BY DATE(Timestamp), AccountName, RuleName, ActionType;
        ";

        cmd.ExecuteNonQuery();
    }

    public async Task LogActionAsync(
        AuditEntry entry,
        CancellationToken ct = default)
    {
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(ct);

        using var cmd = connection.CreateCommand();
        cmd.CommandText = @"
            INSERT INTO AuditLog
            (Timestamp, AccountName, MessageUID, MessageId, Subject, FromAddress,
             RuleName, ActionType, ActionDetails, Success, ErrorMessage, DryRun)
            VALUES
            (@Timestamp, @AccountName, @MessageUID, @MessageId, @Subject, @FromAddress,
             @RuleName, @ActionType, @ActionDetails, @Success, @ErrorMessage, @DryRun)
        ";

        cmd.Parameters.AddWithValue("@Timestamp", entry.Timestamp.ToString("O"));
        cmd.Parameters.AddWithValue("@AccountName", entry.AccountName);
        cmd.Parameters.AddWithValue("@MessageUID", entry.MessageUID);
        cmd.Parameters.AddWithValue("@MessageId", entry.MessageId ?? (object)DBNull.Value);
        cmd.Parameters.AddWithValue("@Subject", TruncateSubject(entry.Subject));
        cmd.Parameters.AddWithValue("@FromAddress", entry.FromAddress ?? (object)DBNull.Value);
        cmd.Parameters.AddWithValue("@RuleName", entry.RuleName ?? (object)DBNull.Value);
        cmd.Parameters.AddWithValue("@ActionType", entry.ActionType);
        cmd.Parameters.AddWithValue("@ActionDetails", entry.ActionDetails ?? (object)DBNull.Value);
        cmd.Parameters.AddWithValue("@Success", entry.Success ? 1 : 0);
        cmd.Parameters.AddWithValue("@ErrorMessage", entry.ErrorMessage ?? (object)DBNull.Value);
        cmd.Parameters.AddWithValue("@DryRun", entry.DryRun ? 1 : 0);

        await cmd.ExecuteNonQueryAsync(ct);
    }

    public async Task<List<AuditEntry>> QueryAsync(
        AuditQuery query,
        CancellationToken ct = default)
    {
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(ct);

        var whereClause = BuildWhereClause(query, out var parameters);

        using var cmd = connection.CreateCommand();
        cmd.CommandText = $@"
            SELECT Id, Timestamp, AccountName, MessageUID, MessageId, Subject, FromAddress,
                   RuleName, ActionType, ActionDetails, Success, ErrorMessage, DryRun
            FROM AuditLog
            {whereClause}
            ORDER BY Timestamp DESC
            LIMIT @Limit
        ";

        foreach (var param in parameters)
        {
            cmd.Parameters.AddWithValue(param.Key, param.Value);
        }
        cmd.Parameters.AddWithValue("@Limit", query.Limit);

        var entries = new List<AuditEntry>();

        using var reader = await cmd.ExecuteReaderAsync(ct);
        while (await reader.ReadAsync(ct))
        {
            entries.Add(new AuditEntry
            {
                Id = reader.GetInt64(0),
                Timestamp = DateTime.Parse(reader.GetString(1)),
                AccountName = reader.GetString(2),
                MessageUID = (uint)reader.GetInt64(3),
                MessageId = reader.IsDBNull(4) ? null : reader.GetString(4),
                Subject = reader.IsDBNull(5) ? null : reader.GetString(5),
                FromAddress = reader.IsDBNull(6) ? null : reader.GetString(6),
                RuleName = reader.IsDBNull(7) ? null : reader.GetString(7),
                ActionType = reader.GetString(8),
                ActionDetails = reader.IsDBNull(9) ? null : reader.GetString(9),
                Success = reader.GetInt32(10) == 1,
                ErrorMessage = reader.IsDBNull(11) ? null : reader.GetString(11),
                DryRun = reader.GetInt32(12) == 1
            });
        }

        return entries;
    }

    public async Task<List<AuditSummary>> GetSummaryAsync(
        DateTime? startDate = null,
        DateTime? endDate = null,
        CancellationToken ct = default)
    {
        using var connection = new SqliteConnection(_connectionString);
        await connection.OpenAsync(ct);

        using var cmd = connection.CreateCommand();

        var whereClause = "";
        if (startDate.HasValue || endDate.HasValue)
        {
            var conditions = new List<string>();
            if (startDate.HasValue)
            {
                conditions.Add("Date >= @StartDate");
                cmd.Parameters.AddWithValue("@StartDate", startDate.Value.ToString("yyyy-MM-dd"));
            }
            if (endDate.HasValue)
            {
                conditions.Add("Date <= @EndDate");
                cmd.Parameters.AddWithValue("@EndDate", endDate.Value.ToString("yyyy-MM-dd"));
            }
            whereClause = "WHERE " + string.Join(" AND ", conditions);
        }

        cmd.CommandText = $@"
            SELECT Date, AccountName, RuleName, ActionType, ActionCount, SuccessCount, FailureCount
            FROM AuditSummary
            {whereClause}
            ORDER BY Date DESC, AccountName, RuleName, ActionType
        ";

        var summaries = new List<AuditSummary>();

        using var reader = await cmd.ExecuteReaderAsync(ct);
        while (await reader.ReadAsync(ct))
        {
            summaries.Add(new AuditSummary
            {
                Date = DateTime.Parse(reader.GetString(0)),
                AccountName = reader.GetString(1),
                RuleName = reader.IsDBNull(2) ? null : reader.GetString(2),
                ActionType = reader.GetString(3),
                ActionCount = reader.GetInt32(4),
                SuccessCount = reader.GetInt32(5),
                FailureCount = reader.GetInt32(6)
            });
        }

        return summaries;
    }

    private string BuildWhereClause(
        AuditQuery query,
        out Dictionary<string, object> parameters)
    {
        parameters = new Dictionary<string, object>();
        var conditions = new List<string>();

        if (query.StartDate.HasValue)
        {
            conditions.Add("Timestamp >= @StartDate");
            parameters["@StartDate"] = query.StartDate.Value.ToString("O");
        }

        if (query.EndDate.HasValue)
        {
            conditions.Add("Timestamp <= @EndDate");
            parameters["@EndDate"] = query.EndDate.Value.ToString("O");
        }

        if (!string.IsNullOrWhiteSpace(query.AccountName))
        {
            conditions.Add("AccountName = @AccountName");
            parameters["@AccountName"] = query.AccountName;
        }

        if (!string.IsNullOrWhiteSpace(query.RuleName))
        {
            conditions.Add("RuleName = @RuleName");
            parameters["@RuleName"] = query.RuleName;
        }

        if (!string.IsNullOrWhiteSpace(query.ActionType))
        {
            conditions.Add("ActionType = @ActionType");
            parameters["@ActionType"] = query.ActionType;
        }

        if (query.Success.HasValue)
        {
            conditions.Add("Success = @Success");
            parameters["@Success"] = query.Success.Value ? 1 : 0;
        }

        if (!query.IncludeDryRun)
        {
            conditions.Add("DryRun = 0");
        }

        return conditions.Count > 0
            ? "WHERE " + string.Join(" AND ", conditions)
            : "";
    }

    private string? TruncateSubject(string? subject)
    {
        if (subject == null)
            return null;

        const int maxLength = 200;
        return subject.Length > maxLength
            ? subject.Substring(0, maxLength) + "..."
            : subject;
    }
}
```

## Audit Query CLI Commands

```csharp
public class AuditCommand
{
    private readonly IAuditLogger _auditLogger;

    public async Task QueryAsync(AuditQueryOptions options)
    {
        var query = new AuditQuery
        {
            StartDate = options.StartDate,
            EndDate = options.EndDate,
            AccountName = options.Account,
            RuleName = options.Rule,
            ActionType = options.ActionType,
            Success = options.SuccessOnly ? true : options.FailuresOnly ? false : null,
            IncludeDryRun = options.IncludeDryRun,
            Limit = options.Limit
        };

        var entries = await _auditLogger.QueryAsync(query);

        Console.WriteLine($"Found {entries.Count} audit entries:\n");

        foreach (var entry in entries)
        {
            var status = entry.DryRun ? "[DRY RUN]" : entry.Success ? "[SUCCESS]" : "[FAILED]";
            var rule = entry.RuleName != null ? $" | Rule: {entry.RuleName}" : "";

            Console.WriteLine(
                $"{entry.Timestamp:yyyy-MM-dd HH:mm:ss} {status} " +
                $"{entry.AccountName} | UID {entry.MessageUID} | " +
                $"Action: {entry.ActionType}{rule}");

            if (!string.IsNullOrWhiteSpace(entry.Subject))
                Console.WriteLine($"  Subject: {entry.Subject}");

            if (!entry.Success && !string.IsNullOrWhiteSpace(entry.ErrorMessage))
                Console.WriteLine($"  Error: {entry.ErrorMessage}");

            Console.WriteLine();
        }
    }

    public async Task SummaryAsync(SummaryOptions options)
    {
        var summaries = await _auditLogger.GetSummaryAsync(
            options.StartDate,
            options.EndDate);

        Console.WriteLine("Audit Summary:\n");
        Console.WriteLine($"{"Date",-12} {"Account",-15} {"Rule",-20} {"Action",-15} {"Total",8} {"Success",8} {"Failed",8}");
        Console.WriteLine(new string('-', 100));

        foreach (var summary in summaries)
        {
            Console.WriteLine(
                $"{summary.Date:yyyy-MM-dd}  " +
                $"{summary.AccountName,-15} " +
                $"{summary.RuleName ?? "N/A",-20} " +
                $"{summary.ActionType,-15} " +
                $"{summary.ActionCount,8} " +
                $"{summary.SuccessCount,8} " +
                $"{summary.FailureCount,8}");
        }
    }
}
```

### Example Usage

```bash
# Query recent actions
dotnet run -- audit query --limit 50

# Query specific account
dotnet run -- audit query --account work --limit 100

# Query failures only
dotnet run -- audit query --failures-only

# Query by date range
dotnet run -- audit query --start-date 2025-12-01 --end-date 2025-12-10

# Get summary statistics
dotnet run -- audit summary

# Get summary for last 7 days
dotnet run -- audit summary --start-date 2025-12-03
```

## Backup Service

### Implementation

```csharp
public interface IBackupService
{
    Task BackupMessageAsync(
        string accountName,
        MimeMessage message,
        uint uid,
        CancellationToken ct = default);
}

public class BackupService : IBackupService
{
    private readonly string _backupPath;
    private readonly ILogger<BackupService> _logger;

    public BackupService(
        StorageConfig storageConfig,
        ILogger<BackupService> logger)
    {
        _backupPath = storageConfig.BackupPath;
        _logger = logger;

        Directory.CreateDirectory(_backupPath);
    }

    public async Task BackupMessageAsync(
        string accountName,
        MimeMessage message,
        uint uid,
        CancellationToken ct = default)
    {
        try
        {
            // Create account directory
            var accountPath = Path.Combine(_backupPath, SanitizeFileName(accountName));
            Directory.CreateDirectory(accountPath);

            // Create backup filename: YYYYMMDD_HHMMSS_UID_MessageId.eml
            var timestamp = DateTime.UtcNow.ToString("yyyyMMdd_HHmmss");
            var messageId = SanitizeFileName(message.MessageId ?? "unknown");
            var fileName = $"{timestamp}_UID{uid}_{messageId}.eml";
            var filePath = Path.Combine(accountPath, fileName);

            // Save message
            await using var stream = File.Create(filePath);
            await message.WriteToAsync(stream, ct);

            _logger.LogInformation(
                "Backed up message {UID} from {Account} to {Path}",
                uid,
                accountName,
                filePath);
        }
        catch (Exception ex)
        {
            _logger.LogError(
                ex,
                "Failed to backup message {UID} from {Account}",
                uid,
                accountName);

            // Don't throw - backup failure shouldn't stop processing
        }
    }

    private string SanitizeFileName(string fileName)
    {
        var invalid = Path.GetInvalidFileNameChars();
        var sanitized = string.Join("_", fileName.Split(invalid));

        // Limit length
        const int maxLength = 50;
        return sanitized.Length > maxLength
            ? sanitized.Substring(0, maxLength)
            : sanitized;
    }
}
```

### Backup Cleanup

```csharp
public class BackupCleanupService
{
    private readonly string _backupPath;
    private readonly ILogger<BackupCleanupService> _logger;

    public async Task CleanupOldBackupsAsync(
        int retentionDays,
        CancellationToken ct = default)
    {
        _logger.LogInformation(
            "Cleaning up backups older than {Days} days",
            retentionDays);

        var cutoffDate = DateTime.UtcNow.AddDays(-retentionDays);
        var deletedCount = 0;

        await Task.Run(() =>
        {
            var files = Directory.GetFiles(_backupPath, "*.eml", SearchOption.AllDirectories);

            foreach (var file in files)
            {
                if (ct.IsCancellationRequested)
                    break;

                var fileInfo = new FileInfo(file);
                if (fileInfo.LastWriteTimeUtc < cutoffDate)
                {
                    try
                    {
                        File.Delete(file);
                        deletedCount++;
                    }
                    catch (Exception ex)
                    {
                        _logger.LogWarning(ex, "Failed to delete backup file {File}", file);
                    }
                }
            }
        }, ct);

        _logger.LogInformation("Deleted {Count} old backup files", deletedCount);
    }
}
```

## Performance Monitoring

### Logging Performance Metrics

```csharp
public class PerformanceLogger
{
    private readonly ILogger<PerformanceLogger> _logger;

    public async Task<T> MeasureAsync<T>(
        string operationName,
        Func<Task<T>> operation,
        CancellationToken ct = default)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            var result = await operation();
            stopwatch.Stop();

            _logger.LogInformation(
                "Operation {Operation} completed in {ElapsedMs}ms",
                operationName,
                stopwatch.ElapsedMilliseconds);

            return result;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();

            _logger.LogError(
                ex,
                "Operation {Operation} failed after {ElapsedMs}ms",
                operationName,
                stopwatch.ElapsedMilliseconds);

            throw;
        }
    }
}
```

### Usage Example

```csharp
var result = await _perfLogger.MeasureAsync(
    $"Process account {accountName}",
    async () => await ProcessAccountAsync(account, rules, ct),
    ct);
```

## Best Practices

1. **Structured Logging**: Always use structured properties, never string interpolation
2. **Appropriate Levels**: Use correct log levels (Debug, Information, Warning, Error)
3. **Context Enrichment**: Use scopes to add context to related log entries
4. **Audit Everything**: Log all actions that modify data
5. **Query-Friendly**: Design audit schema for common queries
6. **Retention Policy**: Clean up old logs and backups regularly
7. **Performance**: Use async methods for all database operations
8. **Error Handling**: Don't let logging failures crash the application
