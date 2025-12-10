# Rule Engine Specification

## Overview

The rule engine evaluates message conditions and executes actions when conditions are met. It provides flexible, Thunderbird-style message filtering with support for complex condition matching and various action types.

## Rule Processing Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Message Retrieved                        │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│            Filter Rules by Account Applicability            │
│        (Check applyToAccounts: "ALL", specific names)       │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                Sort Rules by Priority                       │
│              (Lower number = higher priority)               │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│             For Each Rule (in priority order):              │
│                                                             │
│  1. Evaluate Conditions                                     │
│     ├─ Match ALL (AND) or ANY (OR)                         │
│     ├─ Text conditions (from, to, subject, body)           │
│     ├─ Meta conditions (flags, attachments, date, size)    │
│     └─ Return true/false                                   │
│                                                             │
│  2. If Conditions Match:                                    │
│     ├─ Execute Actions (in order)                          │
│     ├─ Log Results                                         │
│     └─ Check stopProcessing flag                           │
│                                                             │
│  3. If stopProcessing = true:                               │
│     └─ Stop evaluating remaining rules                     │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│            Mark Message as Processed (UID store)            │
└─────────────────────────────────────────────────────────────┘
```

## Condition Evaluation

### Text Condition Matching

```csharp
public class TextConditionEvaluator
{
    public bool Evaluate(string? text, TextCondition condition)
    {
        if (string.IsNullOrEmpty(text))
            return false;

        var comparison = condition.CaseInsensitive
            ? StringComparison.OrdinalIgnoreCase
            : StringComparison.Ordinal;

        // Check Contains
        if (!string.IsNullOrEmpty(condition.Contains))
        {
            return text.Contains(condition.Contains, comparison);
        }

        // Check StartsWith
        if (!string.IsNullOrEmpty(condition.StartsWith))
        {
            return text.StartsWith(condition.StartsWith, comparison);
        }

        // Check EndsWith
        if (!string.IsNullOrEmpty(condition.EndsWith))
        {
            return text.EndsWith(condition.EndsWith, comparison);
        }

        // Check Equals
        if (!string.IsNullOrEmpty(condition.Equals))
        {
            return text.Equals(condition.Equals, comparison);
        }

        // Check Regex
        if (!string.IsNullOrEmpty(condition.Regex))
        {
            var options = condition.CaseInsensitive
                ? RegexOptions.IgnoreCase
                : RegexOptions.None;

            return System.Text.RegularExpressions.Regex.IsMatch(
                text,
                condition.Regex,
                options | RegexOptions.Compiled);
        }

        return false;
    }
}
```

### Email Address Field Matching

**From Field** (checks From + Sender + Reply-To):

```csharp
public bool EvaluateFromCondition(MimeMessage message, TextCondition condition)
{
    var addresses = new List<string>();

    // From addresses
    addresses.AddRange(message.From.Mailboxes.Select(m => m.Address));

    // Sender
    if (message.Sender != null)
        addresses.Add(message.Sender.Address);

    // Reply-To addresses
    addresses.AddRange(message.ReplyTo.Mailboxes.Select(m => m.Address));

    // Check if any address matches
    return addresses.Any(addr => _textEvaluator.Evaluate(addr, condition));
}
```

**To Field** (checks To + Cc + Bcc):

```csharp
public bool EvaluateToCondition(MimeMessage message, TextCondition condition)
{
    var addresses = new List<string>();

    // To addresses
    addresses.AddRange(message.To.Mailboxes.Select(m => m.Address));

    // Cc addresses
    addresses.AddRange(message.Cc.Mailboxes.Select(m => m.Address));

    // Bcc addresses
    addresses.AddRange(message.Bcc.Mailboxes.Select(m => m.Address));

    // Check if any address matches
    return addresses.Any(addr => _textEvaluator.Evaluate(addr, condition));
}
```

### Date Condition Matching

```csharp
public class DateConditionEvaluator
{
    public bool Evaluate(DateTimeOffset messageDate, DateCondition condition)
    {
        var now = DateTimeOffset.UtcNow;

        // Check Before
        if (!string.IsNullOrEmpty(condition.Before))
        {
            if (DateTimeOffset.TryParse(condition.Before, out var before))
            {
                if (messageDate >= before)
                    return false;
            }
        }

        // Check After
        if (!string.IsNullOrEmpty(condition.After))
        {
            if (DateTimeOffset.TryParse(condition.After, out var after))
            {
                if (messageDate <= after)
                    return false;
            }
        }

        // Check WithinLastDays
        if (condition.WithinLastDays.HasValue)
        {
            var cutoff = now.AddDays(-condition.WithinLastDays.Value);
            if (messageDate < cutoff)
                return false;
        }

        // Check OlderThanDays
        if (condition.OlderThanDays.HasValue)
        {
            var cutoff = now.AddDays(-condition.OlderThanDays.Value);
            if (messageDate >= cutoff)
                return false;
        }

        return true;
    }
}
```

### Complete Condition Evaluator

```csharp
public interface IConditionEvaluator
{
    bool Evaluate(MimeMessage message, RuleCondition condition);
}

