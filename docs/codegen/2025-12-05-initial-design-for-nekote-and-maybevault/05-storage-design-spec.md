# Storage Design Specification

## Overview

This document describes the file and directory structure for MaybeVault storage, designed to be:
- **Repo-friendly**: Two users can add files independently with minimal conflicts
- **Human-readable**: Files and metadata can be browsed directly
- **Efficient**: Quick lookups without loading everything into memory

---

## Directory Structure

```
{StoragePath}/                           # User-configurable root
├── files/                               # Actual archived files
│   ├── 2024-06/
│   │   ├── vacation-photo.jpg
│   │   └── receipt.pdf
│   ├── 2024-07/
│   │   ├── homework-math.docx
│   │   ├── image.jpg
│   │   └── image(2).jpg
│   └── 2024-08/
│       └── notes.txt
├── metadata/                            # File metadata by month
│   ├── 2024-06.json
│   ├── 2024-07.json
│   └── 2024-08.json
├── tags.json                            # User-defined tags
├── categories.json                      # User-defined categories
├── hashes.json                          # SHA256 hash index for duplicate detection
└── settings.json                        # Application settings
```

---

## File Storage

### Month Directories

Files are stored in directories named `YYYY-MM`:
- `2024-01` through `2024-12`
- Natural sorting works correctly
- Matches human memory patterns

### Filename Conflict Resolution

When storing `image.jpg` and a file with that name already exists:

```
Original: image.jpg exists
Attempt:  image.jpg      -> image(2).jpg
Attempt:  image(2).jpg   -> image(3).jpg
...and so on
```

**Rules:**
- No space before parenthesis: `image(2).jpg` not `image (2).jpg`
- Numbering starts at 2 (first file has no number)
- Check both stored names in the directory AND pending files in the same batch

### Original Name Preservation

The `originalName` is always stored in metadata, so users can see what the file was called before archiving.

---

## Metadata Storage

### Monthly Metadata Files

Each month has one JSON file containing metadata for all files archived that month.

**File:** `metadata/2024-07.json`

```json
{
  "month": "2024-07",
  "lastModifiedUtc": "2024-07-20T15:30:00Z",
  "files": [
    {
      "id": "01HYX7ABC123",
      "originalName": "Document1.docx",
      "storedName": "homework-math.docx",
      "month": "2024-07",
      "sha256": "a1b2c3d4e5f6789...",
      "sizeBytes": 15234,
      "mimeType": "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
      "timestamps": {
        "createdUtc": "2024-07-10T08:15:00Z",
        "modifiedUtc": "2024-07-15T14:30:00Z",
        "archivedUtc": "2024-07-16T09:00:00Z"
      },
      "categoryId": "homework",
      "tagIds": ["math", "daughter"],
      "notes": "Multiplication worksheet"
    },
    {
      "id": "01HYX8DEF456",
      "originalName": "image.jpg",
      "storedName": "image.jpg",
      "month": "2024-07",
      "sha256": "f1e2d3c4b5a6...",
      "sizeBytes": 245678,
      "mimeType": "image/jpeg",
      "timestamps": {
        "createdUtc": "2024-07-18T12:00:00Z",
        "modifiedUtc": "2024-07-18T12:00:00Z",
        "archivedUtc": "2024-07-18T14:00:00Z"
      },
      "categoryId": null,
      "tagIds": [],
      "notes": null
    }
  ]
}
```

### Why Monthly Partitions?

| Approach | Pros | Cons |
|----------|------|------|
| Single file | Simple | Merge conflicts, large file |
| Daily files | Minimal conflicts | Too many files |
| **Monthly files** | **Good balance** | **Occasional conflicts** |
| Database | Fast queries | Binary, merge impossible |

Monthly partitions are optimal because:
- Two users rarely archive on the same day
- When conflicts occur, they're small and merge-friendly
- Files stay reasonably sized

---

## Hash Index

### Purpose

The hash index enables O(1) duplicate detection without scanning all metadata files.

**File:** `hashes.json`

```json
{
  "lastModifiedUtc": "2024-07-20T15:30:00Z",
  "hashes": {
    "a1b2c3d4e5f6789...": {
      "fileId": "01HYX7ABC123",
      "month": "2024-07",
      "storedName": "homework-math.docx"
    },
    "f1e2d3c4b5a6...": {
      "fileId": "01HYX8DEF456",
      "month": "2024-07",
      "storedName": "image.jpg"
    }
  }
}
```

### Operations

**Adding a file:**
1. Compute SHA256 of source file
2. Check `hashes.json` for existing entry
3. If found → duplicate warning
4. If not found → add entry after successful archive

**Rebuilding the index:**
If `hashes.json` is corrupted or missing, rebuild by scanning all metadata files:
```csharp
foreach (var metadataFile in Directory.GetFiles(metadataDir, "*.json"))
{
    var partition = JsonSerializer.Deserialize<MonthPartition>(File.ReadAllText(metadataFile));
    foreach (var file in partition.Files)
    {
        hashIndex[file.Sha256] = new HashEntry(file.Id, file.Month, file.StoredName);
    }
}
```

---

## Tags Storage

**File:** `tags.json`

```json
{
  "lastModifiedUtc": "2024-07-15T10:00:00Z",
  "tags": [
    {
      "id": "homework",
      "name": "Homework",
      "color": "#3B82F6",
      "createdUtc": "2024-01-15T10:00:00Z"
    },
    {
      "id": "daughter",
      "name": "Daughter",
      "color": "#EC4899",
      "createdUtc": "2024-01-15T10:00:00Z"
    },
    {
      "id": "math",
      "name": "Math",
      "color": "#10B981",
      "createdUtc": "2024-02-20T14:30:00Z"
    }
  ]
}
```

