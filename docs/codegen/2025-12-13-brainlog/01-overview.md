# BrainLog - Documentation Overview

**Date**: 2025-12-13
**Purpose**: Complete specification for building a cross-platform Avalonia UI application for capturing, refining, and managing brainstorming drafts

## Problem Statement

When brainstorming on the go (e.g., voice-typing while driving), raw input contains errors, typos, and inaccuracies. These drafts need:

- A place to be temporarily saved without pressure
- AI-assisted correction to fix obvious errors
- A workflow to review, refine, and finalize
- State management to prevent unactionable chaos from piling up

BrainLog provides a focused workflow for capturing raw thoughts, refining them with AI assistance, and systematically handling them to completion.

## Key Differentiator

Unlike IdeaDump (which sends short ideas to AI for full analysis via email), BrainLog handles **full-context brainstorming sessions** where the user provides substantial input and focuses on **preserving intent while fixing errors**.

## Document Index

| Document | Description |
|----------|-------------|
| [01-overview.md](01-overview.md) | This file - project summary and navigation |
| [02-architecture-and-design.md](02-architecture-and-design.md) | MVVM architecture, state machine, DTO/Domain patterns |
| [03-data-models-and-configuration.md](03-data-models-and-configuration.md) | Entry model, JSON schemas, appsettings.json, usersettings.json |
| [04-ai-integration.md](04-ai-integration.md) | AI service abstraction, OpenAI/Gemini, prompts, title generation |
| [05-ui-specification.md](05-ui-specification.md) | Layout, controls, buttons, keyboard shortcuts, dialogs |
| [06-file-management.md](06-file-management.md) | File watching, conflict detection, auto-save, portable mode |
| [07-project-structure.md](07-project-structure.md) | Solution structure, namespaces, NuGet packages |

## Key Features

### Draft Capture
- Text fields for draft input (voice-typed, pasted, or manually typed)
- No in-app audio handling - uses OS/external tools for voice input
- Auto-save with configurable interval and retention

### AI-Assisted Refinement
- Configurable prompts for text correction
- AI-generated titles on first save
- Support for multiple AI providers (OpenAI, Gemini)
- JSON Schema responses for structured output

### Workflow Management
- Four states: Editing → Checked → Handled → Archived
- Sequential state transitions with meaningful actions
- 24-hour cooldown before archive button appears (configurable)
- Filter to hide archived entries by default

### File Management
- One JSON file per entry for OneDrive/Dropbox compatibility
- FileSystemWatcher for external change detection
- Conflict warning without blocking saves
- Portable mode support (data next to executable)

### Multi-Process Architecture
- Single document per process (simple, focused editing)
- Multiple processes can run simultaneously
- Conflict detection handles concurrent edits

## Scope

### In Scope
- Cross-platform Avalonia UI application (Windows, macOS)
- Entry CRUD with state management
- AI integration for text correction and title generation
- File watching and conflict detection
- Auto-save functionality
- Configurable prompts

### Out of Scope
- Audio input/voice recognition (uses OS features)
- Cloud sync (relies on OneDrive/Dropbox)
- Chat-style AI conversations
- Document collaboration