public class ConditionEvaluator : IConditionEvaluator
{
    private readonly TextConditionEvaluator _textEvaluator;
    private readonly DateConditionEvaluator _dateEvaluator;
    private readonly ILogger<ConditionEvaluator> _logger;

    public ConditionEvaluator(ILogger<ConditionEvaluator> logger)
    {
        _textEvaluator = new TextConditionEvaluator();
        _dateEvaluator = new DateConditionEvaluator();
        _logger = logger;
    }

    public bool Evaluate(MimeMessage message, RuleCondition condition)
    {
        var results = new List<bool>();

        // Evaluate From
        if (condition.From != null)
        {
            results.Add(EvaluateFromCondition(message, condition.From));
        }

        // Evaluate To
        if (condition.To != null)
        {
            results.Add(EvaluateToCondition(message, condition.To));
        }

        // Evaluate Subject
        if (condition.Subject != null)
        {
            results.Add(_textEvaluator.Evaluate(message.Subject, condition.Subject));
        }

        // Evaluate Body
        if (condition.Body != null)
        {
            var bodyText = GetBodyText(message);
            results.Add(_textEvaluator.Evaluate(bodyText, condition.Body));
        }

        // Evaluate HasAttachment
        if (condition.HasAttachment.HasValue)
        {
            var hasAttachment = message.Attachments.Any();
            results.Add(hasAttachment == condition.HasAttachment.Value);
        }

        // Evaluate IsFlagged
        if (condition.IsFlagged.HasValue)
        {
            var isFlagged = message.Flags.HasFlag(MessageFlags.Flagged);
            results.Add(isFlagged == condition.IsFlagged.Value);
        }

        // Evaluate IsUnread
        if (condition.IsUnread.HasValue)
        {
            var isUnread = !message.Flags.HasFlag(MessageFlags.Seen);
            results.Add(isUnread == condition.IsUnread.Value);
        }

        // Evaluate ReceivedDate
        if (condition.ReceivedDate != null)
        {
            results.Add(_dateEvaluator.Evaluate(message.Date, condition.ReceivedDate));
        }

        // Evaluate Size
        if (condition.SizeGreaterThanKB.HasValue)
        {
            var sizeKB = GetMessageSizeKB(message);
            results.Add(sizeKB > condition.SizeGreaterThanKB.Value);
        }

        if (condition.SizeLessThanKB.HasValue)
        {
            var sizeKB = GetMessageSizeKB(message);
            results.Add(sizeKB < condition.SizeLessThanKB.Value);
        }

        // No conditions = always match
        if (results.Count == 0)
            return true;

        // Combine results based on matchType
        var matchesConditions = condition.MatchType.ToLowerInvariant() switch
        {
            "all" => results.All(r => r),
            "any" => results.Any(r => r),
            _ => throw new NotSupportedException(
                $"MatchType '{condition.MatchType}' is not supported")
        };

        return matchesConditions;
    }

