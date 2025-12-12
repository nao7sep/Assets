# AI Integration Specification

## Overview

IdeaDump uses AI services to elaborate fragmented user ideas into comprehensive, actionable reports. This document specifies the AI integration architecture, prompt engineering, API key management, and rate limiting.

## AI Service Abstraction

### IAiService Interface

```csharp
public interface IAiService
{
    /// <summary>
    /// Elaborate a raw idea into a comprehensive report
    /// </summary>
    Task<ElaborationResult> ElaborateIdeaAsync(
        string rawText,
        string instructions,
        string apiKey,
        CancellationToken cancellationToken);

    /// <summary>
    /// Test if an API key is valid
    /// </summary>
    Task<bool> ValidateApiKeyAsync(string apiKey);
}

public class ElaborationResult
{
    /// <summary>
    /// AI-generated concise title for the idea
    /// </summary>
    public string Title { get; set; } = string.Empty;

    /// <summary>
    /// Full elaborated content (markdown format)
    /// </summary>
    public string Content { get; set; } = string.Empty;

    /// <summary>
    /// Email subject line
    /// </summary>
    public string EmailSubject { get; set; } = string.Empty;

    /// <summary>
    /// Email body (HTML format)
    /// </summary>
    public string EmailBody { get; set; } = string.Empty;

    /// <summary>
    /// Was elaboration successful?
    /// </summary>
    public bool Success { get; set; }

    /// <summary>
    /// Error message if elaboration failed
    /// </summary>
    public string? ErrorMessage { get; set; }

    /// <summary>
    /// Token usage for cost tracking
    /// </summary>
    public TokenUsage? Usage { get; set; }
}

public class TokenUsage
{
    public int PromptTokens { get; set; }
    public int CompletionTokens { get; set; }
    public int TotalTokens { get; set; }
}
```

## Supported AI Providers

### 1. GitHub Models (Primary)

**Configuration**:
```json
{
  "Ai": {
    "Provider": "GitHub",
    "Endpoint": "https://models.inference.ai.azure.com",
    "Model": "gpt-4o",
    "MaxTokens": 4000,
    "Temperature": 0.7
  }
}
```

**Implementation**:
```csharp
public class GitHubModelsAiService : IAiService
{
    private readonly HttpClient _httpClient;
    private readonly IConfiguration _config;

    public async Task<ElaborationResult> ElaborateIdeaAsync(
        string rawText, string instructions, string apiKey, CancellationToken ct)
    {
        var endpoint = _config["Ai:Endpoint"];
        var model = _config["Ai:Model"];

        var request = new
        {
            model = model,
            messages = new[]
            {
                new { role = "system", content = instructions },
                new { role = "user", content = rawText }
            },
            temperature = double.Parse(_config["Ai:Temperature"]),
            max_tokens = int.Parse(_config["Ai:MaxTokens"])
        };

        _httpClient.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", apiKey);

        var response = await _httpClient.PostAsJsonAsync(
            $"{endpoint}/chat/completions", request, ct);

        if (!response.IsSuccessStatusCode)
        {
            var error = await response.Content.ReadAsStringAsync();
            return new ElaborationResult
            {
                Success = false,
                ErrorMessage = $"API error: {response.StatusCode} - {error}"
            };
        }

        var result = await response.Content.ReadFromJsonAsync<ChatCompletionResponse>(ct);
        return ParseElaboration(result);
    }
}
```

### 2. Azure OpenAI (Alternative)

**Configuration**:
```json
{
  "Ai": {
    "Provider": "AzureOpenAI",
    "Endpoint": "https://your-resource.openai.azure.com",
    "Deployment": "gpt-4o",
    "ApiVersion": "2024-02-01",
    "MaxTokens": 4000,
    "Temperature": 0.7
  }
}
```

### 3. OpenAI (Fallback)

**Configuration**:
```json
{
  "Ai": {
    "Provider": "OpenAI",
    "Endpoint": "https://api.openai.com/v1",
    "Model": "gpt-4o",
    "MaxTokens": 4000,
    "Temperature": 0.7
  }
}
```

## API Key Hierarchy

### Resolution Order

