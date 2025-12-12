# Mazooka - Documentation Overview

**Date**: 2025-12-10
**Purpose**: Complete specification for building a .NET console application that monitors multiple IMAP accounts and applies Thunderbird-style message rules## Problem Statement

Mozilla Sync for Thunderbird message filters may be unreliable. This project provides a standalone .NET console application that:

- Monitors multiple IMAP email accounts
- Applies configurable message rules (similar to Thunderbird's filters)
- Uses a simple, version-controlled JSON configuration file
- Supports Gmail OAuth2 authentication
- Tracks processed messages to avoid reprocessing
- Provides comprehensive audit logging

## Document Index

| Document | Description |
|----------|-------------|
| [01-overview.md](01-overview.md) | This file - project summary and navigation |
| [02-architecture-and-design.md](02-architecture-and-design.md) | High-level architecture, design principles, execution modes |
| [03-data-models-and-configuration.md](03-data-models-and-configuration.md) | JSON configuration schemas, data models, security |
| [04-authentication-oauth2.md](04-authentication-oauth2.md) | OAuth2 implementation for Gmail, password auth for others |
| [05-rule-engine-specification.md](05-rule-engine-specification.md) | Rule conditions, text matching, action execution |
| [06-imap-operations.md](06-imap-operations.md) | MailKit integration, UID tracking, cross-account moves |
| [07-logging-and-auditing.md](07-logging-and-auditing.md) | Serilog configuration, SQLite audit database |
| [08-project-structure.md](08-project-structure.md) | Solution structure, NuGet packages, namespaces |

## Key Features

### Configuration Management
- Simple JSON configuration file (committable to git)
- Account credentials (password or OAuth2)
- Rule definitions with conditions and actions
- Rate limiting and backup policies

### Authentication Support
- **Password authentication** for standard IMAP servers
- **OAuth2 authentication** for Gmail (and other OAuth providers)
- Automatic token refresh
- Secure credential storage

### Rule Engine
- Flexible condition matching (from, to, subject, body, etc.)
- Text conditions: contains, startsWith, endsWith, equals, regex
- Multiple action types: move, delete, mark read/unread, flag
- Cross-account message moving
- Rule-to-account mapping

### Message Processing
- UID-based tracking to prevent reprocessing
- Configurable rate limits per account
- Backup strategies (all, moved/deleted only, none)
- Dry-run mode for testing rules

### Audit & Logging
- Dual logging: file-based + SQLite database
- Comprehensive action tracking
- Easy querying and reporting

## Scope

### In Scope
- **Mazooka.Core**: Core library with rule engine, IMAP operations, authentication
- **Mazooka**: Console application with CLI commands
- **Mazooka.Tests**: Unit tests
- Gmail OAuth2 support
- Password-based IMAP support
- UID tracking with SQLite
- Serilog logging
- Cross-account message moves
- Configurable rate limiting and backups

### Out of Scope
- Web UI or desktop GUI
- SMTP sending (only IMAP reading/management)
- POP3 protocol support
- Exchange/ActiveSync protocols
- Email composition or sending
- Calendar/contacts management

## Quick Reference

### Execution Modes

```bash
# One-time check of all accounts
dotnet run

# Setup Gmail OAuth2
dotnet run -- setup-account gmail work

# Test authentication
dotnet run -- test-auth work

# Dry run (preview without changes)
dotnet run -- --dry-run

# Daemon mode (continuous monitoring)
dotnet run -- --daemon
```

### Example Rule Configuration

```json
{
  "name": "Newsletter Organization",
  "applyToAccounts": ["work", "personal"],
  "conditions": {
    "matchType": "all",
    "from": { "contains": "newsletter" },
    "subject": { "contains": "weekly" }
  },
  "actions": [
    { "type": "moveToFolder", "folder": "Newsletters" },
    { "type": "markAsRead" }
  ]
}
```

## Design Philosophy

1. **Simplicity**: JSON configuration that's easy to read, edit, and version control
2. **Reliability**: UID tracking prevents duplicate processing
3. **Transparency**: Comprehensive logging of all actions
4. **Safety**: Dry-run mode and configurable backups
5. **Flexibility**: Support for various authentication methods and rule conditions
6. **Maintainability**: Clean architecture with separated concerns

## Technology Stack

- **.NET 8.0+**: Modern C# with nullable reference types
- **MailKit**: Mature IMAP library with OAuth2 support
- **Serilog**: Structured logging with multiple sinks
- **System.Text.Json**: Built-in JSON serialization
- **Microsoft.Data.Sqlite**: Embedded database for tracking and audit
- **Microsoft.Extensions.DependencyInjection**: Dependency injection
- **Microsoft.Extensions.Logging**: Logging abstractions

## Next Steps

1. Read [02-architecture-and-design.md](02-architecture-and-design.md) for architectural overview
2. Review [03-data-models-and-configuration.md](03-data-models-and-configuration.md) for configuration structure
3. Study [04-authentication-oauth2.md](04-authentication-oauth2.md) for Gmail OAuth2 implementation
4. Understand [05-rule-engine-specification.md](05-rule-engine-specification.md) for rule processing logic
5. Reference remaining documents for implementation details
