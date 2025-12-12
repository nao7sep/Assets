# Nekote Library Specification

## Overview

Nekote is a reusable C# class library providing common utilities that cover "90% of cases" for typical application needs. It emphasizes standard behavior with extensibility points.

## Repository Structure

```
Nekote/
├── src/
│   └── Nekote/
│       ├── Nekote.csproj
│       ├── IO/
│       │   ├── FileNameResolver.cs
│       │   ├── FileCopier.cs
│       │   └── PathUtilities.cs
│       ├── Hashing/
│       │   ├── IFileHasher.cs
│       │   ├── Sha256FileHasher.cs
│       │   └── HashCache.cs
│       ├── Tagging/
│       │   ├── Tag.cs
│       │   ├── ITaggable.cs
│       │   ├── TagCollection.cs
│       │   └── ITagStore.cs
│       ├── Categorization/
│       │   ├── Category.cs
│       │   ├── ICategorizable.cs
│       │   └── ICategoryStore.cs
│       ├── Storage/
│       │   ├── IDocumentStore.cs
│       │   ├── JsonDocumentStore.cs
│       │   ├── IPartitionedStore.cs
│       │   └── MonthlyPartitionedStore.cs
│       └── Time/
│           ├── TimestampInfo.cs
│           └── UtcTimestamp.cs
├── tests/
│   ├── NekoteTests/
│   │   ├── NekoteTests.csproj
│   │   ├── IO/
│   │   ├── Hashing/
│   │   ├── Tagging/
│   │   ├── Categorization/
│   │   ├── Storage/
│   │   └── Time/
│   └── NekoteSandbox/
│       ├── NekoteSandbox.csproj
│       └── Program.cs
└── Nekote.sln
```

## Namespace Structure

```csharp
Nekote.IO           // File operations, naming
Nekote.Hashing      // Hash computation, caching
Nekote.Tagging      // Tag models and storage
Nekote.Categorization  // Category models and storage
Nekote.Storage      // JSON document storage patterns
Nekote.Time         // Timestamp utilities
```

## Module Specifications

### Nekote.IO

#### FileNameResolver

Resolves duplicate filename conflicts using a configurable pattern.

```csharp
namespace Nekote.IO;

public class FileNameResolver
{
    /// <summary>
    /// Creates a resolver with the specified format.
    /// Default: "{name}({number}){extension}" produces "file(2).jpg"
    /// Note: No space before parenthesis.
    /// </summary>
    public FileNameResolver(string format = "{name}({number}){extension}");

    /// <summary>
    /// Starting number for duplicates. Default is 2.
    /// The first file has no number (e.g., "file.jpg"),
    /// second is (2) (e.g., "file(2).jpg"), third is (3), etc.
    /// </summary>
    public int StartNumber { get; set; } = 2;

    /// <summary>
    /// Resolves a filename to avoid conflicts.
    /// </summary>
    /// <param name="fileName">The desired filename</param>
    /// <param name="exists">Function to check if a name is already taken</param>
    /// <returns>An available filename</returns>
    public string Resolve(string fileName, Func<string, bool> exists);

    /// <summary>
    /// Async version for I/O-bound existence checks.
    /// </summary>
    public Task<string> ResolveAsync(string fileName, Func<string, Task<bool>> exists);
}
```

**Usage Example:**
```csharp
var resolver = new FileNameResolver();

// Check against file system
string safeName = resolver.Resolve("image.jpg",
    name => File.Exists(Path.Combine(targetDir, name)));

// Check against a list
var existing = new HashSet<string> { "image.jpg", "image(2).jpg" };
string safeName = resolver.Resolve("image.jpg", existing.Contains);
// Returns: "image(3).jpg"
```

#### FileCopier

Copies files with progress reporting and cancellation support.