    private bool EvaluateFromCondition(MimeMessage message, TextCondition condition)
    {
        var addresses = new List<string>();
        addresses.AddRange(message.From.Mailboxes.Select(m => m.Address));
        if (message.Sender != null) addresses.Add(message.Sender.Address);
        addresses.AddRange(message.ReplyTo.Mailboxes.Select(m => m.Address));

        return addresses.Any(addr => _textEvaluator.Evaluate(addr, condition));
    }

    private bool EvaluateToCondition(MimeMessage message, TextCondition condition)
    {
        var addresses = new List<string>();
        addresses.AddRange(message.To.Mailboxes.Select(m => m.Address));
        addresses.AddRange(message.Cc.Mailboxes.Select(m => m.Address));
        addresses.AddRange(message.Bcc.Mailboxes.Select(m => m.Address));

        return addresses.Any(addr => _textEvaluator.Evaluate(addr, condition));
    }

    private string GetBodyText(MimeMessage message)
    {
        if (message.TextBody != null)
            return message.TextBody;

        if (message.HtmlBody != null)
            return message.HtmlBody;

        return string.Empty;
    }

    private long GetMessageSizeKB(MimeMessage message)
    {
        using var stream = new MemoryStream();
        message.WriteTo(stream);
        return stream.Length / 1024;
    }
}
```

## Action Execution

### Action Executor Interface

```csharp
public interface IActionExecutor
{
    Task<ActionResult> ExecuteAsync(
        RuleAction action,
        MimeMessage message,
        uint uid,
        IMailFolder folder,
        ProcessingContext context,
        CancellationToken ct = default);
}
```

### Action Implementations

#### Move to Folder

```csharp
public class MoveToFolderAction : IActionHandler
{
    private readonly ILogger<MoveToFolderAction> _logger;

    public async Task<ActionResult> ExecuteAsync(
        RuleAction action,
        MimeMessage message,
        uint uid,
        IMailFolder folder,
        ProcessingContext context,
        CancellationToken ct)
    {
        if (string.IsNullOrWhiteSpace(action.Folder))
            return ActionResult.Failure("Folder name is required");

        if (context.DryRun)
        {
            _logger.LogInformation(
                "DRY RUN: Would move message {UID} to folder '{Folder}'",
                uid,
                action.Folder);
            return ActionResult.Success(dryRun: true);
        }

        try
        {
            // Get destination folder
            var destFolder = await GetOrCreateFolderAsync(
                folder.Client,
                action.Folder,
                ct);

            // Move message
            await folder.MoveToAsync(uid, destFolder, ct);

            _logger.LogInformation(
                "Moved message {UID} to folder '{Folder}'",
                uid,
                action.Folder);

            return ActionResult.Success(new Dictionary<string, object>
            {
                ["targetFolder"] = action.Folder
            });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to move message {UID}", uid);
            return ActionResult.Failure(ex.Message);
        }
    }

    private async Task<IMailFolder> GetOrCreateFolderAsync(
        IImapClient client,
        string folderPath,
        CancellationToken ct)
    {
        try
        {
            return await client.GetFolderAsync(folderPath, ct);
        }
        catch (FolderNotFoundException)
        {
            _logger.LogInformation("Creating folder '{Folder}'", folderPath);
            return await client.CreateFolderAsync(folderPath, ct);
        }
    }
}
```

#### Move to Account (Cross-Account)

```csharp
public class MoveToAccountAction : IActionHandler
{
    private readonly IImapClientFactory _clientFactory;
    private readonly IAuthenticatorFactory _authFactory;
    private readonly IProcessedMessageStore _uidStore;
    private readonly ILogger<MoveToAccountAction> _logger;

