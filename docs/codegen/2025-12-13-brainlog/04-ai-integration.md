# AI Integration

## Overview

BrainLog integrates with AI services for two purposes:
1. **Title Generation**: Auto-generate entry titles on first save
2. **Prompt Execution**: Run user-selected prompts for text correction/analysis

## AI Service Abstraction

### IAiService Interface

```csharp
public interface IAiService
{
    string ServiceName { get; }
    bool IsConfigured { get; }
    Task<AiResponse> ExecutePromptAsync(
        string systemPrompt,
        string userPrompt,
        CancellationToken cancellationToken = default);
}
```

### AiResponse

```csharp
public class AiResponse
{
    public bool Success { get; init; }
    public string? Content { get; init; }      // Maps to CONTENT in JSON response
    public string? Analysis { get; init; }     // Maps to ANALYSIS in JSON response
    public string? Title { get; init; }        // Maps to title in JSON response (for title generation)
    public string? ErrorMessage { get; init; }
    public string? RawResponse { get; init; }  // For debugging
}
```

## AI Service Implementations

### OpenAI Service

```csharp
public class OpenAiService : IAiService
{
    public string ServiceName => "OpenAI";

    public bool IsConfigured =>
        !string.IsNullOrEmpty(_options.ApiKey) &&
        !string.IsNullOrEmpty(_options.Model);
}
```

**JSON Schema Request (OpenAI)**:
```json
{
  "model": "gpt-4o-mini",
  "messages": [...],
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "correction_result",
      "strict": true,
      "schema": {
        "type": "object",
        "properties": {
          "CONTENT": { "type": "string" },
          "ANALYSIS": { "type": "string" }
        },
        "required": ["CONTENT", "ANALYSIS"],
        "additionalProperties": false
      }
    }
  }
}
```

### Gemini Service

```csharp
public class GeminiService : IAiService
{
    public string ServiceName => "Gemini";

    public bool IsConfigured =>
        !string.IsNullOrEmpty(_options.ApiKey) &&
        !string.IsNullOrEmpty(_options.Model);
}
```

**JSON Schema Request (Gemini)**:
Uses `response_mime_type: "application/json"` with `response_schema` for structured output.

## AI Service Factory

```csharp
public interface IAiServiceFactory
{
    IReadOnlyList<IAiService> GetConfiguredServices();
    IAiService? GetService(string serviceName);
}

public class AiServiceFactory : IAiServiceFactory
{
    private readonly IEnumerable<IAiService> _services;

    public IReadOnlyList<IAiService> GetConfiguredServices() =>
        _services.Where(s => s.IsConfigured).ToList();

    public IAiService? GetService(string serviceName) =>
        _services.FirstOrDefault(s => s.ServiceName == serviceName && s.IsConfigured);
}
```

## Title Generation

### When Title is Generated
- On **first save** of a new entry
- Async operation (does not block save)
- User sees truncated content as title until AI completes
- If AI fails, title remains null (truncated content continues to display)

### Title Generation Flow

```
1. User saves new entry
2. Entry saved immediately with Title = null
3. Background: Call AI with TitleGenerationPrompt
4. If success: Update entry file with new title
5. If failure: Log error, leave Title = null
```

### Title Prompt Template

From `appsettings.json`:
```
Generate a concise title (maximum 60 characters) for the following text.
Return JSON with a single property 'title' containing only the title text.

Text:
{CONTENT}
```

**Expected Response**:
```json
{
  "title": "Meeting Ideas for Q1 Planning"
}
```

### Manual Title Regeneration
- User can click "Generate" button next to title field
- Uses same prompt and flow
- Overwrites existing title

## Prompt Execution

### Prompt Variables

| Variable | Replaced With |
|----------|---------------|
| `{DRAFT}` | Current Draft text |
| `{CONTENT}` | Current Content text (or Draft if Content is empty) |

### Execution Flow

```
1. User selects prompt from dropdown
2. User clicks "Run" button
3. UI disabled, progress dialog shown with Cancel button
4. Replace variables in UserPrompt
5. Call AI service with SystemPrompt and processed UserPrompt
6. Parse JSON response for CONTENT and ANALYSIS
7. On success: Populate Content field with CONTENT, Analysis area with ANALYSIS
8. On failure: Show error in progress dialog
9. On cancel: Restore original state, close dialog
```

### Response Parsing

```csharp
public AiResponse ParseResponse(string json)
{
    try
    {
        using var doc = JsonDocument.Parse(json);
        var root = doc.RootElement;

        return new AiResponse
        {
            Success = true,
            Content = root.TryGetProperty("CONTENT", out var c) ? c.GetString() : null,
            Analysis = root.TryGetProperty("ANALYSIS", out var a) ? a.GetString() : null,
            Title = root.TryGetProperty("title", out var t) ? t.GetString() : null,
            RawResponse = json
        };
    }
    catch (JsonException ex)
    {
        return new AiResponse
        {
            Success = false,
            ErrorMessage = $"Failed to parse AI response: {ex.Message}",
            RawResponse = json
        };
    }
}
```

## Prompt Management

### Default Prompt Initialization

On app startup:
1. Check if `prompts.json` exists
2. If empty or missing: Copy `DefaultPrompt` from `appsettings.json` as first prompt
3. If has prompts: Use as-is (even if default was deleted)

### Prompt CRUD Operations

| Operation | Behavior |
|-----------|----------|
| Add | Create new prompt with generated GUID |
| Edit | Modify name, system prompt, user prompt |
| Delete | Remove from list (no confirmation needed) |
| Reorder | Move up/down buttons |

### Prompt Management Window

- Modal dialog
- List of prompts on left
- Edit fields on right (Name, System Prompt, User Prompt)
- Buttons: Add, Delete, Move Up, Move Down, Save, Cancel

## Error Handling

### Network/API Errors
- Show error message in progress dialog
- User can close dialog and retry
- Original content unchanged

### Invalid JSON Response
- Parse as much as possible
- If CONTENT missing: Show error
- If ANALYSIS missing: Show warning, proceed with CONTENT only

### Timeout
- Configurable timeout (default: 60 seconds)
- On timeout: Treat as error, show message

## No AI Configured State

When no AI services are properly configured:
- AI-related buttons disabled
- Show info text: "Configure AI in appsettings.json"
- Or show button/link to open configuration help

### Checking Configuration

```csharp
// In ViewModel initialization
var configuredServices = _aiServiceFactory.GetConfiguredServices();
if (configuredServices.Count == 0)
{
    ShowAiConfigurationHelp = true;
    AiEnabled = false;
}
else
{
    AiServiceOptions = configuredServices.Select(s => s.ServiceName).ToList();
    SelectedAiService = _userSettings.SelectedAiService ?? configuredServices[0].ServiceName;
    AiEnabled = true;
}
```