---

## Categories Storage

**File:** `categories.json`

```json
{
  "lastModifiedUtc": "2024-07-15T10:00:00Z",
  "categories": [
    {
      "id": "homework",
      "name": "Homework",
      "icon": "📚",
      "parentId": null,
      "sortOrder": 0,
      "createdUtc": "2024-01-15T10:00:00Z"
    },
    {
      "id": "receipts",
      "name": "Receipts",
      "icon": "🧾",
      "parentId": null,
      "sortOrder": 1,
      "createdUtc": "2024-01-15T10:00:00Z"
    },
    {
      "id": "photos",
      "name": "Photos",
      "icon": "📷",
      "parentId": null,
      "sortOrder": 2,
      "createdUtc": "2024-03-10T09:00:00Z"
    }
  ]
}
```

---

## Settings Storage

**File:** `settings.json`

```json
{
  "storagePath": "C:\\Users\\User\\Documents\\MaybeVault",
  "recentFilesCount": 10,
  "warnOnDuplicates": true,
  "copyInsteadOfMove": true,
  "defaultCategoryId": null
}
```

**Note:** Settings may also be stored in the user's app data folder rather than the vault itself, depending on whether settings should sync with the vault or be per-machine.

---

## Repo-Friendly Design

### Git Workflow

```bash
# Before adding files
git pull

# Add files through MaybeVault app

# Commit and push
git add .
git commit -m "Add archived files"
git push
```

### Conflict Scenarios

| Scenario | Likelihood | Resolution |
|----------|------------|------------|
| Same month, different files | Common | JSON merge (array concat) |
| Same month, same filename | Rare | Manual resolution |
| Same file (duplicate) | Prevented | Hash check before add |
| Tags/categories edited | Rare | Manual merge |

### Minimizing Conflicts

1. **Pull before add**: Ensure local hash index is current
2. **Monthly partitions**: Limits conflict scope
3. **Append-only metadata**: New files = new array entries
4. **Unique IDs**: No ID collisions with ULIDs

### Recommended .gitattributes

```gitattributes
# Treat JSON as mergeable text
*.json text eol=lf merge=union

# Treat binary files properly
files/**/*.jpg binary
files/**/*.png binary
files/**/*.pdf binary
files/**/*.docx binary
files/**/*.xlsx binary
```

The `merge=union` strategy for JSON helps with array-based merges, though manual review is still recommended.

---

## Storage Service Interface

```csharp
namespace MaybeVault.Core.Storage;

public interface IVaultStorage
{
    /// <summary>
    /// Root storage path.
    /// </summary>
    string RootPath { get; }

    /// <summary>
    /// Ensures all required directories exist.
    /// </summary>
    Task InitializeAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// Stores a file in the vault.
    /// </summary>
    Task<string> StoreFileAsync(
        string sourcePath,
        string month,
        string desiredFileName,
        bool copy = true,
        CancellationToken cancellationToken = default);

    /// <summary>
    /// Gets the full path to a stored file.
    /// </summary>
    string GetFilePath(string month, string storedName);

    /// <summary>
    /// Checks if a filename exists in a month.
    /// </summary>
    Task<bool> FileExistsAsync(string month, string fileName);

    /// <summary>
    /// Lists all month directories.
    /// </summary>
    Task<string[]> GetMonthsAsync(CancellationToken cancellationToken = default);

    /// <summary>
    /// Deletes a stored file (use with caution).
    /// </summary>
    Task DeleteFileAsync(string month, string storedName, CancellationToken cancellationToken = default);
}
```

---

## Metadata Store Interface

```csharp
namespace MaybeVault.Core.Storage;

public interface IVaultMetadataStore
{
    /// <summary>
    /// Gets all files for a month.
    /// </summary>
    Task<VaultFile[]> GetFilesAsync(string month, CancellationToken cancellationToken = default);

    /// <summary>
    /// Adds a file to a month's metadata.
    /// </summary>
    Task AddFileAsync(VaultFile file, CancellationToken cancellationToken = default);

    /// <summary>
    /// Updates a file's metadata.
    /// </summary>
    Task UpdateFileAsync(VaultFile file, CancellationToken cancellationToken = default);

    /// <summary>
    /// Gets a file by ID (searches all months).
    /// </summary>
    Task<VaultFile?> GetFileByIdAsync(string id, CancellationToken cancellationToken = default);

    /// <summary>
    /// Queries files across all months.
    /// </summary>
    Task<VaultFile[]> QueryFilesAsync(
        Func<VaultFile, bool> predicate,
        CancellationToken cancellationToken = default);

    /// <summary>
    /// Gets recent files by archive date.
    /// </summary>
    Task<VaultFile[]> GetRecentFilesAsync(int count, CancellationToken cancellationToken = default);
}
```

---

## Hash Index Interface

```csharp
namespace MaybeVault.Core.Storage;

public interface IVaultHashIndex
{
    /// <summary>
    /// Checks if a hash exists in the index.
    /// </summary>
    Task<HashIndexEntry?> FindByHashAsync(string sha256, CancellationToken cancellationToken = default);

    /// <summary>
    /// Adds a hash to the index.
    /// </summary>
    Task AddHashAsync(string sha256, HashIndexEntry entry, CancellationToken cancellationToken = default);

    /// <summary>
    /// Removes a hash from the index.
    /// </summary>
    Task RemoveHashAsync(string sha256, CancellationToken cancellationToken = default);

    /// <summary>
    /// Rebuilds the entire index from metadata files.
    /// </summary>
    Task RebuildAsync(CancellationToken cancellationToken = default);
}

public class HashIndexEntry
{
    public required string FileId { get; init; }
    public required string Month { get; init; }
    public required string StoredName { get; init; }
}
```