    public async Task<ActionResult> ExecuteAsync(
        RuleAction action,
        MimeMessage message,
        uint uid,
        IMailFolder sourceFolder,
        ProcessingContext context,
        CancellationToken ct)
    {
        if (string.IsNullOrWhiteSpace(action.TargetAccountName))
            return ActionResult.Failure("Target account name is required");

        if (context.ChainDepth >= ProcessingContext.MaxChainDepth)
        {
            _logger.LogWarning(
                "Max chain depth {MaxDepth} reached for UID {UID}",
                ProcessingContext.MaxChainDepth,
                uid);
            return ActionResult.Failure("Max processing chain depth reached");
        }

        if (context.DryRun)
        {
            _logger.LogInformation(
                "DRY RUN: Would move message {UID} to account '{Account}' folder '{Folder}'",
                uid,
                action.TargetAccountName,
                action.TargetFolder ?? "INBOX");
            return ActionResult.Success(dryRun: true);
        }

        try
        {
            // Get target account config
            var targetAccount = GetAccountConfig(action.TargetAccountName);

            // Connect to target account
            using var targetClient = await ConnectToAccountAsync(targetAccount, ct);

            // Get target folder
            var targetFolder = await targetClient.GetFolderAsync(
                action.TargetFolder ?? "INBOX",
                ct);

            // Append message to target
            var targetUid = await targetFolder.AppendAsync(message, ct);

            // Track cross-account move
            await _uidStore.MarkCrossAccountMoveAsync(
                context.Account.Name,
                uid,
                action.TargetAccountName,
                targetUid,
                ct);

            // Delete from source if move was successful
            if (action.MarkAsProcessed)
            {
                await sourceFolder.AddFlagsAsync(uid, MessageFlags.Deleted, ct);
                await sourceFolder.ExpungeAsync(ct);
            }

            _logger.LogInformation(
                "Moved message {SourceUID} from {SourceAccount} to {TargetAccount} (UID {TargetUID})",
                uid,
                context.Account.Name,
                action.TargetAccountName,
                targetUid);

            return ActionResult.Success(new Dictionary<string, object>
            {
                ["targetAccount"] = action.TargetAccountName,
                ["targetFolder"] = action.TargetFolder ?? "INBOX",
                ["targetUid"] = targetUid
            });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to move message to account {Account}", action.TargetAccountName);
            return ActionResult.Failure(ex.Message);
        }
    }
}
```

#### Mark as Read/Unread

```csharp
public class MarkAsReadAction : IActionHandler
{
    public async Task<ActionResult> ExecuteAsync(
        RuleAction action,
        MimeMessage message,
        uint uid,
        IMailFolder folder,
        ProcessingContext context,
        CancellationToken ct)
    {
        if (context.DryRun)
        {
            _logger.LogInformation("DRY RUN: Would mark message {UID} as read", uid);
            return ActionResult.Success(dryRun: true);
        }

        await folder.AddFlagsAsync(uid, MessageFlags.Seen, ct);
        return ActionResult.Success();
    }
}

public class MarkAsUnreadAction : IActionHandler
{
    public async Task<ActionResult> ExecuteAsync(
        RuleAction action,
        MimeMessage message,
        uint uid,
        IMailFolder folder,
        ProcessingContext context,
        CancellationToken ct)
    {
        if (context.DryRun)
        {
            _logger.LogInformation("DRY RUN: Would mark message {UID} as unread", uid);
            return ActionResult.Success(dryRun: true);
        }

        await folder.RemoveFlagsAsync(uid, MessageFlags.Seen, ct);
        return ActionResult.Success();
    }
}
```

#### Add/Remove Flag

```csharp
public class AddFlagAction : IActionHandler
{
    public async Task<ActionResult> ExecuteAsync(
        RuleAction action,
        MimeMessage message,
        uint uid,
        IMailFolder folder,
        ProcessingContext context,
        CancellationToken ct)
    {
        if (string.IsNullOrWhiteSpace(action.Flag))
            return ActionResult.Failure("Flag name is required");

        var flag = ParseFlag(action.Flag);

        if (context.DryRun)
        {
            _logger.LogInformation("DRY RUN: Would add flag {Flag} to message {UID}", action.Flag, uid);
            return ActionResult.Success(dryRun: true);
        }

        await folder.AddFlagsAsync(uid, flag, ct);
        return ActionResult.Success();
    }