```csharp
namespace Nekote.IO;

public class FileCopier
{
    public event EventHandler<FileCopyProgressEventArgs>? Progress;

    public Task CopyAsync(
        string source,
        string destination,
        CancellationToken cancellationToken = default);

    public Task CopyAsync(
        string source,
        string destination,
        bool overwrite,
        CancellationToken cancellationToken = default);
}

public class FileCopyProgressEventArgs : EventArgs
{
    public long BytesCopied { get; }
    public long TotalBytes { get; }
    public double PercentComplete => (double)BytesCopied / TotalBytes * 100;
}
```

#### PathUtilities

Common path operations.

```csharp
namespace Nekote.IO;

public static class PathUtilities
{
    /// <summary>
    /// Generates a month-based directory name in YYYY-MM format: "2024-07"
    /// </summary>
    public static string GetMonthDirectory(DateTime date);
    public static string GetMonthDirectory(DateTimeOffset date);

    /// <summary>
    /// Parses a month directory name (YYYY-MM) back to a date range.
    /// Returns the first and last moments of that month in UTC.
    /// </summary>
    public static (DateTime Start, DateTime End) ParseMonthDirectory(string name);

    /// <summary>
    /// Ensures a path uses forward slashes (for cross-platform/repo storage).
    /// </summary>
    public static string NormalizeSlashes(string path);
}
```

---

### Nekote.Hashing

#### IFileHasher

```csharp
namespace Nekote.Hashing;

public interface IFileHasher
{
    /// <summary>
    /// Computes a hash from a file path.
    /// Returns lowercase hexadecimal string.
    /// </summary>
    Task<string> ComputeHashAsync(string filePath, CancellationToken cancellationToken = default);

    /// <summary>
    /// Computes a hash from a stream.
    /// Returns lowercase hexadecimal string.
    /// </summary>
    Task<string> ComputeHashAsync(Stream stream, CancellationToken cancellationToken = default);

    /// <summary>
    /// The algorithm name (e.g., "SHA256").
    /// </summary>
    string Algorithm { get; }
}
    string Algorithm { get; }
}
```

#### Sha256FileHasher

Default implementation using SHA256.

```csharp
namespace Nekote.Hashing;

public class Sha256FileHasher : IFileHasher
{
    public string Algorithm => "SHA256";

    public async Task<string> ComputeHashAsync(string filePath, CancellationToken cancellationToken = default)
    {
        await using var stream = new FileStream(filePath, FileMode.Open, FileAccess.Read, FileShare.Read,
            bufferSize: 81920, useAsync: true);
        return await ComputeHashAsync(stream, cancellationToken);
    }

    public async Task<string> ComputeHashAsync(Stream stream, CancellationToken cancellationToken = default)
    {
        var hash = await SHA256.HashDataAsync(stream, cancellationToken);
        return Convert.ToHexString(hash).ToLowerInvariant();
    }
}
```

#### HashCache

Caches computed hashes to avoid recomputation.

```csharp
namespace Nekote.Hashing;

public class HashCache
{
    public HashCache(IFileHasher hasher);

    /// <summary>
    /// Gets or computes the hash for a file.
    /// Uses file path + last write time as cache key.
    /// </summary>
    public Task<string> GetHashAsync(string filePath, CancellationToken cancellationToken = default);

    /// <summary>
    /// Clears the cache.
    /// </summary>
    public void Clear();

    /// <summary>
    /// Removes a specific file from cache.
    /// </summary>
    public void Invalidate(string filePath);
}
```

---

### Nekote.Tagging

#### Tag

```csharp
namespace Nekote.Tagging;

public class Tag : IEquatable<Tag>
{
    public required string Id { get; init; }          // Unique identifier (slug-like: "math-homework")
    public required string Name { get; set; }          // Display name ("Math Homework")
    public string? Color { get; set; }        // Optional hex color ("#3B82F6")
    public DateTime CreatedUtc { get; init; } = DateTime.UtcNow;

    // Equality based on Id
    public bool Equals(Tag? other) => other is not null && Id == other.Id;
    public override bool Equals(object? obj) => Equals(obj as Tag);
    public override int GetHashCode() => Id.GetHashCode();
}
```

