# Project Overview

## Summary

This document describes two related projects:

1. **Nekote** - A reusable C# class library providing common utilities for file management, tagging, categorization, hashing, and storage patterns.
2. **MaybeVault** - An Avalonia UI MVVM application for archiving files that users "probably won't need again" but want to keep just in case.

## Problem Statement

When working on temporary tasks (homework help, quick memos, photos to print), users create files that:
- Are often poorly named or organized
- Feel wasteful to delete permanently
- Occasionally turn out to be needed later
- Cause mild anxiety when deleted

MaybeVault provides a low-friction way to archive these files with minimal metadata, making them findable if ever needed again.

## Naming Decisions

### Project Names (Dot-Free)

To avoid macOS bundle detection issues (where `.app` is a special extension), all project and folder names avoid dots:

| Purpose | Name |
|---------|------|
| Reusable library | `Nekote` |
| Library tests | `NekoteTests` |
| Library sandbox | `NekoteSandbox` |
| Archive app | `MaybeVault` |
| App business logic | `MaybeVaultCore` |
| App tests | `MaybeVaultTests` |

### C# Namespaces (Dotted)

Namespaces can still use dots for logical organization:
- `Nekote.IO`
- `Nekote.Hashing`
- `MaybeVault.Core`
- `MaybeVault.ViewModels`

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      MaybeVault                             │
│                   (Avalonia UI App)                         │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     MaybeVaultCore                          │
│              (App-specific business logic)                  │
│  - VaultFile, VaultService                                  │
│  - Vault-specific workflows                                 │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                        Nekote                               │
│              (Reusable primitives library)                  │
│  - Tagging, Categorization                                  │
│  - File hashing, Duplicate detection                        │
│  - JSON storage, Name conflict resolution                   │
└─────────────────────────────────────────────────────────────┘
```

## Key Design Principles

### 1. Repo-Friendly Storage

Files and metadata are stored in a way that supports version control:
- Monthly partitioned JSON files (not one giant database)
- Two users can add files independently with minimal conflicts
- Human-readable metadata files

### 2. Duplicate Detection

Identical files (by SHA256 hash) are detected regardless of filename.
Users are warned but can choose to add anyway.

### 3. Month-Based Organization

Files are stored in `YYYY-MM` directories:
- Matches human memory ("that was last summer")
- Not too granular (day) or too coarse (year)
- ~12 folders per year is manageable

### 4. Duplicate Filename Handling

When `image.jpg` already exists:
- Second file becomes `image(2).jpg`
- No space before parenthesis
- Starts at (2) because the first is implicitly "the original"

### 5. Fire-and-Forget Primary, Search Secondary

- Main UI is for adding files quickly
- Search/browse is available but not the primary focus
- Users shouldn't be browsing often (that defeats the "vault" concept)

## Target Platforms

- **Nekote**: .NET 8+ class library (cross-platform)
- **MaybeVault**: Avalonia UI (Windows, macOS, Linux)

## Repository Structure

### Nekote Repository

```
Nekote/
├── src/
│   └── Nekote/
├── tests/
│   ├── NekoteTests/
│   └── NekoteSandbox/
└── Nekote.sln
```

### MaybeVault Repository

```
MaybeVault/
├── src/
│   ├── MaybeVault/
│   └── MaybeVaultCore/
├── tests/
│   └── MaybeVaultTests/
└── MaybeVault.sln
```