```csharp
public class ApiKeyResolver
{
    private readonly IConfiguration _config;
    private readonly IEncryptionService _encryption;

    public async Task<string> ResolveApiKeyAsync(Channel channel, ApplicationUser user)
    {
        // 1. Channel-specific API key (highest priority)
        if (!string.IsNullOrEmpty(channel.ApiKey))
        {
            return _encryption.Decrypt(channel.ApiKey);
        }

        // 2. User-wide API key
        if (!string.IsNullOrEmpty(user.ApiKey))
        {
            return _encryption.Decrypt(user.ApiKey);
        }

        // 3. System-wide API key (fallback)
        var systemKey = await _config.GetValueAsync("System.ApiKey");
        if (string.IsNullOrEmpty(systemKey))
        {
            throw new InvalidOperationException("No API key configured at any level");
        }

        return _encryption.Decrypt(systemKey);
    }

    public string GetApiKeySource(Channel channel, ApplicationUser user)
    {
        if (!string.IsNullOrEmpty(channel.ApiKey)) return "Channel";
        if (!string.IsNullOrEmpty(user.ApiKey)) return "User";
        return "System";
    }
}
```

## Prompt Engineering

### System Instructions

**Default System Prompt**:
```
You are an AI assistant helping to elaborate fragmented ideas into comprehensive, actionable reports.

Your task:
1. Analyze the user's rough idea
2. Research relevant context (conceptually, not live research)
3. Generate a detailed elaboration including:
   - Clear problem statement
   - Potential solutions or approaches
   - List of concrete, actionable tasks
   - Suggested timeline or schedule
   - Likelihood of success and potential risks
   - Resources needed
   - Key considerations

Output format:
Return a JSON object with the following structure:
{
  "title": "Concise title (max 10 words)",
  "content": "Full elaboration in markdown format",
  "emailSubject": "Engaging email subject line",
  "emailBody": "HTML-formatted email body"
}

Guidelines:
- Be specific and actionable
- Break down complex ideas into manageable steps
- Provide realistic timelines
- Highlight potential challenges
- Keep tone professional but encouraging
- Use markdown formatting for structure (headings, lists, bold)
- Aim for 500-1500 words depending on idea complexity
```

**Custom Instructions** (per channel):
Users can override with channel-specific instructions like:
```
Focus on technical feasibility and implementation details.
Assume the user is a software developer.
Prioritize code architecture and technology choices.
```

### Prompt Construction

```csharp
public class PromptBuilder
{
    public string BuildSystemPrompt(string? customInstructions, string defaultInstructions)
    {
        if (!string.IsNullOrEmpty(customInstructions))
        {
            // Combine default with custom
            return $"{defaultInstructions}\n\nAdditional Instructions:\n{customInstructions}";
        }

        return defaultInstructions;
    }

    public string BuildUserPrompt(string rawIdea)
    {
        return $@"Please elaborate the following idea:

{rawIdea}

Remember to provide a comprehensive analysis with actionable tasks, timeline, and success likelihood.";
    }
}
```

### Response Parsing

```csharp
public class ElaborationParser
{
    public ElaborationResult ParseResponse(string aiResponse)
    {
        try
        {
            // Try parsing as JSON first (structured output)
            var json = JsonSerializer.Deserialize<ElaborationJson>(aiResponse);

            if (json != null)
            {
                return new ElaborationResult
                {
                    Success = true,
                    Title = json.Title,
                    Content = json.Content,
                    EmailSubject = json.EmailSubject,
                    EmailBody = json.EmailBody
                };
            }
        }
        catch
        {
            // Fallback: Treat entire response as markdown content
        }

        // Extract title from first H1 heading
        var titleMatch = Regex.Match(aiResponse, @"^#\s+(.+)$", RegexOptions.Multiline);
        var title = titleMatch.Success
            ? titleMatch.Groups[1].Value
            : "Elaborated Idea";

        // Convert markdown to HTML for email
        var htmlBody = Markdig.Markdown.ToHtml(aiResponse);

        return new ElaborationResult
        {
            Success = true,
            Title = title,
            Content = aiResponse,
            EmailSubject = $"Idea: {title}",
            EmailBody = htmlBody
        };
    }
}
```

## Rate Limiting

### User-Level Rate Limits

