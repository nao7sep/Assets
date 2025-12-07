# taskKiller Data Specification

Complete specification for taskKiller's data structures and file formats.

## Table of Contents

1. [Task Data Model](#task-data-model)
2. [Note Data Model](#note-data-model)
3. [AI Sections in Note Content](#ai-sections-in-note-content)
4. [File Attachment Data Model](#file-attachment-data-model)
5. [Directory Structure](#directory-structure)
6. [File Format: taskKiller1](#file-format-taskkiller1)
7. [Text Escaping](#text-escaping)
8. [Backward Compatibility](#backward-compatibility)
9. [CRUD Operations](#crud-operations)
10. [Parsing Algorithms](#parsing-algorithms)

---

## Task Data Model

### TaskState Enum

```csharp
public enum TaskState
{
    Later = 0,    // Default state
    Soon,
    Now,
    Done,
    Cancelled
}
```

**Important**: `Later` has index 0, making it the default value for uninitialized enums.

### TaskDto

```csharp
public sealed record TaskDto
{
    public required Guid Guid { get; init; }
    public required DateTime CreationUtc { get; init; }
    public required string Content { get; init; }
    public required TaskState State { get; init; }
    public DateTime? HandlingUtc { get; init; }      // Set when Done/Cancelled
    public Guid? RepeatedGuid { get; init; }         // Original task if repeated
    public required long OrderingUtc { get; init; }  // Sort key (higher = first, descending)
    public bool IsSpecial { get; init; }             // Persisted highlight flag
    public DateTime? HiddenUntilUtc { get; init; }   // Task hidden until this time
    public IReadOnlyList<NoteDto> Notes { get; init; } = [];
}
```

### Field Details

| Field | Storage | Description |
|-------|---------|-------------|
| `Guid` | File: `Guid:{value}` | Format "D" (hyphenated), must match filename |
| `CreationUtc` | File: `CreationUtc:{ticks}` | `DateTime.UtcNow.Ticks` at creation |
| `Content` | File: `Content:{escaped}` | [Escaped](#text-escaping) text, can be multi-line |
| `State` | File: `State:{value}` | Actual state name (not "Queued") |
| `HandlingUtc` | File: `HandlingUtc:{ticks}` | Optional, set when task completed |
| `RepeatedGuid` | File: `RepeatedGuid:{guid}` | Optional, links to original task |
| `OrderingUtc` | File: `OrderingUtc:{ticks}` | `DateTime.UtcNow.Ticks`, higher = first |
| `IsSpecial` | File: `IsSpecial:True` | Optional, persisted highlight state |
| `HiddenUntilUtc` | File: `HiddenUntilUtc:{ticks}` | Optional, hide task until this time |
| `Notes` | File: additional paragraphs | Parsed from same file |

### Auxiliary Files (Legacy/Optional)

By default, the new app writes all data to the main task file. For backward compatibility with tools that read separate files, an option can enable writing to auxiliary files as well.

#### States/{GUID}.txt (Optional)

Single line containing: `Later`, `Soon`, or `Now`

- **Only for active tasks** (not Done/Cancelled)
- **Takes precedence** over `State` field in task file when read by old app
- **New app default**: Does not create these files
- **Option**: `WriteSeparateStateFiles = true` to create for legacy tool compatibility

#### Ordering/{GUID}.txt (Optional)

Single line containing the ordering value as `long` (ticks)

- **Takes precedence** over `OrderingUtc` field in task file when read by old app
- **New app default**: Does not create these files
- **Option**: `WriteSeparateOrderingFiles = true` to create for legacy tool compatibility

#### IsSpecial/{GUID}.txt (New, Optional)

Single line containing: `True` or `False`

- **New app default**: Does not create (writes to main file instead)
- **Option**: `WriteSeparateIsSpecialFiles = true` to create separate files

### Filename Validation

Task filename must match its GUID (case-insensitive):

```csharp
bool IsValidFilename(string filename, Guid taskGuid) =>
    string.Equals(
        Path.GetFileNameWithoutExtension(filename),
        taskGuid.ToString("D"),
        StringComparison.OrdinalIgnoreCase);
```

Files failing validation are skipped during loading.

---

## Note Data Model

### NoteDto

```csharp
public sealed record NoteDto
{
    public required Guid Guid { get; init; }
    public required DateTime CreationUtc { get; init; }
    public required string Content { get; init; }

    // Parsed AI sections (computed from Content)
    public IReadOnlyList<NotePart> Parts => NoteContentParser.Parse(Content);
}
```

### Storage

Notes are embedded in the parent task file as additional paragraphs (separated by blank lines).

| Field | Storage | Description |
|-------|---------|-------------|
| `Guid` | `Guid:{value}` | Format "D" (hyphenated) |
| `CreationUtc` | `CreationUtc:{ticks}` | `DateTime.UtcNow.Ticks` at creation |
| `Content` | `Content:{escaped}` | [Escaped](#text-escaping) text |

### Example Task File with Notes

```
Format:taskKiller1
Guid:a1b2c3d4-e5f6-7890-abcd-ef1234567890
CreationUtc:638372841234567890
Content:Buy groceries
State:Queued

Guid:b2c3d4e5-f6a7-8901-bcde-f23456789012
CreationUtc:638372842000000000
Content:Check expiry dates

Guid:c3d4e5f6-a7b8-9012-cdef-345678901234
CreationUtc:638372843000000000
Content:Shopping list:\n- Milk\n- Eggs
```

### Display Order

Notes are sorted by `CreationUtc` ascending (oldest first).

---

## AI Sections in Note Content

Note content may contain AI response quotations marked with `@`. This is a semantic convention used by tk2Text for HTML export. taskKiller itself does NOT parse these markers.

> **For modern implementations**: Parse and render AI sections with distinct styling.

### Format Rules

1. A **paragraph starting with `@`** begins an AI response section
2. The section continues until:
   - A **paragraph ending with `@`** (explicit close), OR
   - The **end of the note content** (implicit close)
3. A note can contain **multiple** AI sections
4. Text before the first `@` and between sections is user content

### Examples

**Single AI section (implicit close):**
```
User question here

@ Here is the AI response.
It continues until the end.
```

**Single AI section (explicit close):**
```
User question

@ AI response here @

More user text after.
```

**Multiple AI sections:**
```
First question

@ First AI response @

Second question

@ Second AI response @

Final notes.
```

### NotePart Model

```csharp
public enum NotePartType
{
    UserContent,
    AiResponse
}

public sealed record NotePart
{
    public required NotePartType Type { get; init; }
    public required string Content { get; init; }
}
```

### Parsing Algorithm (from tk2Text)

Based on `tk2Text/iHtmlPageGenerator.cs` lines 91-122:

```csharp
public static class NoteContentParser
{
    public static IReadOnlyList<NotePart> Parse(string content)
    {
        var paragraphs = SplitIntoParagraphs(content);
        var parts = new List<NotePart>();

        for (int i = 0; i < paragraphs.Length; i++)
        {
            if (!paragraphs[i].StartsWith("@"))
            {
                parts.Add(new NotePart
                {
                    Type = NotePartType.UserContent,
                    Content = paragraphs[i]
                });
            }
            else
            {
                // Find closing @ (paragraph ending with @)
                int endIndex = -1;
                for (int j = i; j < paragraphs.Length; j++)
                {
                    if (paragraphs[j].EndsWith("@"))
                    {
                        endIndex = j;
                        break;
                    }
                }

                // If no explicit close, continue to end
                if (endIndex < 0)
                    endIndex = paragraphs.Length - 1;

                // Join paragraphs and trim @ markers
                var aiContent = string.Join("\n\n",
                    paragraphs[i..(endIndex + 1)]).Trim('@', ' ');

                parts.Add(new NotePart
                {
                    Type = NotePartType.AiResponse,
                    Content = aiContent
                });

                i = endIndex; // Loop will increment
            }
        }

        return parts;
    }
}
```

---

## File Attachment Data Model

### FileAttachmentDto

```csharp
public sealed record FileAttachmentDto
{
    public required Guid Guid { get; init; }
    public required string RelativePath { get; init; }  // e.g., "Files/document.pdf" or "Files/1/document.pdf"
    public Guid? ParentGuid { get; init; }              // Task GUID, Note GUID, or null (list-level)
    public required DateTime AttachedAtUtc { get; init; }
    public required DateTime ModifiedAtUtc { get; init; }  // Source file's last modified time
}
```

### Attachment Targets

| ParentGuid | Meaning |
|------------|---------|
| Task GUID | Attached to specific task |
| Note GUID | Attached to specific note |
| `null` | Attached to task list itself |

### Storage: Files/Info.txt

INI-style format with sections:

```ini
[Files/document.pdf]
Guid:d4e5f6a7-b8c9-0123-def4-567890123456
ParentGuid:a1b2c3d4-e5f6-7890-abcd-ef1234567890
AttachedAt:2024-01-15T10:30:00.0000000Z
ModifiedAt:2024-01-10T08:15:30.1234567Z

[Files/1/image.png]
Guid:e5f6a7b8-c9d0-1234-ef56-789012345678
ParentGuid:
AttachedAt:2024-01-16T14:20:00.0000000Z
ModifiedAt:2024-01-16T14:00:00.0000000Z
```

### Field Formats

| Field | Format | Example |
|-------|--------|---------|
| Section header | `[relative-path]` | `[Files/document.pdf]` or `[Files/1/document.pdf]` |
| Guid | "D" format | `d4e5f6a7-b8c9-0123-def4-567890123456` |
| ParentGuid | "D" format or empty | Empty = attached to task list |
| AttachedAt | ISO8601 "O" format | `2024-01-15T10:30:00.0000000Z` |
| ModifiedAt | ISO8601 "O" format | Source file's `LastWriteTimeUtc` |

### Conflict Resolution

When attaching a file with a name that already exists:

1. Try `Files/{filename}`
2. If exists (or filename is `Info.txt`), try `Files/1/{filename}`
3. Continue incrementing: `Files/2/{filename}`, `Files/3/{filename}`, etc.

---

## Directory Structure

### Single Task List

```
[TaskListDirectory]/
├── Settings.txt              # Contains "Title:{name}"
├── Tasks/
│   ├── {GUID-1}.txt
│   ├── {GUID-2}.txt
│   └── ...
├── States/
│   ├── {GUID-1}.txt          # Only for active tasks
│   └── ...
├── Ordering/
│   ├── {GUID-1}.txt
│   └── ...
└── Files/
    ├── Info.txt              # Attachment metadata
    ├── document.pdf
    ├── 1/
    │   └── document.pdf      # Conflict resolution
    └── ...
```

### Multiple Task Lists (Subtask Lists)

```
[ParentDirectory]/
├── MainProject/
│   ├── Settings.txt
│   ├── Tasks/
│   └── ...
├── SubProject-A/
│   ├── Settings.txt
│   └── ...
└── SubProject-B/
    ├── Settings.txt
    └── ...
```

A directory is recognized as a task list if it contains `Settings.txt` with a `Title:` key.

---

## File Format: taskKiller1

### General Rules

- **Encoding**: UTF-8 (no BOM required, handle BOM on read)
- **Line endings**: Write CRLF (`\r\n`), tolerate LF (`\n`) on read
- **Format**: Key-value pairs, one per line: `Key:Value`
- **Paragraphs**: Separated by one or more blank lines
- **First paragraph**: Task metadata
- **Subsequent paragraphs**: Notes (one per paragraph)

### Required Fields (First Paragraph)

| Field | Description |
|-------|-------------|
| `Format` | Must be `taskKiller1` |
| `Guid` | Task identifier, must match filename |
| `CreationUtc` | Creation timestamp in ticks |
| `Content` | Task content ([escaped](#text-escaping)) |
| `State` | `Queued`, `Done`, or `Cancelled` |

### Optional Fields (First Paragraph)

| Field | Description |
|-------|-------------|
| `HandlingUtc` | Completion timestamp in ticks |
| `RepeatedGuid` | Original task GUID if repeated |
| `OrderingUtc` | Legacy ordering (prefer Ordering/{GUID}.txt) |

### Note Paragraph Fields

| Field | Description |
|-------|-------------|
| `Guid` | Note identifier |
| `CreationUtc` | Creation timestamp in ticks |
| `Content` | Note content ([escaped](#text-escaping)) |

### Complete Example

```
Format:taskKiller1
Guid:a1b2c3d4-e5f6-7890-abcd-ef1234567890
CreationUtc:638372841234567890
Content:Implement user authentication\nIncluding OAuth2 support
State:Queued
RepeatedGuid:98765432-dcba-0987-fedc-ba9876543210

Guid:note-guid-1234-5678-9012-345678901234
CreationUtc:638372850000000000
Content:Started research on OAuth providers

Guid:note-guid-5678-9012-3456-789012345678
CreationUtc:638372860000000000
Content:@ Claude suggested using PKCE flow for SPAs @
```

---

## Text Escaping

Task and note `Content` fields use C-style escaping.

### Escape Sequences

| Character | Escaped | Code Point |
|-----------|---------|------------|
| Tab | `\t` | 0x09 |
| Carriage Return | `\r` | 0x0D |
| Line Feed | `\n` | 0x0A |
| Backslash | `\\` | 0x5C |

**Only these four sequences are supported.** Other backslash sequences (e.g., `\x`, `\u`) are invalid.

### Implementation

```csharp
public static class TextEscaping
{
    public static string Escape(string? text)
    {
        if (string.IsNullOrEmpty(text))
            return text ?? string.Empty;

        var builder = new StringBuilder(text.Length);
        foreach (char c in text)
        {
            builder.Append(c switch
            {
                '\t' => @"\t",
                '\r' => @"\r",
                '\n' => @"\n",
                '\\' => @"\\",
                _ => c.ToString()
            });
        }
        return builder.ToString();
    }

    public static string Unescape(string? text)
    {
        if (string.IsNullOrEmpty(text))
            return text ?? string.Empty;

        var builder = new StringBuilder(text.Length);
        for (int i = 0; i < text.Length; i++)
        {
            if (text[i] == '\\' && i + 1 < text.Length)
            {
                char? unescaped = text[i + 1] switch
                {
                    't' => '\t',
                    'r' => '\r',
                    'n' => '\n',
                    '\\' => '\\',
                    _ => null
                };

                if (unescaped.HasValue)
                {
                    builder.Append(unescaped.Value);
                    i++;
                    continue;
                }
                // Invalid sequence - could throw or pass through
                throw new FormatException($"Invalid escape sequence: \\{text[i + 1]}");
            }
            builder.Append(text[i]);
        }
        return builder.ToString();
    }
}
```

### Usage Rules

- **Escape** when writing `Content` values to files
- **Unescape** when reading `Content` values from files
- **Do NOT escape** other fields (Guid, timestamps, state, etc.)

---

## Backward Compatibility

### Design Philosophy

The new app writes all task data to the main task file by default. This simplifies disk access and keeps related data together. Separate auxiliary files are optional and only needed for compatibility with legacy tools.

### New Fields

The following fields are new and will be ignored by the old WPF app:

| Field | Format | Description |
|-------|--------|-------------|
| `HiddenUntilUtc` | `HiddenUntilUtc:{ticks}` | Hide task until this UTC time |
| `IsSpecial` | `IsSpecial:True` | Persistent highlight flag |

The old app's `ParseKeyValueCollection` puts all key-value pairs into a dictionary, and `LoadTask` only accesses known keys. **Unknown keys are silently ignored.**

⚠️ **Data Loss Warning**: If you edit a task in the old app and save it, `HiddenUntilUtc` and `IsSpecial` will be lost. This is acceptable as these are convenience features, not core data.

### State Field (New Behavior)

**Old app behavior**:
- Writes `State:Queued` to task file for active tasks
- Writes actual state to `States/{GUID}.txt`
- On read: checks `States/{GUID}.txt` first, falls back to task file

**New app behavior (default)**:
- Writes actual state name directly to task file: `State:Later`, `State:Soon`, etc.
- Does NOT create `States/{GUID}.txt`

**Compatibility**:
- Old app reading new files: Sees `State:Later` (not "Queued"), uses it directly ✅
- New app reading old files: Checks separate file first, falls back to task file ✅

### OrderingUtc Field (New Behavior)

**Old app behavior (before update)**:
- Writes ordering to `Ordering/{GUID}.txt`
- On read: checks `Ordering/{GUID}.txt` first, falls back to task file
- Sort: **Ascending** (lower values first)
- New tasks: `GetMinOrderingUtc() - 1` (top of list)
- Postpone: `DateTime.UtcNow.Ticks` (bottom of list)

**New app behavior** (and updated old app):
- Writes ordering to task file: `OrderingUtc:{ticks}`
- Does NOT create `Ordering/{GUID}.txt` by default
- Sort: **Descending** (higher values first)
- New tasks / Prioritize: `DateTime.UtcNow.Ticks` (top of list)
- Postpone: `GetMinOrderingUtc() - 1` (bottom of list)

**Compatibility**:
- Old app (updated) reading new files: Uses task file value ✅
- New app reading old files: Checks separate file first ✅

### Optional: Write Auxiliary Files

For compatibility with legacy tools that read separate files:

```csharp
public sealed record TaskKillerWriteOptions
{
    /// <summary>Write state to States/{GUID}.txt in addition to task file.</summary>
    public bool WriteSeparateStateFiles { get; init; } = false;

    /// <summary>Write ordering to Ordering/{GUID}.txt in addition to task file.</summary>
    public bool WriteSeparateOrderingFiles { get; init; } = false;

    /// <summary>Write IsSpecial to IsSpecial/{GUID}.txt in addition to task file.</summary>
    public bool WriteSeparateIsSpecialFiles { get; init; } = false;
}
```

### Reading Priority (Backward Compatible)

When reading, always check auxiliary files first for maximum compatibility:

```csharp
TaskState ResolveState(string? taskFileState, string? statesFileContent)
{
    // 1. Check States/{GUID}.txt first (old app format)
    if (!string.IsNullOrEmpty(statesFileContent))
        if (Enum.TryParse<TaskState>(statesFileContent.Trim(), out var state))
            return state;

    // 2. Check State field in task file
    if (!string.IsNullOrEmpty(taskFileState) && taskFileState != "Queued")
        if (Enum.TryParse<TaskState>(taskFileState, out var state))
            return state;

    // 3. Default to Later
    return TaskState.Later;
}

long ResolveOrderingUtc(long? taskFileValue, string? orderingFileContent)
{
    // 1. Check Ordering/{GUID}.txt first (old app format)
    if (!string.IsNullOrEmpty(orderingFileContent))
        if (long.TryParse(orderingFileContent.Trim(), out var ordering))
            return ordering;

    // 2. Check OrderingUtc field in task file
    if (taskFileValue.HasValue)
        return taskFileValue.Value;

    // 3. Default (imported task, needs reassignment)
    return -1;
}
```

### Ordering Mechanism

#### Sort Direction: Descending

Tasks are sorted by `OrderingUtc` in **descending** order (higher values first).

```csharp
var sortedTasks = tasks.OrderByDescending(t => t.OrderingUtc);
```

#### Task Operations Summary

| Operation | OrderingUtc Value | Result |
|-----------|-------------------|--------|
| Create Task | `DateTime.UtcNow.Ticks` | Top of list |
| Repeat Task | `DateTime.UtcNow.Ticks` | Top of list |
| Prioritize (Ctrl+P) | `DateTime.UtcNow.Ticks` | Top of list |
| Postpone (Space) | `GetMinOrderingUtc() - 1` | Bottom of list |
| Up/Down Arrow | Swap values with neighbor | Move one position |
| Export Task | Preserved (ordering file moved) | Same position in destination |
| Legacy Import (OrderingUtc < 0) | `DateTime.UtcNow.Ticks` | Top of list |

#### New Task / Prioritize Ordering

New tasks and prioritized tasks get `DateTime.UtcNow.Ticks`, which ensures they appear at the top:

```csharp
newTask.OrderingUtc = DateTime.UtcNow.Ticks;
```

This enables consistent ordering across multiple task lists in a merged view.

#### Postpone Ordering

Postponed tasks get `GetMinOrderingUtc() - 1`, which sends them to the bottom:

```csharp
public static long GetMinOrderingUtcForPostpone()
{
    var tasks = Tasks.Where(t => t.OrderingUtc >= 0);

    if (tasks.Any())
        return tasks.Min(t => t.OrderingUtc) - 1;
    else
        return DateTime.UtcNow.Ticks;
}
```

#### Moving Tasks (Slide Algorithm)

To move task B after task D in sequence `[A, B, C, D, E]`:

```
Before: A(50), B(40), C(30), D(20), E(10)  // descending order
Move B after D:
  - B gets D's value (20)
  - C gets B's old value (40)
  - D gets C's old value (30)
After:  A(50), C(40), D(30), B(20), E(10)
```

```csharp
void MoveTaskAfter(List<TaskDto> tasks, int sourceIndex, int targetIndex)
{
    if (sourceIndex == targetIndex || sourceIndex == targetIndex + 1)
        return; // No-op

    var movingTask = tasks[sourceIndex];
    var oldValue = movingTask.OrderingUtc;

    // Slide values
    if (sourceIndex < targetIndex)
    {
        // Moving down: shift items up
        for (int i = sourceIndex; i < targetIndex; i++)
        {
            var nextValue = tasks[i + 1].OrderingUtc;
            tasks[i] = tasks[i] with { OrderingUtc = nextValue };
        }
        tasks[targetIndex] = movingTask with { OrderingUtc = oldValue };
    }
    else
    {
        // Moving up: shift items down
        for (int i = sourceIndex; i > targetIndex + 1; i--)
        {
            var prevValue = tasks[i - 1].OrderingUtc;
            tasks[i] = tasks[i] with { OrderingUtc = prevValue };
        }
        tasks[targetIndex + 1] = movingTask with { OrderingUtc = tasks[targetIndex].OrderingUtc };
    }
}
```

### Handling Legacy Tasks with Negative OrderingUtc

Normally, when a task is exported (moved) to another list, its ordering file is moved along with it, preserving its position. However, for backward compatibility with older data or manual file transfers, tasks may have negative `OrderingUtc` values.

When loading tasks with negative `OrderingUtc`:

1. Collect them separately during load
2. Sort them by `CreationUtc` ascending (older tasks first)
3. Assign new values: `DateTime.UtcNow.Ticks` sequentially (older tasks get earlier times, maintaining creation order)
4. Persist immediately so they have valid ordering

> **Note**: `IsSpecial` is NOT automatically set for imported tasks. `IsSpecial` is purely a user-controlled temporary emphasis (toggled via Ctrl+Space in taskKiller).

---

## CRUD Operations

### Task Operations

#### Create Task

```csharp
async Task CreateTaskAsync(TaskDto task, TaskKillerWriteOptions? options = null)
{
    options ??= new TaskKillerWriteOptions();

    // 1. Assign ordering if not set
    if (task.OrderingUtc <= 0)
        task = task with { OrderingUtc = DateTime.UtcNow.Ticks };

    // 2. Ensure GUID doesn't collide
    var path = GetTaskFilePath(task.Guid);
    while (File.Exists(path))
    {
        task = task with { Guid = Guid.NewGuid() };
        path = GetTaskFilePath(task.Guid);
    }

    // 3. Write task file (includes State, OrderingUtc, IsSpecial, HiddenUntilUtc)
    await WriteTaskFileAsync(path, task);

    // 4. Optionally write auxiliary files for legacy tool compatibility
    if (options.WriteSeparateStateFiles && task.State is TaskState.Later or TaskState.Soon or TaskState.Now)
        await WriteStatesFileAsync(task.Guid, task.State);

    if (options.WriteSeparateOrderingFiles)
        await WriteOrderingFileAsync(task.Guid, task.OrderingUtc);

    if (options.WriteSeparateIsSpecialFiles && task.IsSpecial)
        await WriteIsSpecialFileAsync(task.Guid, task.IsSpecial);
}
```

#### Read Task

```csharp
async Task<TaskDto> ReadTaskAsync(Guid guid)
{
    var path = GetTaskFilePath(guid);
    var content = await File.ReadAllTextAsync(path, Encoding.UTF8);

    var paragraphs = SplitIntoParagraphs(content);
    var taskProps = ParseKeyValuePairs(paragraphs[0]);

    // Validate
    if (taskProps["Format"] != "taskKiller1")
        throw new FormatException("Invalid format");
    if (!IsValidFilename(Path.GetFileName(path), guid))
        throw new FormatException("GUID mismatch");

    // Resolve state and ordering (check auxiliary files first for backward compat)
    var state = await ResolveStateAsync(guid, taskProps.GetValueOrDefault("State"));
    var ordering = await ResolveOrderingAsync(guid, taskProps.GetValueOrDefault("OrderingUtc"));

    // Read new fields (ignored by old app, may not exist)
    var isSpecial = taskProps.GetValueOrDefault("IsSpecial") == "True";
    DateTime? hiddenUntilUtc = taskProps.TryGetValue("HiddenUntilUtc", out var h) && long.TryParse(h, out var hTicks)
        ? new DateTime(hTicks, DateTimeKind.Utc) : null;

    // Parse notes
    var notes = paragraphs.Skip(1)
        .Select(p => ParseNote(p))
        .OrderBy(n => n.CreationUtc)
        .ToList();

    return new TaskDto
    {
        Guid = guid,
        CreationUtc = new DateTime(long.Parse(taskProps["CreationUtc"]), DateTimeKind.Utc),
        Content = taskProps["Content"].UnescapeC(),
        State = state,
        OrderingUtc = ordering,
        IsSpecial = isSpecial,  // Only from file, not auto-set
        HiddenUntilUtc = hiddenUntilUtc,
        HandlingUtc = taskProps.TryGetValue("HandlingUtc", out var hu) ? new DateTime(long.Parse(hu), DateTimeKind.Utc) : null,
        RepeatedGuid = taskProps.TryGetValue("RepeatedGuid", out var rg) ? Guid.Parse(rg) : null,
        Notes = notes
    };
}
```

#### Read All Active Tasks

```csharp
async Task<IReadOnlyList<TaskDto>> GetActiveTasksAsync()
{
    var tasks = new List<TaskDto>();
    var legacyTasks = new List<TaskDto>();  // OrderingUtc < 0 (legacy/imported without ordering file)

    foreach (var file in Directory.GetFiles(TasksDirectory, "*.txt"))
    {
        var task = await ReadTaskAsync(file);

        if (task.State is TaskState.Done or TaskState.Cancelled)
            continue;

        if (task.OrderingUtc < 0)
            legacyTasks.Add(task);
        else
            tasks.Add(task);
    }

    // Reassign ordering for legacy tasks with negative OrderingUtc
    // Sort by CreationUtc ascending so older tasks get earlier (smaller) ordering values
    // This preserves creation order in the final descending-sorted list
    if (legacyTasks.Count > 0)
    {
        legacyTasks.Sort((a, b) => a.CreationUtc.CompareTo(b.CreationUtc));

        foreach (var task in legacyTasks)
        {
            var newOrdering = DateTime.UtcNow.Ticks;
            var reassigned = task with { OrderingUtc = newOrdering };
            tasks.Add(reassigned);

            // Persist immediately so they have valid ordering
            await UpdateTaskAsync(reassigned);
        }
    }

    // Sort descending (higher OrderingUtc = first)
    return tasks.OrderByDescending(t => t.OrderingUtc).ToList();
}
```

#### Update Task

```csharp
async Task UpdateTaskAsync(TaskDto task, TaskKillerWriteOptions? options = null)
{
    options ??= new TaskKillerWriteOptions();

    // Write task file (all data in one place)
    await WriteTaskFileAsync(GetTaskFilePath(task.Guid), task);

    // Optionally write auxiliary files
    if (options.WriteSeparateStateFiles)
    {
        if (task.State is TaskState.Later or TaskState.Soon or TaskState.Now)
            await WriteStatesFileAsync(task.Guid, task.State);
        else
            DeleteStatesFileIfExists(task.Guid);
    }

    if (options.WriteSeparateOrderingFiles)
        await WriteOrderingFileAsync(task.Guid, task.OrderingUtc);

    if (options.WriteSeparateIsSpecialFiles)
        await WriteIsSpecialFileAsync(task.Guid, task.IsSpecial);
}
```

#### Delete Task

```csharp
async Task DeleteTaskAsync(Guid guid)
{
    File.Delete(GetTaskFilePath(guid));
    // Clean up auxiliary files if they exist
    DeleteStatesFileIfExists(guid);
    DeleteOrderingFileIfExists(guid);
    DeleteIsSpecialFileIfExists(guid);
    // Note: Orphaned attachments are NOT automatically deleted
}
```

### Note Operations

Notes modify the parent task file:

```csharp
async Task AddNoteAsync(Guid taskId, NoteDto note)
{
    var task = await ReadTaskAsync(taskId);
    var updatedTask = task with { Notes = task.Notes.Append(note).ToList() };
    await WriteTaskFileAsync(GetTaskFilePath(taskId), updatedTask);
}

async Task DeleteNoteAsync(Guid taskId, Guid noteId)
{
    var task = await ReadTaskAsync(taskId);
    var updatedTask = task with { Notes = task.Notes.Where(n => n.Guid != noteId).ToList() };
    await WriteTaskFileAsync(GetTaskFilePath(taskId), updatedTask);
}
```

### Attachment Operations

```csharp
async Task AttachFileAsync(string sourcePath, Guid? parentGuid)
{
    // 1. Determine target path (with conflict resolution)
    var filename = Path.GetFileName(sourcePath);
    var targetPath = ResolveAttachmentPath(filename);

    // 2. Copy file
    File.Copy(sourcePath, targetPath, overwrite: false);

    // 3. Append to Info.txt
    var entry = new FileAttachmentDto
    {
        Guid = Guid.NewGuid(),
        RelativePath = GetRelativePath(targetPath),
        ParentGuid = parentGuid,
        AttachedAtUtc = DateTime.UtcNow,
        ModifiedAtUtc = File.GetLastWriteTimeUtc(sourcePath)
    };
    await AppendToInfoFileAsync(entry);
}
```

---

## Parsing Algorithms

### Split into Paragraphs

```csharp
public static string[] SplitIntoParagraphs(string text)
{
    var paragraphs = new List<string>();
    var current = new StringBuilder();

    foreach (var line in text.Split('\n'))
    {
        var trimmedLine = line.TrimEnd('\r');

        if (string.IsNullOrEmpty(trimmedLine))
        {
            if (current.Length > 0)
            {
                paragraphs.Add(current.ToString());
                current.Clear();
            }
        }
        else
        {
            if (current.Length > 0)
                current.Append('\n');
            current.Append(trimmedLine);
        }
    }

    if (current.Length > 0)
        paragraphs.Add(current.ToString());

    return paragraphs.ToArray();
}
```

### Parse Key-Value Pairs

```csharp
public static Dictionary<string, string> ParseKeyValuePairs(string paragraph)
{
    var dict = new Dictionary<string, string>();

    foreach (var line in paragraph.Split('\n'))
    {
        var colonIndex = line.IndexOf(':');
        if (colonIndex > 0)
        {
            var key = line[..colonIndex];
            var value = line[(colonIndex + 1)..];
            dict[key] = value;  // Later keys overwrite
        }
    }

    return dict;
}
```

### Parse Info.txt (Attachments)

```csharp
public static List<FileAttachmentDto> ParseInfoFile(string content)
{
    var entries = new List<FileAttachmentDto>();
    string? currentPath = null;
    var currentProps = new Dictionary<string, string>();

    foreach (var line in content.Split('\n'))
    {
        var trimmed = line.TrimEnd('\r');

        if (string.IsNullOrWhiteSpace(trimmed))
            continue;

        if (trimmed.StartsWith('[') && trimmed.EndsWith(']'))
        {
            // Save previous entry
            if (currentPath != null)
                entries.Add(CreateAttachment(currentPath, currentProps));

            currentPath = trimmed[1..^1];
            currentProps.Clear();
        }
        else if (currentPath != null)
        {
            var colonIndex = trimmed.IndexOf(':');
            if (colonIndex > 0)
                currentProps[trimmed[..colonIndex]] = trimmed[(colonIndex + 1)..];
        }
    }

    // Save last entry
    if (currentPath != null)
        entries.Add(CreateAttachment(currentPath, currentProps));

    return entries;
}
```

---

## Summary

| Aspect | Key Points |
|--------|------------|
| **Tasks** | GUID-named `.txt` files, `taskKiller1` format |
| **State** | Written to task file (actual value, not "Queued"); auxiliary file optional |
| **Ordering** | **Descending sort** (higher = first); new/prioritize = `UtcNow.Ticks` (top); postpone = `Min - 1` (bottom) |
| **IsSpecial** | New persistent field for highlight; written to task file |
| **HiddenUntilUtc** | New field to hide task until specified time |
| **Notes** | Embedded paragraphs in task file, sorted by CreationUtc ascending |
| **AI Sections** | `@` markers in notes, parsed per tk2Text algorithm |
| **Attachments** | `Files/` directory + `Info.txt` metadata |
| **Encoding** | UTF-8, CRLF (write), tolerate LF (read) |
| **Escaping** | `\t`, `\r`, `\n`, `\\` for Content fields only |
| **Compatibility** | Read legacy formats (auxiliary files); write to main file by default |
| **Auxiliary Files** | Optional via `TaskKillerWriteOptions` for legacy tool compatibility |
