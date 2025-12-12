# Email and SMTP Specification

## Overview

IdeaDump sends elaborated ideas to confirmed recipients via SMTP. This document specifies email delivery, retry logic, and SMTP configuration.

## SMTP Configuration

### Configuration Hierarchy

1. **Channel-level**: Each channel can have custom SMTP settings
2. **System-level**: Default SMTP for channels without custom config

### SmtpConfiguration Model

```csharp
public class SmtpConfiguration
{
    public string Server { get; set; } = string.Empty;
    public int Port { get; set; } = 587;
    public bool UseSsl { get; set; } = true;
    public string Username { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
    public string FromEmail { get; set; } = string.Empty;
    public string FromName { get; set; } = string.Empty;
}
```

### Supported SMTP Providers

| Provider | Server | Port | SSL | Notes |
|----------|--------|------|-----|-------|
| SendGrid | smtp.sendgrid.net | 587 | Yes | Recommended |
| Office 365 | smtp.office365.com | 587 | Yes | Good reliability |
| Gmail | smtp.gmail.com | 587 | Yes | Requires app password |
| Mailgun | smtp.mailgun.org | 587 | Yes | Transactional emails |

## Email Service Interface

```csharp
public interface IEmailService
{
    Task<EmailResult> SendEmailAsync(
        string to,
        string subject,
        string body,
        SmtpConfiguration smtpConfig,
        CancellationToken ct = default);

    Task<EmailResult> SendBulkEmailAsync(
        List<string> recipients,
        string subject,
        string body,
        SmtpConfiguration smtpConfig,
        CancellationToken ct = default);

    Task<bool> TestSmtpConnectionAsync(SmtpConfiguration config);
}

public class EmailResult
{
    public bool Success { get; set; }
    public string? ErrorMessage { get; set; }
    public List<string> FailedRecipients { get; set; } = new();
}
```

## Email Implementation

### SmtpEmailService

```csharp
public class SmtpEmailService : IEmailService
{
    private readonly ILogger<SmtpEmailService> _logger;

    public async Task<EmailResult> SendEmailAsync(
        string to,
        string subject,
        string body,
        SmtpConfiguration smtpConfig,
        CancellationToken ct)
    {
        try
        {
            using var client = new SmtpClient(smtpConfig.Server, smtpConfig.Port);
            client.EnableSsl = smtpConfig.UseSsl;
            client.Credentials = new NetworkCredential(
                smtpConfig.Username,
                smtpConfig.Password);

            var message = new MailMessage
            {
                From = new MailAddress(smtpConfig.FromEmail, smtpConfig.FromName),
                Subject = subject,
                Body = body,
                IsBodyHtml = true
            };
            message.To.Add(to);

            await client.SendMailAsync(message, ct);

            return new EmailResult { Success = true };
        }
        catch (SmtpException ex)
        {
            _logger.LogError(ex, $"SMTP error sending to {to}");
            return new EmailResult
            {
                Success = false,
                ErrorMessage = $"SMTP error: {ex.Message}",
                FailedRecipients = new List<string> { to }
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, $"Error sending email to {to}");
            return new EmailResult
            {
                Success = false,
                ErrorMessage = ex.Message,
                FailedRecipients = new List<string> { to }
            };
        }
    }

    public async Task<EmailResult> SendBulkEmailAsync(
        List<string> recipients,
        string subject,
        string body,
        SmtpConfiguration smtpConfig,
        CancellationToken ct)
    {
        var failed = new List<string>();
        string? lastError = null;

        foreach (var recipient in recipients)
        {
            var result = await SendEmailAsync(recipient, subject, body, smtpConfig, ct);
            if (!result.Success)
            {
                failed.Add(recipient);
                lastError = result.ErrorMessage;
            }

            // Rate limiting: delay between sends
            await Task.Delay(100, ct);
        }

        return new EmailResult
        {
            Success = failed.Count == 0,
            ErrorMessage = lastError,
            FailedRecipients = failed
        };
    }
}
```

## Email Templates

### Elaborated Idea Email