**Daily Limit Check**:
```csharp
public class RateLimitService
{
    private readonly ApplicationDbContext _context;

    public async Task<RateLimitStatus> CheckRateLimitAsync(Guid userId)
    {
        var user = await _context.Users.FindAsync(userId);

        if (user == null)
            throw new InvalidOperationException("User not found");

        // Reset counter if new day
        if (user.LastLimitReset < DateTime.UtcNow.Date)
        {
            user.IdeasElaboratedToday = 0;
            user.LastLimitReset = DateTime.UtcNow.Date;
            await _context.SaveChangesAsync();
        }

        return new RateLimitStatus
        {
            Remaining = user.DailyIdeaLimit - user.IdeasElaboratedToday,
            Limit = user.DailyIdeaLimit,
            ResetsAt = DateTime.UtcNow.Date.AddDays(1)
        };
    }

    public async Task<bool> TryIncrementUsageAsync(Guid userId)
    {
        var status = await CheckRateLimitAsync(userId);

        if (status.Remaining <= 0)
            return false;

        var user = await _context.Users.FindAsync(userId);
        user!.IdeasElaboratedToday++;
        await _context.SaveChangesAsync();

        return true;
    }
}

public class RateLimitStatus
{
    public int Remaining { get; set; }
    public int Limit { get; set; }
    public DateTime ResetsAt { get; set; }

    public bool IsExceeded => Remaining <= 0;
}
```

### Admin Override

```csharp
public async Task SetUserDailyLimitAsync(Guid userId, int newLimit)
{
    var user = await _context.Users.FindAsync(userId);

    if (user == null)
        throw new InvalidOperationException("User not found");

    user.DailyIdeaLimit = newLimit;
    await _context.SaveChangesAsync();

    await _systemLog.LogAsync(LogLevel.Information, "Admin",
        $"User {user.Email} daily limit set to {newLimit}");
}
```

## Background Processing

### IdeaElaborationService

```csharp
public class IdeaElaborationService : BackgroundService
{
    private readonly IServiceProvider _services;
    private readonly ILogger<IdeaElaborationService> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Idea Elaboration Service started");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ProcessPendingIdeasAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing ideas");
            }

            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }

    private async Task ProcessPendingIdeasAsync(CancellationToken ct)
    {
        using var scope = _services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        var aiService = scope.ServiceProvider.GetRequiredService<IAiService>();
        var rateLimitService = scope.ServiceProvider.GetRequiredService<RateLimitService>();
        var apiKeyResolver = scope.ServiceProvider.GetRequiredService<ApiKeyResolver>();

        // Get ideas with status "New"
        var pendingIdeas = await context.IdeaEntries
            .Include(i => i.Channel)
            .Include(i => i.User)
            .Where(i => i.Status == IdeaEntryStatus.New)
            .OrderBy(i => i.CreatedAt)
            .Take(5) // Process 5 at a time
            .ToListAsync(ct);

        foreach (var idea in pendingIdeas)
        {
            await ProcessIdeaAsync(idea, context, aiService, rateLimitService, apiKeyResolver, ct);
        }
    }

    private async Task ProcessIdeaAsync(
        IdeaEntry idea,
        ApplicationDbContext context,
        IAiService aiService,
        RateLimitService rateLimitService,
        ApiKeyResolver apiKeyResolver,
        CancellationToken ct)
    {
        try
        {
            // Check rate limit
            var canProceed = await rateLimitService.TryIncrementUsageAsync(idea.UserId);
            if (!canProceed)
            {
                _logger.LogWarning($"Rate limit exceeded for user {idea.UserId}, idea {idea.Id}");
                idea.Status = IdeaEntryStatus.Failed;
                idea.ErrorMessage = "Daily rate limit exceeded";
                await context.SaveChangesAsync();
                return;
            }

            // Update status
            idea.Status = IdeaEntryStatus.Elaborating;
            idea.ElaborationStartedAt = DateTime.UtcNow;
            await context.SaveChangesAsync();

            // Log start
            await LogIdeaEventAsync(context, idea.Id, LogLevel.Information,
                "ElaborationStarted", "AI elaboration started");

            // Resolve API key
            var apiKey = await apiKeyResolver.ResolveApiKeyAsync(idea.Channel, idea.User);
            var keySource = apiKeyResolver.GetApiKeySource(idea.Channel, idea.User);
            idea.ApiKeySource = keySource;

            // Get AI instructions
            var instructions = !string.IsNullOrEmpty(idea.Channel.AiInstructions)
                ? idea.Channel.AiInstructions
                : await GetDefaultInstructionsAsync(context);

            // Call AI service
            var result = await aiService.ElaborateIdeaAsync(
                idea.RawText,
                instructions,
                apiKey,
                ct);

            if (result.Success)
            {
                // Store results
                idea.GeneratedTitle = result.Title;
                idea.ElaboratedContent = result.Content;
                idea.EmailSubject = result.EmailSubject;
                idea.EmailBody = result.EmailBody;
                idea.Status = IdeaEntryStatus.Elaborated;
                idea.ElaboratedAt = DateTime.UtcNow;

                await LogIdeaEventAsync(context, idea.Id, LogLevel.Information,
                    "ElaborationCompleted", $"AI elaboration completed (using {keySource} key)");
            }
            else
            {
                // Elaboration failed
                idea.Status = IdeaEntryStatus.Failed;
                idea.ErrorMessage = result.ErrorMessage;
                idea.NextRetryAt = DateTime.UtcNow.AddMinutes(Math.Pow(2, idea.RetryCount));
                idea.RetryCount++;

                await LogIdeaEventAsync(context, idea.Id, LogLevel.Error,
                    "ElaborationFailed", $"AI elaboration failed: {result.ErrorMessage}");
            }

            await context.SaveChangesAsync();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, $"Error processing idea {idea.Id}");

            idea.Status = IdeaEntryStatus.Failed;
            idea.ErrorMessage = ex.Message;
            idea.NextRetryAt = DateTime.UtcNow.AddMinutes(Math.Pow(2, idea.RetryCount));
            idea.RetryCount++;

            await context.SaveChangesAsync();

            await LogIdeaEventAsync(context, idea.Id, LogLevel.Error,
                "ElaborationError", $"Exception: {ex.Message}");
        }
    }
}
```

