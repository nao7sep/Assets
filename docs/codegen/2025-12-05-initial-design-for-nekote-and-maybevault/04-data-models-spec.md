# Data Models Specification

## Overview

This document defines all data models used in MaybeVault and Nekote, including their serialization format for JSON storage.

---

## Nekote Models

### Tag

Represents a user-defined tag for organizing items.

```csharp
namespace Nekote.Tagging;

public class Tag : IEquatable<Tag>
{
    /// <summary>
    /// Unique identifier, slug-like format (e.g., "math-homework").
    /// </summary>
    public required string Id { get; init; }

    /// <summary>
    /// Human-readable display name (e.g., "Math Homework").
    /// </summary>
    public required string Name { get; set; }

    /// <summary>
    /// Optional hex color for UI display (e.g., "#3B82F6").
    /// </summary>
    public string? Color { get; set; }

    /// <summary>
    /// When this tag was created.
    /// </summary>
    public DateTime CreatedUtc { get; init; } = DateTime.UtcNow;

    public bool Equals(Tag? other) => other is not null && Id == other.Id;
    public override bool Equals(object? obj) => Equals(obj as Tag);
    public override int GetHashCode() => Id.GetHashCode();
}
```

**JSON Format:**
```json
{
  "id": "math-homework",
  "name": "Math Homework",
  "color": "#3B82F6",
  "createdUtc": "2024-07-15T10:30:00Z"
}
```

### Category

Represents a user-defined category.

```csharp
namespace Nekote.Categorization;

public class Category : IEquatable<Category>
{
    /// <summary>
    /// Unique identifier.
    /// </summary>
    public required string Id { get; init; }

    /// <summary>
    /// Human-readable display name.
    /// </summary>
    public required string Name { get; set; }

    /// <summary>
    /// Optional icon (emoji or icon identifier).
    /// </summary>
    public string? Icon { get; set; }

    /// <summary>
    /// Optional parent category ID for hierarchy.
    /// </summary>
    public string? ParentId { get; set; }

    /// <summary>
    /// Display order within the same parent.
    /// </summary>
    public int SortOrder { get; set; }

    /// <summary>
    /// When this category was created.
    /// </summary>
    public DateTime CreatedUtc { get; init; } = DateTime.UtcNow;

    public bool Equals(Category? other) => other is not null && Id == other.Id;
    public override bool Equals(object? obj) => Equals(obj as Category);
    public override int GetHashCode() => Id.GetHashCode();
}
```

**JSON Format:**
```json
{
  "id": "homework",
  "name": "Homework",
  "icon": "📚",
  "parentId": null,
  "sortOrder": 0,
  "createdUtc": "2024-07-15T10:30:00Z"
}
```

### TimestampInfo

Captures file timestamps.

```csharp
namespace Nekote.Time;

public class TimestampInfo
{
    /// <summary>
    /// File creation time in UTC.
    /// </summary>
    public DateTime CreatedUtc { get; set; }

    /// <summary>
    /// File last modification time in UTC.
    /// </summary>
    public DateTime ModifiedUtc { get; set; }

    /// <summary>
    /// File last access time in UTC (optional, less reliable).
    /// </summary>
    public DateTime? AccessedUtc { get; set; }

    /// <summary>
    /// When the file was archived/added to the system.
    /// </summary>
    public DateTime ArchivedUtc { get; set; }

    public static TimestampInfo FromFile(string filePath)
    {
        var info = new FileInfo(filePath);
        return new TimestampInfo
        {
            CreatedUtc = info.CreationTimeUtc,
            ModifiedUtc = info.LastWriteTimeUtc,
            AccessedUtc = info.LastAccessTimeUtc,
            ArchivedUtc = DateTime.UtcNow
        };
    }

    public static TimestampInfo FromFile(FileInfo fileInfo)
    {
        return new TimestampInfo
        {
            CreatedUtc = fileInfo.CreationTimeUtc,
            ModifiedUtc = fileInfo.LastWriteTimeUtc,
            AccessedUtc = fileInfo.LastAccessTimeUtc,
            ArchivedUtc = DateTime.UtcNow
        };
    }
}
```

**JSON Format:**
```json
{
  "createdUtc": "2024-07-10T08:15:00Z",
  "modifiedUtc": "2024-07-15T14:30:00Z",
  "accessedUtc": "2024-07-15T14:30:00Z",
  "archivedUtc": "2024-07-16T09:00:00Z"
}
```