```csharp
public class EmailTemplateService
{
    public string BuildElaboratedIdeaEmail(
        IdeaEntry idea,
        Channel channel,
        string unsubscribeUrl)
    {
        return $@"
<!DOCTYPE html>
<html>
<head>
    <meta charset=""utf-8"">
    <meta name=""viewport"" content=""width=device-width, initial-scale=1.0"">
    <style>
        body {{
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
            line-height: 1.6;
            color: #333;
            max-width: 600px;
            margin: 0 auto;
            padding: 20px;
        }}
        .header {{
            border-bottom: 3px solid #007bff;
            padding-bottom: 20px;
            margin-bottom: 30px;
        }}
        .header h1 {{
            margin: 0;
            font-size: 28px;
            color: #007bff;
        }}
        .channel-name {{
            font-size: 14px;
            color: #666;
            margin-top: 5px;
        }}
        .content {{
            margin: 30px 0;
        }}
        .original-idea {{
            background: #f8f9fa;
            border-left: 4px solid #007bff;
            padding: 15px;
            margin: 20px 0;
            font-style: italic;
        }}
        .footer {{
            border-top: 1px solid #ddd;
            padding-top: 20px;
            margin-top: 40px;
            font-size: 12px;
            color: #666;
        }}
        .footer a {{
            color: #007bff;
            text-decoration: none;
        }}
        h2 {{ color: #007bff; }}
        h3 {{ color: #333; }}
        ul, ol {{ padding-left: 25px; }}
    </style>
</head>
<body>
    <div class=""header"">
        <h1>💡 {HttpUtility.HtmlEncode(idea.EmailSubject)}</h1>
        <div class=""channel-name"">From channel: {HttpUtility.HtmlEncode(channel.Name)}</div>
    </div>

    <div class=""original-idea"">
        <strong>Original Idea:</strong><br>
        {HttpUtility.HtmlEncode(idea.RawText)}
    </div>

    <div class=""content"">
        {idea.EmailBody}
    </div>

    <div class=""footer"">
        <p>You are receiving this because you subscribed to the channel ""<strong>{HttpUtility.HtmlEncode(channel.Name)}</strong>"".</p>
        <p>
            <a href=""{unsubscribeUrl}"">Unsubscribe from this channel</a> |
            Powered by IdeaDump
        </p>
    </div>
</body>
</html>";
    }
}
```

### Confirmation Email

```csharp
public string BuildConfirmationEmail(
    ChannelRecipient recipient,
    Channel channel,
    string confirmUrl)
{
    return $@"
<!DOCTYPE html>
<html>
<head>
    <meta charset=""utf-8"">
    <style>
        body {{ font-family: Arial, sans-serif; line-height: 1.6; padding: 20px; }}
        .container {{ max-width: 600px; margin: 0 auto; }}
        .button {{
            display: inline-block;
            padding: 12px 24px;
            background: #007bff;
            color: white;
            text-decoration: none;
            border-radius: 4px;
            margin: 20px 0;
        }}
    </style>
</head>
<body>
    <div class=""container"">
        <h2>Confirm Your Subscription</h2>
        <p>You have been added to the IdeaDump channel: <strong>{HttpUtility.HtmlEncode(channel.Name)}</strong></p>
        <p>By confirming, you will receive elaborated idea reports via email.</p>
        <p>
            <a href=""{confirmUrl}"" class=""button"">Confirm Subscription</a>
        </p>
        <p><small>Or copy this link: {confirmUrl}</small></p>
        <p><small>This link expires in 7 days.</small></p>
        <hr>
        <p><small>If you did not expect this email, you can safely ignore it.</small></p>
    </div>
</body>
</html>";
}
```

## Background Email Service

### EmailSendingService

