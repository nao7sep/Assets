# Data Models and Configuration

## Domain Model

### Entry (Domain Model)

```csharp
public class Entry
{
    public Guid Id { get; init; }
    public string? Title { get; set; }
    public string? Draft { get; set; }
    public string Content { get; set; } = string.Empty;
    public EntryState State { get; set; }
    public DateTime CreatedUtc { get; init; }
    public DateTime? CheckedUtc { get; set; }
    public DateTime? HandledUtc { get; set; }
    public DateTime? ArchivedUtc { get; set; }
}
```

### Business Rules
- `Id`: Generated on first save, never changes
- `Title`: Nullable, AI-generated on first save, can be manually edited or regenerated
- `Draft`: Nullable, original input text
- `Content`: The refined/final text (if Draft is null, Content is also the original)
- `State`: Current workflow state
- `CreatedUtc`: Set on first save, never changes
- Timestamps: Set when transitioning TO that state

### Display Title Logic
```csharp
public string DisplayTitle => Title ?? GetFirstNCharacters(Draft ?? Content, 50) ?? "(Empty)";
```

## Data Transfer Object

### EntryDto (for JSON serialization)

```csharp
public class EntryDto
{
    [JsonPropertyName("id")]
    public Guid? Id { get; set; }

    [JsonPropertyName("title")]
    public string? Title { get; set; }

    [JsonPropertyName("draft")]
    public string[]? Draft { get; set; }  // Multiline as array for diff-friendly JSON

    [JsonPropertyName("content")]
    public string[]? Content { get; set; }  // Multiline as array for diff-friendly JSON

    [JsonPropertyName("state")]
    public string? State { get; set; }

    [JsonPropertyName("createdUtc")]
    public DateTime? CreatedUtc { get; set; }

    [JsonPropertyName("checkedUtc")]
    public DateTime? CheckedUtc { get; set; }

    [JsonPropertyName("handledUtc")]
    public DateTime? HandledUtc { get; set; }

    [JsonPropertyName("archivedUtc")]
    public DateTime? ArchivedUtc { get; set; }
}
```

### JSON Format (Entry File)

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "title": "Meeting Ideas for Q1",
  "draft": [
    "First line of draft",
    "Second line of draft",
    "Third line with typos and errors"
  ],
  "content": [
    "First line of content",
    "Second line of content",
    "Third line, corrected and refined"
  ],
  "state": "Checked",
  "createdUtc": "2025-12-13T10:30:00Z",
  "checkedUtc": "2025-12-13T14:45:00Z",
  "handledUtc": null,
  "archivedUtc": null
}
```

## Mapper

### EntryMapper

```csharp
public static class EntryMapper
{
    public static Entry ToDomain(EntryDto dto, string filePath)
    {
        // Validation
        if (dto.Id is null || dto.Id == Guid.Empty)
            throw new InvalidOperationException($"Invalid entry: missing Id in {filePath}");

        if (dto.CreatedUtc is null)
            throw new InvalidOperationException($"Invalid entry: missing CreatedUtc in {filePath}");

        if (!Enum.TryParse<EntryState>(dto.State, out var state))
            throw new InvalidOperationException($"Invalid entry: invalid State '{dto.State}' in {filePath}");

        // At least Draft or Content must have value
        var draft = dto.Draft is not null ? string.Join(Environment.NewLine, dto.Draft) : null;
        var content = dto.Content is not null ? string.Join(Environment.NewLine, dto.Content) : null;

        if (string.IsNullOrEmpty(draft) && string.IsNullOrEmpty(content))
            throw new InvalidOperationException($"Invalid entry: both Draft and Content are empty in {filePath}");

        return new Entry
        {
            Id = dto.Id.Value,
            Title = dto.Title,
            Draft = draft,
            Content = content ?? draft!,  // If Content is empty, use Draft
            State = state,
            CreatedUtc = dto.CreatedUtc.Value,
            CheckedUtc = dto.CheckedUtc,
            HandledUtc = dto.HandledUtc,
            ArchivedUtc = dto.ArchivedUtc
        };
    }