    private MessageFlags ParseFlag(string flagName) => flagName.ToLowerInvariant() switch
    {
        "seen" => MessageFlags.Seen,
        "flagged" => MessageFlags.Flagged,
        "answered" => MessageFlags.Answered,
        "draft" => MessageFlags.Draft,
        "deleted" => MessageFlags.Deleted,
        _ => throw new NotSupportedException($"Flag '{flagName}' is not supported")
    };
}
```

#### Delete Message

```csharp
public class DeleteMessageAction : IActionHandler
{
    private readonly IBackupService _backupService;

    public async Task<ActionResult> ExecuteAsync(
        RuleAction action,
        MimeMessage message,
        uint uid,
        IMailFolder folder,
        ProcessingContext context,
        CancellationToken ct)
    {
        if (context.DryRun)
        {
            _logger.LogInformation("DRY RUN: Would delete message {UID}", uid);
            return ActionResult.Success(dryRun: true);
        }

        // Create backup if configured
        if (ShouldBackup(context.Account.Backup.Mode, "delete"))
        {
            await _backupService.BackupMessageAsync(
                context.Account.Name,
                message,
                uid,
                ct);
        }

        // Mark as deleted and expunge
        await folder.AddFlagsAsync(uid, MessageFlags.Deleted, ct);
        await folder.ExpungeAsync(ct);

        _logger.LogInformation("Deleted message {UID}", uid);
        return ActionResult.Success();
    }
}
```

### Action Executor Implementation

```csharp
public class ActionExecutor : IActionExecutor
{
    private readonly Dictionary<string, IActionHandler> _handlers;
    private readonly ILogger<ActionExecutor> _logger;

    public ActionExecutor(
        IServiceProvider serviceProvider,
        ILogger<ActionExecutor> logger)
    {
        _logger = logger;

        // Register action handlers
        _handlers = new Dictionary<string, IActionHandler>(StringComparer.OrdinalIgnoreCase)
        {
            ["moveToFolder"] = serviceProvider.GetRequiredService<MoveToFolderAction>(),
            ["moveToAccount"] = serviceProvider.GetRequiredService<MoveToAccountAction>(),
            ["delete"] = serviceProvider.GetRequiredService<DeleteMessageAction>(),
            ["markAsRead"] = serviceProvider.GetRequiredService<MarkAsReadAction>(),
            ["markAsUnread"] = serviceProvider.GetRequiredService<MarkAsUnreadAction>(),
            ["addFlag"] = serviceProvider.GetRequiredService<AddFlagAction>(),
            ["removeFlag"] = serviceProvider.GetRequiredService<RemoveFlagAction>()
        };
    }

    public async Task<ActionResult> ExecuteAsync(
        RuleAction action,
        MimeMessage message,
        uint uid,
        IMailFolder folder,
        ProcessingContext context,
        CancellationToken ct = default)
    {
        if (!_handlers.TryGetValue(action.Type, out var handler))
        {
            _logger.LogError("Unknown action type: {ActionType}", action.Type);
            return ActionResult.Failure($"Unknown action type: {action.Type}");
        }

        _logger.LogDebug(
            "Executing action {ActionType} on message {UID}",
            action.Type,
            uid);

        return await handler.ExecuteAsync(action, message, uid, folder, context, ct);
    }
}
```

## Rule Engine Implementation

```csharp
public interface IRuleEngine
{
    Task<List<RuleExecutionResult>> ProcessMessageAsync(
        MimeMessage message,
        uint uid,
        ProcessingContext context,
        CancellationToken ct = default);
}

public class RuleEngine : IRuleEngine
{
    private readonly IConditionEvaluator _conditionEvaluator;
    private readonly IActionExecutor _actionExecutor;
    private readonly IAuditLogger _auditLogger;
    private readonly ILogger<RuleEngine> _logger;

    public RuleEngine(
        IConditionEvaluator conditionEvaluator,
        IActionExecutor actionExecutor,
        IAuditLogger auditLogger,
        ILogger<RuleEngine> logger)
    {
        _conditionEvaluator = conditionEvaluator;
        _actionExecutor = actionExecutor;
        _auditLogger = auditLogger;
        _logger = logger;
    }