```csharp
public class EmailSendingService : BackgroundService
{
    private readonly IServiceProvider _services;
    private readonly ILogger<EmailSendingService> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ProcessElaboratedIdeasAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error sending emails");
            }

            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }

    private async Task ProcessElaboratedIdeasAsync(CancellationToken ct)
    {
        using var scope = _services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        var emailService = scope.ServiceProvider.GetRequiredService<IEmailService>();
        var templateService = scope.ServiceProvider.GetRequiredService<EmailTemplateService>();

        var elaboratedIdeas = await context.IdeaEntries
            .Include(i => i.Channel)
                .ThenInclude(c => c.Recipients)
            .Where(i => i.Status == IdeaEntryStatus.Elaborated)
            .OrderBy(i => i.ElaboratedAt)
            .Take(5)
            .ToListAsync(ct);

        foreach (var idea in elaboratedIdeas)
        {
            await SendIdeaEmailAsync(idea, context, emailService, templateService, ct);
        }
    }

    private async Task SendIdeaEmailAsync(
        IdeaEntry idea,
        ApplicationDbContext context,
        IEmailService emailService,
        EmailTemplateService templateService,
        CancellationToken ct)
    {
        try
        {
            idea.Status = IdeaEntryStatus.Sending;
            await context.SaveChangesAsync();

            // Get confirmed recipients
            var recipients = idea.Channel.Recipients
                .Where(r => r.IsConfirmed && !r.IsUnsubscribed)
                .ToList();

            if (!recipients.Any())
            {
                _logger.LogWarning($"No confirmed recipients for idea {idea.Id}");
                idea.Status = IdeaEntryStatus.Sent; // Mark as sent even with no recipients
                await context.SaveChangesAsync();
                return;
            }

            // Build email content
            var smtpConfig = GetSmtpConfigFromChannel(idea.Channel);
            var failed = new List<string>();

            foreach (var recipient in recipients)
            {
                var unsubscribeUrl = $"{_config["App:BaseUrl"]}/unsubscribe?token={recipient.UnsubscribeToken}";
                var emailBody = templateService.BuildElaboratedIdeaEmail(idea, idea.Channel, unsubscribeUrl);

                var result = await emailService.SendEmailAsync(
                    recipient.Email,
                    idea.EmailSubject!,
                    emailBody,
                    smtpConfig,
                    ct);

                if (!result.Success)
                {
                    failed.Add(recipient.Email);
                }

                await Task.Delay(100, ct); // Rate limiting
            }

            if (failed.Any())
            {
                idea.Status = IdeaEntryStatus.Failed;
                idea.ErrorMessage = $"Failed to send to: {string.Join(", ", failed)}";
                idea.NextRetryAt = DateTime.UtcNow.AddMinutes(Math.Pow(2, idea.RetryCount));
                idea.RetryCount++;
            }
            else
            {
                idea.Status = IdeaEntryStatus.Sent;
                idea.EmailSentAt = DateTime.UtcNow;
            }

            await context.SaveChangesAsync();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, $"Error sending email for idea {idea.Id}");

            idea.Status = IdeaEntryStatus.Failed;
            idea.ErrorMessage = ex.Message;
            idea.NextRetryAt = DateTime.UtcNow.AddMinutes(Math.Pow(2, idea.RetryCount));
            idea.RetryCount++;

            await context.SaveChangesAsync();
        }
    }
}
```

## Retry Logic

### Exponential Backoff

```csharp
public class RetryService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ProcessFailedIdeasAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in retry service");
            }

            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
    }

    private async Task ProcessFailedIdeasAsync(CancellationToken ct)
    {
        using var scope = _services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();

        var failedIdeas = await context.IdeaEntries
            .Where(i => i.Status == IdeaEntryStatus.Failed
                     && i.NextRetryAt <= DateTime.UtcNow
                     && i.RetryCount < i.MaxRetries)
            .ToListAsync(ct);

        foreach (var idea in failedIdeas)
        {
            if (idea.ElaboratedAt != null)
            {
                // Failed at email stage, retry email
                idea.Status = IdeaEntryStatus.Elaborated;
            }
            else
            {
                // Failed at elaboration stage, retry elaboration
                idea.Status = IdeaEntryStatus.New;
            }

            await context.SaveChangesAsync();
        }

        // Handle abandoned ideas
        var abandoned = await context.IdeaEntries
            .Where(i => i.Status == IdeaEntryStatus.Failed
                     && i.RetryCount >= i.MaxRetries)
            .ToListAsync(ct);

        foreach (var idea in abandoned)
        {
            idea.Status = IdeaEntryStatus.Abandoned;
            await context.SaveChangesAsync();

            await NotifyAdminOfAbandonedIdeaAsync(idea);
        }
    }
}
```

## Error Handling

### Common SMTP Errors

| Error | Description | Retry? |
|-------|-------------|--------|
| Network timeout | Connection timed out | Yes |
| Authentication failed | Invalid credentials | No |
| Recipient rejected | Invalid email address | No |
| Rate limit exceeded | Too many requests | Yes |
| Server unavailable | SMTP server down | Yes |

### Error Classification

```csharp
public static class SmtpErrorClassifier
{
    public static bool IsTransient(SmtpException ex)
    {
        return ex.StatusCode == SmtpStatusCode.ServiceNotAvailable
            || ex.StatusCode == SmtpStatusCode.MailboxBusy
            || ex.StatusCode == SmtpStatusCode.TransactionFailed;
    }

    public static bool IsPermanent(SmtpException ex)
    {
        return ex.StatusCode == SmtpStatusCode.MailboxUnavailable
            || ex.StatusCode == SmtpStatusCode.UserNotLocalWillForward
            || ex.StatusCode == SmtpStatusCode.MailboxNameNotAllowed;
    }
}
```

---

**Next Document**: [08-ui-blazor-design.md](08-ui-blazor-design.md)