---

## MaybeVault Models

### VaultFile

The main entity representing an archived file.

```csharp
namespace MaybeVault.Core.Models;

public class VaultFile : ITaggable, ICategorizable
{
    /// <summary>
    /// Unique identifier (GUID or ULID).
    /// </summary>
    public required string Id { get; init; }

    /// <summary>
    /// Original filename before archiving.
    /// </summary>
    public required string OriginalName { get; init; }

    /// <summary>
    /// Filename as stored in vault (may differ due to conflict resolution).
    /// </summary>
    public required string StoredName { get; init; }

    /// <summary>
    /// Month partition where file is stored (e.g., "2024-07").
    /// </summary>
    public required string Month { get; init; }

    /// <summary>
    /// Relative path within vault storage.
    /// Format: "{month}/{storedName}" (e.g., "2024-07/homework.docx")
    /// </summary>
    public string RelativePath => $"{Month}/{StoredName}";

    /// <summary>
    /// SHA256 hash of file contents (lowercase hex).
    /// </summary>
    public required string Sha256 { get; init; }

    /// <summary>
    /// File size in bytes.
    /// </summary>
    public required long SizeBytes { get; init; }

    /// <summary>
    /// MIME type if detected (e.g., "image/jpeg").
    /// </summary>
    public string? MimeType { get; set; }

    /// <summary>
    /// File timestamps.
    /// </summary>
    public required TimestampInfo Timestamps { get; init; }

    /// <summary>
    /// Category ID (null if uncategorized).
    /// </summary>
    public string? CategoryId { get; set; }

    /// <summary>
    /// Tag IDs.
    /// </summary>
    public List<string> TagIds { get; set; } = new();

    /// <summary>
    /// User notes/description.
    /// </summary>
    public string? Notes { get; set; }

    // ITaggable implementation
    IReadOnlyCollection<string> ITaggable.TagIds => TagIds;
    public void AddTag(string tagId) => TagIds.Add(tagId);
    public void RemoveTag(string tagId) => TagIds.Remove(tagId);
    public bool HasTag(string tagId) => TagIds.Contains(tagId);
    public void ClearTags() => TagIds.Clear();
}
```

**JSON Format:**
```json
{
  "id": "01HYX7ABC123",
  "originalName": "Document1.docx",
  "storedName": "homework-math.docx",
  "month": "2024-07",
  "sha256": "a1b2c3d4e5f6...",
  "sizeBytes": 15234,
  "mimeType": "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
  "timestamps": {
    "createdUtc": "2024-07-10T08:15:00Z",
    "modifiedUtc": "2024-07-15T14:30:00Z",
    "accessedUtc": "2024-07-15T14:30:00Z",
    "archivedUtc": "2024-07-16T09:00:00Z"
  },
  "categoryId": "homework",
  "tagIds": ["math", "daughter", "grade-5"],
  "notes": "Multiplication practice worksheet"
}
```

### VaultSettings

Application configuration.

```csharp
namespace MaybeVault.Core.Models;

public class VaultSettings
{
    /// <summary>
    /// Root directory for vault storage.
    /// </summary>
    public string StoragePath { get; set; } = GetDefaultStoragePath();

    /// <summary>
    /// Number of recent files to show on add screen.
    /// </summary>
    public int RecentFilesCount { get; set; } = 10;

    /// <summary>
    /// Whether to show duplicate warnings.
    /// </summary>
    public bool WarnOnDuplicates { get; set; } = true;

    /// <summary>
    /// Whether to copy or move files when archiving.
    /// </summary>
    public bool CopyInsteadOfMove { get; set; } = true;

    /// <summary>
    /// Default category ID for new files (null = none).
    /// </summary>
    public string? DefaultCategoryId { get; set; }

    private static string GetDefaultStoragePath()
    {
        var documents = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments);
        return Path.Combine(documents, "MaybeVault");
    }
}
```

**JSON Format:**
```json
{
  "storagePath": "C:\\Users\\User\\Documents\\MaybeVault",
  "recentFilesCount": 10,
  "warnOnDuplicates": true,
  "copyInsteadOfMove": true,
  "defaultCategoryId": null
}
```

### AddFileOptions

Options when adding a file to the vault.