#### ITaggable

```csharp
namespace Nekote.Tagging;

public interface ITaggable
{
    IReadOnlyCollection<string> TagIds { get; }

    void AddTag(string tagId);
    void RemoveTag(string tagId);
    bool HasTag(string tagId);
    void ClearTags();
}
```

#### TagCollection

A helper implementation for ITaggable.

```csharp
namespace Nekote.Tagging;

public class TagCollection : ITaggable
{
    private readonly HashSet<string> _tagIds = new();

    public IReadOnlyCollection<string> TagIds => _tagIds;

    public void AddTag(string tagId) => _tagIds.Add(tagId);
    public void RemoveTag(string tagId) => _tagIds.Remove(tagId);
    public bool HasTag(string tagId) => _tagIds.Contains(tagId);
    public void ClearTags() => _tagIds.Clear();

    // For serialization
    public IEnumerable<string> GetTagIds() => _tagIds;
    public void SetTagIds(IEnumerable<string> tagIds);
}
```

#### ITagStore

```csharp
namespace Nekote.Tagging;

public interface ITagStore
{
    Task<IReadOnlyList<Tag>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<Tag?> GetByIdAsync(string id, CancellationToken cancellationToken = default);
    Task AddAsync(Tag tag, CancellationToken cancellationToken = default);
    Task UpdateAsync(Tag tag, CancellationToken cancellationToken = default);
    Task DeleteAsync(string id, CancellationToken cancellationToken = default);
    Task<bool> ExistsAsync(string id, CancellationToken cancellationToken = default);
}
```

---

### Nekote.Categorization

#### Category

```csharp
namespace Nekote.Categorization;

public class Category : IEquatable<Category>
{
    public required string Id { get; init; }           // Unique identifier
    public required string Name { get; set; }          // Display name
    public string? Icon { get; set; }         // Optional icon (emoji or icon name)
    public string? ParentId { get; set; }     // Optional parent for hierarchy
    public int SortOrder { get; set; }        // Display order
    public DateTime CreatedUtc { get; init; } = DateTime.UtcNow;

    public bool Equals(Category? other) => other is not null && Id == other.Id;
    public override bool Equals(object? obj) => Equals(obj as Category);
    public override int GetHashCode() => Id.GetHashCode();
}
```

#### ICategorizable

```csharp
namespace Nekote.Categorization;

public interface ICategorizable
{
    string? CategoryId { get; set; }
}
```

#### ICategoryStore

```csharp
namespace Nekote.Categorization;

public interface ICategoryStore
{
    Task<IReadOnlyList<Category>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<IReadOnlyList<Category>> GetRootsAsync(CancellationToken cancellationToken = default);
    Task<IReadOnlyList<Category>> GetChildrenAsync(string parentId, CancellationToken cancellationToken = default);
    Task<Category?> GetByIdAsync(string id, CancellationToken cancellationToken = default);
    Task AddAsync(Category category, CancellationToken cancellationToken = default);
    Task UpdateAsync(Category category, CancellationToken cancellationToken = default);
    Task DeleteAsync(string id, CancellationToken cancellationToken = default);
}
```

---

### Nekote.Storage

#### IDocumentStore

Generic JSON document storage.

```csharp
namespace Nekote.Storage;

public interface IDocumentStore<T> where T : class
{
    Task<T?> LoadAsync(CancellationToken cancellationToken = default);
    Task SaveAsync(T document, CancellationToken cancellationToken = default);
    Task<bool> ExistsAsync(CancellationToken cancellationToken = default);
    Task DeleteAsync(CancellationToken cancellationToken = default);
}
```

#### JsonDocumentStore

