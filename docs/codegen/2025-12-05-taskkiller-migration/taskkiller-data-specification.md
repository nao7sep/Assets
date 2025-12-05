# taskKiller Data Specification

A complete specification for the file-based data storage system used by taskKiller. This document enables AI-assisted code generation for implementations that are fully compatible with existing taskKiller data.

---

## Table of Contents

1. [Overview](#overview)
2. [Conventions](#conventions)
3. [Task Data Model](#task-data-model)
4. [Note Data Model](#note-data-model)
5. [File Attachment Data Model](#file-attachment-data-model)
6. [Directory Structure](#directory-structure)
7. [Text Escaping](#text-escaping)
8. [Backward Compatibility](#backward-compatibility)
9. [CRUD Operations](#crud-operations)
10. [Parsing Algorithms](#parsing-algorithms)

---

## Overview

taskKiller uses a **file-based storage system** with plain text files. No database is used.

### Data Entities

| Entity | Description | Storage |
|--------|-------------|---------|
| **Task** | Primary work item with metadata | Individual `.txt` file per task |
| **Note** | Comment/detail attached to a task | Embedded in parent task's file |
| **File Attachment** | File associated with task, note, or task list | Copied to `Files/` directory |

### Design Principles

1. **GUIDs as identifiers** - All entities use GUIDs for unique identification
2. **File-per-task** - Each task is a separate file (filename = GUID)
3. **Separation of volatile data** - State and ordering stored separately from content (for Git stability)
4. **Append-only attachments** - Attachment metadata is append-only
5. **UTF-8 encoding** - All text files use UTF-8 encoding
6. **Backward compatible** - New implementations must read old formats

---

## Conventions

### File Encoding

- **Encoding**: UTF-8 (with or without BOM)
- **Line endings**: Write `\r\n` (CRLF), tolerate `\n` (LF) on read

### GUID Format

- **Format**: `D` format (lowercase with hyphens, no braces)
- **Example**: `a1b2c3d4-e5f6-7890-abcd-ef1234567890`
- **Length**: 36 characters

```csharp
guid.ToString("D", CultureInfo.InvariantCulture)
// or simply: guid.ToString() // defaults to "D"
```

### Timestamp Formats

| Context | Format | Example |
|---------|--------|---------|
| Task/Note timestamps | `long` (UTC ticks) | `638372841234567890` |
| Attachment timestamps | ISO 8601 `O` format | `2023-06-26T05:32:10.9033302Z` |

```csharp
// Task/Note: UTC ticks
DateTime.UtcNow.Ticks  // long
new DateTime(ticks, DateTimeKind.Utc)  // back to DateTime

// Attachments: ISO 8601
DateTime.UtcNow.ToString("O", CultureInfo.InvariantCulture)
DateTime.Parse(str, CultureInfo.InvariantCulture, DateTimeStyles.RoundtripKind)
```

### Key-Value File Format

Task and note files use a simple key-value format:
- One key-value pair per line
- Format: `Key:Value` (colon separator, no spaces around colon)
- Keys are case-sensitive
- Values can be empty (e.g., `RepeatedGuid:`)
- Blank lines separate paragraphs (task header vs. notes)

---

## Task Data Model

### DTO Definition

```csharp
public enum TaskState
{
    Later = 0,    // Default value
    Soon = 1,
    Now = 2,
    Done = 3,
    Cancelled = 4
}

public sealed record TaskDto
{
    public required Guid Guid { get; init; }
    public required DateTime CreationUtc { get; init; }
    public required string Content { get; init; }
    public required TaskState State { get; init; }
    public DateTime? HandlingUtc { get; init; }
    public Guid? RepeatedGuid { get; init; }
    public required long OrderingUtc { get; init; }
    public IReadOnlyList<NoteDto> Notes { get; init; } = [];

    // Runtime-only, not persisted:
    // public bool IsSpecial { get; set; } // true when OrderingUtc was negative at load time
}
```

### Task File: `Tasks/{GUID}.txt`

#### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `Format` | string | Must be `taskKiller1` (first line, validates file format) |
| `Guid` | Guid | Unique identifier, must match filename |
| `CreationUtc` | long | `DateTime.UtcNow.Ticks` at creation |
| `Content` | string | Task content ([escaped](#text-escaping)) |
| `State` | string | `Queued`, `Done`, or `Cancelled` (see [State Handling](#state-handling)) |

#### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `HandlingUtc` | long | Ticks when completed/cancelled (only for Done/Cancelled) |
| `RepeatedGuid` | Guid | Original task's GUID if this is a repeated task |
| `OrderingUtc` | long | Sort order (legacy field, see [Backward Compatibility](#backward-compatibility)) |

#### State Handling

The `State` field in the file does NOT directly represent the actual state for active tasks:

| File Value | Actual State | Notes |
|------------|--------------|-------|
| `Queued` | `Later`, `Soon`, or `Now` | Real state stored in `States/{GUID}.txt` |
| `Done` | `Done` | Final state, stored in file |
| `Cancelled` | `Cancelled` | Final state, stored in file |

#### Example Task File

```
Format:taskKiller1
Guid:a1b2c3d4-e5f6-7890-abcd-ef1234567890
CreationUtc:638372841234567890
Content:Buy groceries\r\nDon't forget milk
State:Queued
```

#### Example Completed Task File

```
Format:taskKiller1
Guid:a1b2c3d4-e5f6-7890-abcd-ef1234567890
CreationUtc:638372841234567890
Content:Buy groceries
State:Done
HandlingUtc:638372899999999999
```

### Auxiliary File: `States/{GUID}.txt`

Single line containing the active state: `Later`, `Soon`, or `Now`

- **Only exists** for active tasks (not Done/Cancelled)
- **Takes precedence** over `State` field in task file
- **Delete** when task becomes Done or Cancelled
- **Default**: If file doesn't exist and State is `Queued`, default to `Later`

### Auxiliary File: `Ordering/{GUID}.txt`

Single line containing the ordering value as `long` (ticks)

- **Takes precedence** over `OrderingUtc` field in task file
- **Lower values** appear first in the list
- **Special value `-1`**: Indicates imported task needing order reassignment

### Filename Validation

The task file's GUID must match its filename (case-insensitive comparison):

```csharp
bool IsValidFilename(string filename, Guid taskGuid) =>
    string.Equals(
        Path.GetFileNameWithoutExtension(filename),
        taskGuid.ToString("D"),
        StringComparison.OrdinalIgnoreCase);
```

Files that fail validation should be skipped during loading.

---

## Note Data Model

### DTO Definition

```csharp
public sealed record NoteDto
{
    public required Guid Guid { get; init; }
    public required DateTime CreationUtc { get; init; }
    public required string Content { get; init; }
}
```

### Storage: Embedded in Task File

Notes are stored as additional paragraphs in the parent task's file, separated by blank lines.

#### Fields

| Field | Type | Description |
|-------|------|-------------|
| `Guid` | Guid | Unique identifier for the note |
| `CreationUtc` | long | `DateTime.UtcNow.Ticks` at creation |
| `Content` | string | Note content ([escaped](#text-escaping), can be multi-line) |

#### Example Task with Notes

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
Content:Shopping list:\r\n- Milk\r\n- Eggs\r\n- Butter
```

### Note Display Order

Notes are sorted by `CreationUtc` ascending (oldest first).

---

## File Attachment Data Model

### DTO Definition

```csharp
public sealed record FileAttachmentDto
{
    public required Guid Guid { get; init; }
    public required string RelativePath { get; init; }  // e.g., "Files/doc.pdf" or "Files/1/doc.pdf"
    public Guid? ParentGuid { get; init; }              // Task GUID, Note GUID, or null
    public required DateTime AttachedAtUtc { get; init; }
    public required DateTime ModifiedAtUtc { get; init; }  // Original file's last modified time
}
```

### Attachment Targets

| Target | ParentGuid Value |
|--------|------------------|
| Task | Task's GUID |
| Note | Note's GUID |
| Task List (global) | `null` (empty in file) |

### Storage: `Files/` Directory

Files are **copied** (not moved) to `Files/` with original filename preserved.

#### Conflict Resolution

1. Try `Files/{filename}`
2. If exists (or filename is `Info.txt`), try `Files/1/{filename}`
3. If exists, try `Files/2/{filename}`, etc.

### Metadata: `Files/Info.txt`

INI-style format with sections. **Append-only** (new entries added at end).

#### Section Format

```
[{relative-path}]
Guid:{guid}
ParentGuid:{guid-or-empty}
AttachedAt:{ISO8601}
ModifiedAt:{ISO8601}
```

#### Fields

| Field | Type | Format |
|-------|------|--------|
| Section header | string | `[Files/path/to/file.ext]` |
| `Guid` | Guid | `D` format |
| `ParentGuid` | Guid? | `D` format or empty string |
| `AttachedAt` | DateTime | ISO 8601 `O` format (UTC) |
| `ModifiedAt` | DateTime | ISO 8601 `O` format (UTC) |

#### Example Info.txt

```
[Files/project-spec.pdf]
Guid:d4e5f6a7-b8c9-0123-def0-456789abcdef
ParentGuid:a1b2c3d4-e5f6-7890-abcd-ef1234567890
AttachedAt:2023-06-26T05:32:10.9033302Z
ModifiedAt:2023-06-25T10:15:30.1234567Z

[Files/meeting-notes.docx]
Guid:e5f6a7b8-c9d0-1234-ef01-56789abcdef0
ParentGuid:
AttachedAt:2023-06-27T08:00:00.0000000Z
ModifiedAt:2023-06-26T12:00:00.0000000Z

[Files/1/project-spec.pdf]
Guid:f6a7b8c9-d0e1-2345-f012-6789abcdef01
ParentGuid:b2c3d4e5-f6a7-8901-bcde-f23456789012
AttachedAt:2023-06-28T09:30:00.0000000Z
ModifiedAt:2023-06-27T14:45:00.0000000Z
```

### Path Separator in Info.txt

The relative path uses forward slash `/` as separator (cross-platform compatible). Both `/` and `\` should be handled on read.

---

## Directory Structure

### Single Task List

```
[TaskListRoot]/
├── Settings.txt              # Title and configuration (out of scope)
├── Tasks/
│   ├── {GUID-1}.txt          # Task with embedded notes
│   ├── {GUID-2}.txt
│   └── ...
├── States/
│   ├── {GUID-1}.txt          # Active state: Later|Soon|Now
│   └── ...                   # (only for active tasks)
├── Ordering/
│   ├── {GUID-1}.txt          # Sort order (long)
│   └── ...
└── Files/
    ├── Info.txt              # Attachment metadata
    ├── document.pdf          # Attached files
    ├── image.png
    ├── 1/                    # Conflict resolution subdirectory
    │   └── document.pdf
    └── 2/
        └── another.pdf
```

### Multiple Task Lists (Subtask Lists)

Task lists can exist as sibling directories under a common parent:

```
[ParentDirectory]/
├── MainProject/
│   ├── Settings.txt          # Contains "Title:Main Project"
│   ├── Tasks/
│   ├── States/
│   ├── Ordering/
│   └── Files/
├── SubProject-A/
│   ├── Settings.txt          # Contains "Title:Sub Project A"
│   └── ...
└── SubProject-B/
    ├── Settings.txt          # Contains "Title:Sub Project B"
    └── ...
```

A directory is recognized as a task list if it contains a `Settings.txt` file with a `Title:` key.

---

## Text Escaping

Task and note `Content` fields use C-style escaping for characters that would break the line-based format.

### Escape Rules

| Character | Escaped As | Description |
|-----------|------------|-------------|
| Tab (`\t`, 0x09) | `\t` | Horizontal tab |
| CR (`\r`, 0x0D) | `\r` | Carriage return |
| LF (`\n`, 0x0A) | `\n` | Line feed |
| Backslash (`\`) | `\\` | Literal backslash |

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
                char next = text[i + 1];
                char? unescaped = next switch
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
                throw new FormatException($"Invalid escape sequence: \\{next}");
            }
            builder.Append(text[i]);
        }
        return builder.ToString();
    }
}
```

### Usage

- **Escape** when writing `Content` values to files
- **Unescape** when reading `Content` values from files
- **Do NOT escape/unescape** other fields (Guid, timestamps, state, etc.)

---

## Backward Compatibility

### State Field Evolution

**Legacy format** (stored directly in task file):
```
State:Later
State:Soon
State:Now
```

**Current format** (active states stored separately):
```
State:Queued  # In task file
```
Real state in `States/{GUID}.txt`: `Later`, `Soon`, or `Now`

#### Reading Priority

1. Check `States/{GUID}.txt` - if exists, use its value
2. Else check `State` field in task file:
   - If not `Queued`, parse as `TaskState` enum
   - If `Queued` or missing, default to `Later`

#### Writing Rules

- **Active tasks** (`Later`/`Soon`/`Now`): Write `State:Queued` to task file, write actual state to `States/{GUID}.txt`
- **Completed tasks** (`Done`/`Cancelled`): Write actual state to task file, delete `States/{GUID}.txt` if exists

### OrderingUtc Field Evolution

**Legacy format** (in task file):
```
OrderingUtc:638372841234567890
```

**Current format** (stored separately):
`Ordering/{GUID}.txt` contains the value as a single line.

#### Reading Priority

1. Check `Ordering/{GUID}.txt` - if exists, use its value
2. Else check `OrderingUtc` field in task file - if exists, use its value
3. Else use `-1` (indicates needs reassignment)

#### Writing Rules

- Always write ordering to `Ordering/{GUID}.txt`
- Do NOT write `OrderingUtc` to task file

### Handling OrderingUtc = -1

When loading tasks with `OrderingUtc < 0`:
1. Collect them separately from normal tasks
2. Sort by their negative values (preserves relative import order)
3. Assign new values: `MinOrderingUtc - 1`, `MinOrderingUtc - 2`, etc.
4. Mark as "special" for visual highlighting (runtime flag, not persisted)
5. Do NOT save the new ordering immediately (preserves highlight until user action)

### Empty/Missing Optional Fields

Optional fields may be:
- **Missing entirely** (key not present)
- **Present with empty value** (e.g., `RepeatedGuid:`)

Both cases should be treated as "not set" / `null`.

---

## CRUD Operations

### Task Operations

#### Create Task

```csharp
1. Generate new GUID
2. Set CreationUtc = DateTime.UtcNow.Ticks
3. Set OrderingUtc = GetMinOrderingUtc() - 1  // Places at top
4. Create Tasks/{GUID}.txt with:
   - Format:taskKiller1
   - Guid:{guid}
   - CreationUtc:{ticks}
   - Content:{escaped-content}
   - State:Queued (or Done/Cancelled if final)
   - HandlingUtc:{ticks} (if Done/Cancelled)
   - RepeatedGuid:{guid} (if applicable)
5. Create States/{GUID}.txt with actual state (if active)
6. Create Ordering/{GUID}.txt with ordering value
```

#### Read Task

```csharp
1. Read Tasks/{GUID}.txt
2. Validate Format == "taskKiller1"
3. Validate filename matches Guid field
4. Parse required fields
5. Parse optional fields (handle missing/empty)
6. Override State from States/{GUID}.txt if exists
7. Override OrderingUtc from Ordering/{GUID}.txt if exists
8. Parse note paragraphs (paragraphs after first)
9. Sort notes by CreationUtc ascending
```

#### Read All Active Tasks

```csharp
1. Enumerate Tasks/*.txt files
2. For each file:
   a. Load task
   b. Skip if State is Done or Cancelled
   c. Skip if filename doesn't match GUID
   d. If OrderingUtc < 0, add to pending list
   e. Else add to main list
3. Sort pending list by OrderingUtc ascending
4. For each pending task:
   a. Set OrderingUtc = GetMinOrderingUtc() - 1
   b. Set IsSpecial = true (runtime flag)
   c. Add to main list
5. Sort main list by OrderingUtc ascending
```

#### Update Task

```csharp
1. Modify task properties as needed
2. Write task file (same format as Create)
3. Update States/{GUID}.txt if state is active, delete if final
4. Update Ordering/{GUID}.txt if ordering changed
```

#### Delete Task

```csharp
1. Delete Tasks/{GUID}.txt
2. Delete States/{GUID}.txt if exists
3. Delete Ordering/{GUID}.txt if exists
// Note: Orphaned attachments are not automatically deleted
```

### Note Operations

#### Create Note

```csharp
1. Generate new GUID for note
2. Set CreationUtc = DateTime.UtcNow.Ticks
3. Add note paragraph to parent task file
4. Save parent task file
```

#### Update Note

```csharp
1. Modify note content
2. Save parent task file (rewrites entire file)
```

#### Delete Note

```csharp
1. Remove note from parent task's note collection
2. Save parent task file (rewrites without that note)
```

### Attachment Operations

#### Create Attachment

```csharp
1. Determine target path:
   a. Try Files/{filename}
   b. If exists or filename is "Info.txt", try Files/1/{filename}
   c. Continue incrementing until available path found
2. Copy source file to target path (do not overwrite)
3. Generate new GUID
4. Append entry to Files/Info.txt:
   [{relative-path}]
   Guid:{guid}
   ParentGuid:{parent-guid-or-empty}
   AttachedAt:{ISO8601-now}
   ModifiedAt:{ISO8601-source-file-modified}
```

#### Read Attachments

```csharp
1. Parse Files/Info.txt section by section
2. For each section:
   a. Extract relative path from header [...]
   b. Parse key-value pairs
   c. Optionally verify file exists at path
3. Return list of FileAttachmentDto
```

#### Update/Delete Attachment (Not in Original)

Currently not implemented. For new implementations:

**Option A - Soft Delete**: Add `Deleted:true` field to section
**Option B - Rewrite**: Parse all sections, modify/remove target, rewrite file

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
                current.Append('\n');  // Preserve internal line breaks
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
        if (colonIndex > 0)  // Key must have at least 1 character
        {
            var key = line[..colonIndex];
            var value = line[(colonIndex + 1)..];
            dict[key] = value;  // Later keys overwrite earlier ones
        }
    }

    return dict;
}
```

### Parse Info.txt (INI Sections)

```csharp
public static List<(string Path, Dictionary<string, string> Props)> ParseInfoFile(string content)
{
    var entries = new List<(string, Dictionary<string, string>)>();
    string? currentSection = null;
    Dictionary<string, string>? currentProps = null;

    foreach (var line in content.Split('\n'))
    {
        var trimmed = line.TrimEnd('\r');

        if (string.IsNullOrWhiteSpace(trimmed))
            continue;

        if (trimmed.StartsWith('[') && trimmed.EndsWith(']'))
        {
            // Save previous section
            if (currentSection != null && currentProps != null)
                entries.Add((currentSection, currentProps));

            currentSection = trimmed[1..^1];  // Remove [ and ]
            currentProps = new Dictionary<string, string>();
        }
        else if (currentProps != null)
        {
            var colonIndex = trimmed.IndexOf(':');
            if (colonIndex > 0)
            {
                var key = trimmed[..colonIndex];
                var value = trimmed[(colonIndex + 1)..];
                currentProps[key] = value;
            }
        }
    }

    // Save last section
    if (currentSection != null && currentProps != null)
        entries.Add((currentSection, currentProps));

    return entries;
}
```

### Get Minimum Ordering UTC

```csharp
public static long GetMinOrderingUtc(IEnumerable<TaskDto> tasks)
{
    var validTasks = tasks.Where(t => t.OrderingUtc >= 0);

    if (validTasks.Any())
        return validTasks.Min(t => t.OrderingUtc);

    return DateTime.UtcNow.Ticks;  // No tasks, use current time
}
```

---

## Summary

This specification covers all aspects needed to implement a taskKiller-compatible data layer:

| Aspect | Key Points |
|--------|------------|
| **Tasks** | GUID-named files, `taskKiller1` format, state/ordering in separate files |
| **Notes** | Embedded in task files as additional paragraphs |
| **Attachments** | Copied to `Files/`, metadata in `Info.txt` |
| **Encoding** | UTF-8, CRLF line endings (tolerate LF) |
| **Escaping** | C-style for Content fields only (`\t`, `\r`, `\n`, `\\`) |
| **Compatibility** | Read old State/OrderingUtc from task files, write to separate files |