## Error Handling

### Transient Errors
- Network timeouts
- Rate limit errors from AI provider
- Temporary service unavailability

**Strategy**: Exponential backoff retry (up to 5 attempts)

### Permanent Errors
- Invalid API key
- Malformed response
- Content policy violation

**Strategy**: Mark as `Abandoned`, notify user

### Error Messages

```csharp
public static class AiErrorMessages
{
    public const string ApiKeyInvalid = "API key is invalid or has expired";
    public const string RateLimitExceeded = "AI service rate limit exceeded";
    public const string NetworkTimeout = "Connection to AI service timed out";
    public const string InvalidResponse = "AI service returned invalid response";
    public const string ContentPolicyViolation = "Content violates AI service policy";
    public const string QuotaExceeded = "AI service quota exceeded";
}
```

## Cost Tracking

### Token Usage Logging

```csharp
public async Task LogTokenUsageAsync(IdeaEntry idea, TokenUsage usage)
{
    await _systemLog.LogAsync(
        LogLevel.Information,
        "AI",
        $"Token usage - Prompt: {usage.PromptTokens}, Completion: {usage.CompletionTokens}, Total: {usage.TotalTokens}",
        userId: idea.UserId,
        channelId: idea.ChannelId,
        ideaEntryId: idea.Id,
        metadataJson: JsonSerializer.Serialize(new
        {
            promptTokens = usage.PromptTokens,
            completionTokens = usage.CompletionTokens,
            totalTokens = usage.TotalTokens,
            estimatedCost = CalculateCost(usage, "gpt-4o")
        })
    );
}

private decimal CalculateCost(TokenUsage usage, string model)
{
    // Example pricing for gpt-4o (adjust per provider)
    const decimal promptCostPer1K = 0.005m;
    const decimal completionCostPer1K = 0.015m;

    var promptCost = (usage.PromptTokens / 1000m) * promptCostPer1K;
    var completionCost = (usage.CompletionTokens / 1000m) * completionCostPer1K;

    return promptCost + completionCost;
}
```

## Testing

### Mock AI Service

```csharp
public class MockAiService : IAiService
{
    public Task<ElaborationResult> ElaborateIdeaAsync(
        string rawText, string instructions, string apiKey, CancellationToken ct)
    {
        return Task.FromResult(new ElaborationResult
        {
            Success = true,
            Title = "Mock Elaborated Idea",
            Content = $"**Original Idea**: {rawText}\n\n**Analysis**: This is a mock elaboration for testing.",
            EmailSubject = "Mock Email Subject",
            EmailBody = "<h1>Mock Email Body</h1><p>This is a test.</p>",
            Usage = new TokenUsage
            {
                PromptTokens = 100,
                CompletionTokens = 200,
                TotalTokens = 300
            }
        });
    }

    public Task<bool> ValidateApiKeyAsync(string apiKey)
    {
        return Task.FromResult(apiKey == "test-key");
    }
}
```

---

**Next Document**: [07-email-smtp-specification.md](07-email-smtp-specification.md)
