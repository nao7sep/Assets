# taskKiller Modern Rewrite - Documentation Overview

This documentation set provides complete specifications for building a modern, cross-platform task management application that is fully compatible with taskKiller's existing data files.

## Document Index

| Document | Description |
|----------|-------------|
| [02-data-specification.md](02-data-specification.md) | Data models, file formats, text escaping, backward compatibility, AI section parsing |
| [03-avalonia-app-design.md](03-avalonia-app-design.md) | Avalonia UI architecture, MVVM patterns, project structure |

## Scope

### In Scope

- **TaskKillerLib**: A .NET class library for reading/writing taskKiller data files
- **TaskKillerApp**: A simple Avalonia UI application for basic CRUD operations
- Full backward compatibility with existing taskKiller data
- Cross-platform support (Windows, macOS, Linux)

### Out of Scope

- Settings management (Settings.txt parsing)
- Email functionality
- HTML report generation (covered by tk2Text)
- Subtask list synchronization features

## Source Code References

These specifications are derived from analysis of:

| Repository | Description |
|------------|-------------|
| `taskKiller` | Original WPF task management application |
| `Nekote2018` | Support library (text escaping, file utilities) |
| `tk2Text` | HTML export tool (AI section parsing reference) |

## Key Design Decisions

1. **File Format Preservation**: Write files in the exact format taskKiller expects
2. **Backward Compatibility**: Read both old and new state/ordering storage formats
3. **Modern C#**: Use records, nullable reference types, async/await
4. **Separation of Concerns**: Data layer (library) separate from UI (app)
5. **AI Section Support**: Parse `@` markers in notes (semantic convention from tk2Text)

## Quick Reference

### Task States

```csharp
enum TaskState { Later = 0, Soon, Now, Done, Cancelled }
```

### File Locations

```
[TaskListDirectory]/
├── Settings.txt              # Title and configuration
├── Tasks/{GUID}.txt          # Task files
├── States/{GUID}.txt         # Active state (Later|Soon|Now)
├── Ordering/{GUID}.txt       # Sort order (long ticks)
└── Files/
    ├── Info.txt              # Attachment metadata
    └── ...                   # Attached files
```

### Encoding

- UTF-8, CRLF line endings (tolerate LF on read)
- GUID format: "D" (hyphenated)
- Timestamps: `long` ticks for tasks/notes, ISO8601 "O" for attachments