```csharp
namespace Nekote.Storage;

public class JsonDocumentStore<T> : IDocumentStore<T> where T : class
{
    public JsonDocumentStore(string filePath);
    public JsonDocumentStore(string filePath, JsonSerializerOptions options);

    public string FilePath { get; }

    // Implementation uses System.Text.Json with:
    // - WriteIndented = true (human readable)
    // - PropertyNamingPolicy = JsonNamingPolicy.CamelCase
}
```

#### IPartitionedStore

For data partitioned by time periods.

```csharp
namespace Nekote.Storage;

public interface IPartitionedStore<TItem, TPartition>
{
    Task<TPartition?> GetPartitionAsync(string partitionKey, CancellationToken cancellationToken = default);
    Task SavePartitionAsync(string partitionKey, TPartition partition, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<string>> GetPartitionKeysAsync(CancellationToken cancellationToken = default);
    Task<IReadOnlyList<TItem>> QueryAsync(Func<TItem, bool> predicate, CancellationToken cancellationToken = default);
}
```

#### MonthlyPartitionedStore

```csharp
namespace Nekote.Storage;

public class MonthlyPartitionedStore<TItem> : IPartitionedStore<TItem, MonthPartition<TItem>>
{
    public MonthlyPartitionedStore(string directoryPath);

    // Partition key format: "YYYY-MM"
    // File naming: "{YYYY-MM}.json"
}

public class MonthPartition<TItem>
{
    public string Month { get; set; }         // "2024-07"
    public List<TItem> Items { get; set; }
    public DateTime LastModifiedUtc { get; set; }
}
```

---

### Nekote.Time

#### TimestampInfo

Captures all relevant timestamps for a file.

```csharp
namespace Nekote.Time;

public class TimestampInfo
{
    public DateTime CreatedUtc { get; set; }
    public DateTime ModifiedUtc { get; set; }
    public DateTime? AccessedUtc { get; set; }  // Optional, less reliable
    public DateTime ArchivedUtc { get; set; }   // When added to system

    /// <summary>
    /// Creates from file system info.
    /// </summary>
    public static TimestampInfo FromFile(string filePath);
    public static TimestampInfo FromFile(FileInfo fileInfo);

    /// <summary>
    /// Creates with current UTC time as archived time.
    /// </summary>
    public static TimestampInfo FromFileWithArchiveTime(string filePath);
    public static TimestampInfo FromFileWithArchiveTime(FileInfo fileInfo);
}
```

#### UtcTimestamp

Helper for UTC timestamp operations.

```csharp
namespace Nekote.Time;

public static class UtcTimestamp
{
    public static DateTime Now => DateTime.UtcNow;

    /// <summary>
    /// Formats for JSON-friendly ISO 8601.
    /// </summary>
    public static string Format(DateTime utc);

    /// <summary>
    /// Parses ISO 8601 string to UTC DateTime.
    /// </summary>
    public static DateTime Parse(string iso8601);
    public static DateTime? TryParse(string iso8601);
}
```

---

## Implementation Priority

Recommended order for implementation:

1. **Phase 1 - Foundation**
   - `Sha256FileHasher` (essential for duplicate detection)
   - `FileNameResolver` (needed for storing files)
   - `JsonDocumentStore<T>` (base storage pattern)
   - `TimestampInfo` (file metadata capture)

2. **Phase 2 - Tagging & Categories**
   - `Tag`, `ITaggable`, `TagCollection`
   - `Category`, `ICategorizable`
   - JSON-based store implementations

3. **Phase 3 - Advanced Storage**
   - `MonthlyPartitionedStore`
   - `HashCache`
   - `FileCopier` with progress

## Dependencies

- **Target Framework**: .NET 10.0 (LTS)
- **External Dependencies**: None (uses only BCL - Base Class Library)
- **Test Framework**: xUnit
- **Test Dependencies**:
  - xunit
  - xunit.runner.visualstudio
  - coverlet.collector
  - FluentAssertions (optional, for readable assertions)
