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
    public required long OrderingUtc { get; init; }  // Sort key (lower = first)
    public bool IsSpecial { get; init; }             // Runtime flag for imported tasks
    public IReadOnlyList<NoteDto> Notes { get; init; } = [];
}
```

### Field Details

| Field | Storage | Description |
|-------|---------|-------------|
| `Guid` | File: `Guid:{value}` | Format "D" (hyphenated), must match filename |
| `CreationUtc` | File: `CreationUtc:{ticks}` | `DateTime.UtcNow.Ticks` at creation |
| `Content` | File: `Content:{escaped}` | [Escaped](#text-escaping) text, can be multi-line |
| `State` | Complex | See [State Storage](#state-field-evolution) |
| `HandlingUtc` | File: `HandlingUtc:{ticks}` | Optional, set when task completed |
| `RepeatedGuid` | File: `RepeatedGuid:{guid}` | Optional, links to original task |
| `OrderingUtc` | Complex | See [Ordering Storage](#orderingutc-field-evolution) |
| `IsSpecial` | Not persisted | Runtime flag, true when `OrderingUtc < 0` on load |
| `Notes` | File: additional paragraphs | Parsed from same file |

### Auxiliary Files

#### States/{GUID}.txt

Single line containing: `Later`, `Soon`, or `Now`

- **Only exists** for active tasks (not Done/Cancelled)
- **Takes precedence** over `State` field in task file
- **Delete** when task becomes Done or Cancelled
- **Default**: If file doesn't exist and State is `Queued`, default to `Later`

#### Ordering/{GUID}.txt

Single line containing the ordering value as `long` (ticks)

- **Takes precedence** over `OrderingUtc` field in task file
- **Lower values** appear first in the list
- **Special value `-1`**: Indicates imported task needing order reassignment

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

### State Field Evolution

**Legacy format** (state stored directly in task file):
```
State:Later
State:Soon
State:Now
```

**Current format** (active states stored separately):
```
State:Queued
```
Actual state in `States/{GUID}.txt`: `Later`, `Soon`, or `Now`

#### Reading Priority

```csharp
TaskState ResolveState(string fileState, string? statesFileContent)
{
    // 1. Check States/{GUID}.txt first
    if (!string.IsNullOrEmpty(statesFileContent))
    {
        if (Enum.TryParse<TaskState>(statesFileContent.Trim(), out var state))
            return state;
    }

    // 2. Check State field in task file
    if (fileState != "Queued" && Enum.TryParse<TaskState>(fileState, out var fileStateEnum))
        return fileStateEnum;

    // 3. Default to Later
    return TaskState.Later;
}
```

#### Writing Rules

| State | Task File | States/{GUID}.txt |
|-------|-----------|-------------------|
| Later, Soon, Now | `State:Queued` | Write actual state |
| Done, Cancelled | `State:Done` or `State:Cancelled` | Delete if exists |

### OrderingUtc Field Evolution

**Legacy format** (in task file):
```
OrderingUtc:638372841234567890
```

**Current format** (stored separately):
`Ordering/{GUID}.txt` contains the value as a single line.

#### Reading Priority

```csharp
long ResolveOrderingUtc(long? fileValue, string? orderingFileContent)
{
    // 1. Check Ordering/{GUID}.txt first
    if (!string.IsNullOrEmpty(orderingFileContent))
    {
        if (long.TryParse(orderingFileContent.Trim(), out var ordering))
            return ordering;
    }

    // 2. Check OrderingUtc field in task file
    if (fileValue.HasValue)
        return fileValue.Value;

    // 3. Default (needs reassignment)
    return -1;
}
```

#### Writing Rules

- Always write ordering to `Ordering/{GUID}.txt`
- Do NOT write `OrderingUtc` to task file

### Handling OrderingUtc < 0 (Imported Tasks)

When loading tasks with negative `OrderingUtc`:

1. Collect them separately from normal tasks
2. Sort by their negative values (preserves relative import order)
3. Assign new values: `MinOrderingUtc - 1`, `MinOrderingUtc - 2`, etc.
4. Set `IsSpecial = true` for visual highlighting
5. Do NOT persist the new ordering immediately (preserves highlight until user edits)

---

## CRUD Operations

### Task Operations

#### Create Task

```csharp
async Task CreateTaskAsync(TaskDto task)
{
    // 1. Ensure GUID doesn't collide
    var path = GetTaskFilePath(task.Guid);
    while (File.Exists(path))
    {
        task = task with { Guid = Guid.NewGuid() };
        path = GetTaskFilePath(task.Guid);
    }

    // 2. Write task file
    await WriteTaskFileAsync(path, task);

    // 3. Write States file (if active)
    if (task.State is TaskState.Later or TaskState.Soon or TaskState.Now)
        await WriteStatesFileAsync(task.Guid, task.State);

    // 4. Write Ordering file
    await WriteOrderingFileAsync(task.Guid, task.OrderingUtc);
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

    // Resolve state and ordering from auxiliary files
    var state = await ResolveStateAsync(guid, taskProps["State"]);
    var ordering = await ResolveOrderingAsync(guid, taskProps.GetValueOrDefault("OrderingUtc"));

    // Parse notes
    var notes = paragraphs.Skip(1)
        .Select(p => ParseNote(p))
        .OrderBy(n => n.CreationUtc)
        .ToList();

    return new TaskDto { /* ... */ };
}
```

#### Read All Active Tasks

```csharp
async Task<IReadOnlyList<TaskDto>> GetActiveTasksAsync()
{
    var tasks = new List<TaskDto>();
    var pendingTasks = new List<TaskDto>();  // OrderingUtc < 0

    foreach (var file in Directory.GetFiles(TasksDirectory, "*.txt"))
    {
        var task = await ReadTaskAsync(file);

        if (task.State is TaskState.Done or TaskState.Cancelled)
            continue;

        if (task.OrderingUtc < 0)
            pendingTasks.Add(task);
        else
            tasks.Add(task);
    }

    // Reassign ordering for imported tasks
    pendingTasks.Sort((a, b) => a.OrderingUtc.CompareTo(b.OrderingUtc));
    foreach (var task in pendingTasks)
    {
        var newOrdering = GetMinOrderingUtc(tasks) - 1;
        tasks.Add(task with { OrderingUtc = newOrdering, IsSpecial = true });
    }

    return tasks.OrderBy(t => t.OrderingUtc).ToList();
}
```

#### Update Task

```csharp
async Task UpdateTaskAsync(TaskDto task)
{
    await WriteTaskFileAsync(GetTaskFilePath(task.Guid), task);

    if (task.State is TaskState.Later or TaskState.Soon or TaskState.Now)
        await WriteStatesFileAsync(task.Guid, task.State);
    else
        DeleteStatesFileIfExists(task.Guid);

    await WriteOrderingFileAsync(task.Guid, task.OrderingUtc);
}
```

#### Delete Task

```csharp
async Task DeleteTaskAsync(Guid guid)
{
    File.Delete(GetTaskFilePath(guid));
    DeleteStatesFileIfExists(guid);
    DeleteOrderingFileIfExists(guid);
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

### Get Minimum Ordering UTC

```csharp
public static long GetMinOrderingUtc(IEnumerable<TaskDto> tasks)
{
    var validTasks = tasks.Where(t => t.OrderingUtc >= 0);

    return validTasks.Any()
        ? validTasks.Min(t => t.OrderingUtc)
        : DateTime.UtcNow.Ticks;
}
```

---

## Summary

| Aspect | Key Points |
|--------|------------|
| **Tasks** | GUID-named `.txt` files, `taskKiller1` format |
| **State** | Active: `States/{GUID}.txt`, Final: in task file |
| **Ordering** | `Ordering/{GUID}.txt`, lower values first |
| **Notes** | Embedded paragraphs in task file, sorted by CreationUtc |
| **AI Sections** | `@` markers, parsed per tk2Text algorithm |
| **Attachments** | `Files/` directory + `Info.txt` metadata |
| **Encoding** | UTF-8, CRLF (tolerate LF) |
| **Escaping** | `\t`, `\r`, `\n`, `\\` for Content fields only |
| **Compatibility** | Read legacy formats, write current format |