    public async Task<List<RuleExecutionResult>> ProcessMessageAsync(
        MimeMessage message,
        uint uid,
        ProcessingContext context,
        CancellationToken ct = default)
    {
        var results = new List<RuleExecutionResult>();

        // Get applicable rules for this account
        var applicableRules = GetApplicableRules(context.Account);

        // Sort by priority
        var sortedRules = applicableRules.OrderBy(r => r.Priority).ToList();

        _logger.LogDebug(
            "Processing message {UID} with {RuleCount} rules",
            uid,
            sortedRules.Count);

        foreach (var rule in sortedRules)
        {
            if (!rule.Enabled)
                continue;

            // Evaluate conditions
            var conditionsMatch = _conditionEvaluator.Evaluate(message, rule.Conditions);

            if (!conditionsMatch)
            {
                _logger.LogDebug(
                    "Rule '{Rule}' did not match message {UID}",
                    rule.Name,
                    uid);
                continue;
            }

            _logger.LogInformation(
                "Rule '{Rule}' matched message {UID}, executing {ActionCount} actions",
                rule.Name,
                uid,
                rule.Actions.Length);

            var result = new RuleExecutionResult
            {
                RuleName = rule.Name,
                RuleMatched = true,
                StopProcessing = rule.StopProcessing
            };

            // Execute actions
            foreach (var action in rule.Actions)
            {
                var actionResult = await _actionExecutor.ExecuteAsync(
                    action,
                    message,
                    uid,
                    context.Folder,
                    context,
                    ct);

                result.ActionResults.Add(actionResult);

                // Log to audit
                await _auditLogger.LogActionAsync(new AuditEntry
                {
                    Timestamp = DateTime.UtcNow,
                    AccountName = context.Account.Name,
                    MessageUID = uid,
                    MessageId = message.MessageId,
                    Subject = message.Subject,
                    RuleName = rule.Name,
                    ActionType = action.Type,
                    ActionDetails = JsonSerializer.Serialize(actionResult.Details),
                    Success = actionResult.Success,
                    ErrorMessage = actionResult.ErrorMessage
                }, ct);
            }

            results.Add(result);

            // Stop processing if rule says so
            if (rule.StopProcessing)
            {
                _logger.LogInformation(
                    "Rule '{Rule}' has stopProcessing=true, skipping remaining rules",
                    rule.Name);
                break;
            }
        }

        return results;
    }

    private IEnumerable<RuleConfig> GetApplicableRules(AccountConfig account)
    {
        // Filter rules that apply to this account
        return _allRules.Where(rule =>
            rule.ApplyToAccounts.Contains("ALL", StringComparer.OrdinalIgnoreCase) ||
            rule.ApplyToAccounts.Contains(account.Name, StringComparer.Ordinal));
    }
}
```

## Testing Rules

### Dry-Run Mode

```bash
# Test rules without making changes
dotnet run -- --dry-run

# Test specific account
dotnet run -- --dry-run --account work
```

### Example Rule Test Cases

```csharp
[Fact]
public void Evaluate_FromContains_MatchesSenderEmail()
{
    // Arrange
    var message = new MimeMessage();
    message.From.Add(new MailboxAddress("Test", "test@example.com"));

    var condition = new RuleCondition
    {
        From = new TextCondition { Contains = "example.com" }
    };

    var evaluator = new ConditionEvaluator(Mock.Of<ILogger<ConditionEvaluator>>());

    // Act
    var result = evaluator.Evaluate(message, condition);

    // Assert
    Assert.True(result);
}
```

## Best Practices

1. **Rule Priority**: Assign lower numbers to more important rules
2. **Stop Processing**: Use `stopProcessing: true` to prevent rule conflicts
3. **Test First**: Use dry-run mode before applying new rules
4. **Specific Before General**: Put specific rules before catch-all rules
5. **Backup Important Actions**: Enable backups for delete/move operations
6. **Monitor Logs**: Review audit logs regularly to verify rules work as expected