```csharp
namespace MaybeVault.Core.Models;

public class AddFileOptions
{
    /// <summary>
    /// Category to assign.
    /// </summary>
    public string? CategoryId { get; set; }

    /// <summary>
    /// Tags to assign.
    /// </summary>
    public List<string> TagIds { get; set; } = new();

    /// <summary>
    /// Notes/description.
    /// </summary>
    public string? Notes { get; set; }

    /// <summary>
    /// Custom filename (if null, uses original).
    /// </summary>
    public string? CustomFileName { get; set; }

    /// <summary>
    /// Override archive date (if null, uses current UTC).
    /// </summary>
    public DateTime? ArchiveDateUtc { get; set; }

    /// <summary>
    /// Skip duplicate check.
    /// </summary>
    public bool SkipDuplicateCheck { get; set; }

    /// <summary>
    /// Force add even if duplicate exists.
    /// </summary>
    public bool ForceAddDuplicate { get; set; }
}
```

### AddFileResult

Result of adding a file.

```csharp
namespace MaybeVault.Core.Models;

public class AddFileResult
{
    public required string SourcePath { get; init; }
    public bool Success { get; init; }
    public VaultFile? File { get; init; }
    public AddFileError? Error { get; init; }
    public DuplicateCheckResult? DuplicateInfo { get; init; }
}

public enum AddFileError
{
    None,
    FileNotFound,
    AccessDenied,
    DuplicateDeclined,
    StorageError,
    Unknown
}
```

### DuplicateCheckResult

Result of checking for duplicates.

```csharp
namespace MaybeVault.Core.Models;

public class DuplicateCheckResult
{
    public required string Sha256 { get; init; }
    public bool IsDuplicate { get; init; }
    public VaultFile? ExistingFile { get; init; }

    public static DuplicateCheckResult NotDuplicate(string sha256) => new()
    {
        Sha256 = sha256,
        IsDuplicate = false,
        ExistingFile = null
    };

    public static DuplicateCheckResult Duplicate(string sha256, VaultFile existing) => new()
    {
        Sha256 = sha256,
        IsDuplicate = true,
        ExistingFile = existing
    };
}
```

### VaultStatistics

Vault usage statistics.

```csharp
namespace MaybeVault.Core.Models;

public class VaultStatistics
{
    public int TotalFiles { get; init; }
    public long TotalSizeBytes { get; init; }
    public int MonthCount { get; init; }
    public int TagCount { get; init; }
    public int CategoryCount { get; init; }
    public string? OldestMonth { get; init; }
    public string? NewestMonth { get; init; }
    public DateTime CalculatedAtUtc { get; init; }

    public string TotalSizeFormatted => FormatBytes(TotalSizeBytes);

    private static string FormatBytes(long bytes)
    {
        string[] suffixes = { "B", "KB", "MB", "GB", "TB" };
        int i = 0;
        double size = bytes;
        while (size >= 1024 && i < suffixes.Length - 1)
        {
            size /= 1024;
            i++;
        }
        return $"{size:0.##} {suffixes[i]}";
    }
}
```

---

## ID Generation

### Recommended: ULID

For file IDs, use ULID (Universally Unique Lexicographically Sortable Identifier):
- Sortable by creation time
- URL-safe (no special characters)
- 26 characters (shorter than GUID)
- Example: `01HYX7ABC123DEF456GHI789JK`

```csharp
// Using Ulid package: https://github.com/Cysharp/Ulid
var id = Ulid.NewUlid().ToString();
```

### Alternative: GUID

If ULID is too complex, use GUID without dashes:
```csharp
var id = Guid.NewGuid().ToString("N"); // "a1b2c3d4e5f6..."
```

### Tag/Category IDs

For user-created items, generate slug-like IDs from names:
```csharp
public static string GenerateSlug(string name)
{
    return name
        .ToLowerInvariant()
        .Replace(' ', '-')
        .Where(c => char.IsLetterOrDigit(c) || c == '-')
        .Aggregate("", (s, c) => s + c)
        .Trim('-');
}

// "Math Homework" -> "math-homework"
```

---

## Serialization Settings

All JSON files use these settings:

```csharp
public static class JsonDefaults
{
    public static JsonSerializerOptions Options { get; } = new()
    {
        WriteIndented = true,
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
        Converters =
        {
            new JsonStringEnumConverter(JsonNamingPolicy.CamelCase)
        }
    };
}
```

This produces:
- Human-readable, indented JSON
- camelCase property names
- Null values omitted
- Enums as strings
