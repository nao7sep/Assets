# Architecture and Design

## Design Principles

### Single-Purpose Simplicity
- One entry per window/process
- No tabs, no multi-document complexity
- User can run multiple processes if needed

### Workflow Focus
- States guide user through a clear process
- Prevent chaos by encouraging completion
- Archive only after cooldown period

### Data Integrity
- DTO/Domain Model separation per Playbook
- Validation during mapping
- Graceful handling of corrupt files

## Application Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        BrainLog                             │
│                    (Avalonia UI App)                        │
│   - Views (XAML)                                            │
│   - ViewModels (MVVM, data binding)                         │
│   - App entry point, DI setup                               │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      BrainLogCore                           │
│               (Business Logic Library)                      │
│   - Domain Models (Entry, Prompt, State)                    │
│   - DTOs and Mappers                                        │
│   - Services (EntryService, AiService, FileWatchService)    │
│   - Interfaces for all services                             │
└─────────────────────────────────────────────────────────────┘
```

## MVVM Pattern

### View Layer (BrainLog)
- XAML views with minimal code-behind
- Data binding to ViewModels
- Commands for user actions

### ViewModel Layer (BrainLog)
- `MainViewModel` - orchestrates all UI state
- `EntryViewModel` - represents single entry for editing
- `EntryListItemViewModel` - list item display
- `PromptViewModel` - prompt management
- Exception handling at command boundaries (per Playbook)

### Model Layer (BrainLogCore)
- Pure domain models (no UI dependencies)
- Services implement business logic
- Interfaces enable DI and testing

## State Machine

### Entry States

```
┌──────────┐    Check    ┌──────────┐   Handle   ┌──────────┐   Archive  ┌──────────┐
│ Editing  │ ─────────►  │ Checked  │ ─────────► │ Handled  │ ─────────► │ Archived │
└──────────┘             └──────────┘            └──────────┘            └──────────┘
     ▲                        │                       │                       │
     │                        │                       │                       │
     │         Revert         │        Revert         │       Unarchive       │
     └────────────────────────┴───────────────────────┴───────────────────────┘
```

### State Enum

```csharp
public enum EntryState
{
    Editing = 0,
    Checked = 1,
    Handled = 2,
    Archived = 3
}
```

### Transition Rules
- **Forward**: Editing → Checked → Handled → Archived (sequential only)
- **Backward**: Any state can revert to previous state
- **Archive Button**: Appears only after configurable cooldown (default 24 hours)
- **Editing**: Allowed in any state (with confirmation dialog for non-Editing states)

### Editing Non-Editing Entries
- User can edit content in any state without changing state
- Save button shows confirmation dialog for Checked/Handled/Archived entries
- Timestamps are NOT updated when editing without state change
- This prevents minor typo fixes from disrupting timestamps

## Three-Content Model

The app tracks three content states for conflict detection:

| Property | Description |
|----------|-------------|
| `OriginalContent` | Content when file was loaded (in-memory only) |
| `EditorContent` | Current content in the editor (bound to UI) |
| `FileContent` | Actual file content (updated by FileWatcher) |

### Conflict Detection Logic

```
On FileWatcher event:
  1. Read new file content → FileContent
  2. Compare OriginalContent vs FileContent
     - If equal: No external change, check EditorContent vs OriginalContent for unsaved changes
     - If different: External change detected
       - If EditorContent == FileContent: "File updated externally, matches your edits"
       - If EditorContent != FileContent: Show conflict warning (border highlight + status bar)

On Save:
  1. Write EditorContent to file
  2. Set OriginalContent = EditorContent
  3. FileWatcher will update FileContent (ignored via self-write flag)
```

## Dependency Injection

### Service Registration

```csharp
services.AddSingleton<IEntryService, EntryService>();
services.AddSingleton<IFileWatchService, FileWatchService>();
services.AddSingleton<IAutoSaveService, AutoSaveService>();
services.AddSingleton<IPromptService, PromptService>();
services.AddSingleton<IAiService, OpenAiService>();  // or GeminiService based on config
services.AddSingleton<IConfigurationService, ConfigurationService>();
```

### Interface-Based Design
- All services registered by interface
- Enables testing with mocks
- Allows swapping implementations (e.g., AI providers)

## Error Handling Strategy

### Service Layer
- Allow exceptions to bubble up naturally
- Throw when operations would fail silently
- Use built-in .NET exceptions where appropriate

### ViewModel Layer (Boundary)
- Catch exceptions in command handlers
- Display user-friendly error dialogs
- Log errors for debugging

### File Operations
- Corrupt file: Show dialog with path and content, skip file
- Missing file: Remove from list, warn user if currently editing
- Write failure: Show error dialog, keep editor state intact