    public static EntryDto ToDto(Entry entry)
    {
        return new EntryDto
        {
            Id = entry.Id,
            Title = entry.Title,
            Draft = entry.Draft?.Split(Environment.NewLine),
            Content = entry.Content.Split(Environment.NewLine),
            State = entry.State.ToString(),
            CreatedUtc = entry.CreatedUtc,
            CheckedUtc = entry.CheckedUtc,
            HandledUtc = entry.HandledUtc,
            ArchivedUtc = entry.ArchivedUtc
        };
    }
}
```

## File Naming Convention

```
{yyyy-MM-dd}-{guid}.json
```

- Date is UTC date from `CreatedUtc`
- Examples:
  - `2025-12-13-a1b2c3d4-e5f6-7890-abcd-ef1234567890.json`
  - `2025-12-14-b2c3d4e5-f6a7-8901-bcde-f23456789012.json`

## Configuration Files

### appsettings.json (Read-Only, Defaults)

Location: Next to executable

```json
{
  "Storage": {
    "EntriesDirectory": null,
    "PortableMode": false
  },
  "AutoSave": {
    "Enabled": true,
    "IntervalSeconds": 30,
    "MaxBackups": 300
  },
  "Workflow": {
    "ArchiveCooldownMinutes": 1440,
    "ConfirmSaveOnNonEditingState": true
  },
  "Notifications": {
    "SuccessDisplaySeconds": 3
  },
  "Ai": {
    "TitleGenerationPrompt": "Generate a concise title (maximum 60 characters) for the following text. Return JSON with a single property 'title' containing only the title text.\n\nText:\n{CONTENT}"
  },
  "DefaultPrompt": {
    "Name": "Fix Transcription Errors",
    "SystemPrompt": "You are a text editor assistant. Fix obvious errors, typos, and voice transcription mistakes while preserving the original meaning and intent. Do not add, remove, or rephrase content beyond error correction.",
    "UserPrompt": "Please fix errors in the following text and explain what you changed.\n\nText:\n{DRAFT}\n\nReturn JSON with two properties:\n- CONTENT: the corrected text\n- ANALYSIS: brief explanation of changes made"
  },
  "AiServices": {
    "OpenAi": {
      "ApiKey": null,
      "Model": "gpt-4o-mini",
      "Endpoint": "https://api.openai.com/v1"
    },
    "Gemini": {
      "ApiKey": null,
      "Model": "gemini-1.5-flash"
    }
  }
}
```

### usersettings.json (User Preferences)

Location:
- Portable mode: Next to executable
- Standard mode: `%APPDATA%/BrainLog/` (Windows) or `~/Library/Application Support/BrainLog/` (macOS)

```json
{
  "SelectedAiService": "OpenAi",
  "WindowPosition": {
    "X": 100,
    "Y": 100,
    "Width": 1200,
    "Height": 800
  }
}
```

### prompts.json (User-Managed Prompts)

Location: Same as usersettings.json

```json
{
  "Prompts": [
    {
      "Id": "d1e2f3a4-b5c6-7890-1234-567890abcdef",
      "Name": "Fix Transcription Errors",
      "SystemPrompt": "You are a text editor assistant...",
      "UserPrompt": "Please fix errors in the following text..."
    },
    {
      "Id": "e2f3a4b5-c6d7-8901-2345-67890abcdef1",
      "Name": "Improve Clarity",
      "SystemPrompt": "You are a writing assistant...",
      "UserPrompt": "Please improve clarity of the following text..."
    }
  ]
}
```

### Prompt Model

```csharp
public class Prompt
{
    public Guid Id { get; init; }
    public string Name { get; set; } = string.Empty;
    public string SystemPrompt { get; set; } = string.Empty;
    public string UserPrompt { get; set; } = string.Empty;
}
```

## Storage Locations

### Standard Mode (Default)

| Item | Windows | macOS |
|------|---------|-------|
| Entries | `%APPDATA%/BrainLog/Entries/` | `~/Library/Application Support/BrainLog/Entries/` |
| Auto-saves | `%APPDATA%/BrainLog/AutoSaves/` | `~/Library/Application Support/BrainLog/AutoSaves/` |
| usersettings.json | `%APPDATA%/BrainLog/` | `~/Library/Application Support/BrainLog/` |
| prompts.json | `%APPDATA%/BrainLog/` | `~/Library/Application Support/BrainLog/` |

### Portable Mode

When `appsettings.json` contains `"PortableMode": true` OR a file named `portable.marker` exists next to executable:

| Item | Location |
|------|----------|
| Entries | `{ExeDir}/Entries/` |
| Auto-saves | `{ExeDir}/AutoSaves/` |
| usersettings.json | `{ExeDir}/` |
| prompts.json | `{ExeDir}/` |

## Auto-Save File Format

Location: `{AutoSaveDir}/{yyyy-MM-dd-HH-mm-ss}-{entryGuid}.json`

Content: Same as EntryDto JSON format

Example: `2025-12-13-14-30-45-a1b2c3d4-e5f6-7890-abcd-ef1234567890.json`
