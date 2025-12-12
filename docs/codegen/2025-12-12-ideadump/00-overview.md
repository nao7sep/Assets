# IdeaDump - Documentation Overview

**Date**: 2025-12-12
**Purpose**: Complete specification for building IdeaDump, a Blazor Server web application that captures fragmented ideas, uses AI to elaborate them into actionable reports, and emails the results to registered recipients.

## Problem Statement

When ideas are captured in todo apps and left for weeks, the main barrier is the gap between a rough idea and actionable tasks. IdeaDump solves this by:

- Accepting short, fragmented idea notes from users
- Using AI to elaborate ideas into detailed reports (research, analysis, tasks, schedules, success likelihood)
- Automatically emailing elaborated reports to registered recipients
- Making ideas interesting to revisit rather than forgotten

## Document Index

| Document | Description |
|----------|-------------|
| [00-overview.md](00-overview.md) | This file - project summary and navigation |
| [01-architecture-and-design.md](01-architecture-and-design.md) | High-level architecture, Blazor Server design, background processing |
| [02-data-models-specification.md](02-data-models-specification.md) | Entity models, database schema, relationships |
| [03-authentication-and-authorization.md](03-authentication-and-authorization.md) | User registration, email confirmation, identity management |
| [04-channel-and-recipient-management.md](04-channel-and-recipient-management.md) | Channel CRUD, recipient confirmation, consent management |
| [05-ai-integration-specification.md](05-ai-integration-specification.md) | AI elaboration workflow, API key hierarchy, rate limiting |
| [06-email-smtp-specification.md](06-email-smtp-specification.md) | SMTP configuration, sending logic, retry mechanisms |
| [07-ui-blazor-design.md](07-ui-blazor-design.md) | Page structure, components, real-time updates |
| [08-admin-features-specification.md](08-admin-features-specification.md) | Admin dashboard, logging, user limit management, system health |
| [09-project-structure-and-dependencies.md](09-project-structure-and-dependencies.md) | Solution structure, NuGet packages, folder organization |

## Key Features

### User Management
- **Registration & Email Confirmation**: Users register with email and password, receive confirmation link
- **Multi-email Support**: Users can register multiple email addresses
- **Account Management**: Profile settings, API key configuration, account deletion

### Channel System
- **Channel Creation**: Users create channels (formerly "profiles") to organize ideas
- **Custom AI Instructions**: Each channel can have specific AI elaboration instructions
- **Recipient Management**: Add/remove recipients per channel with explicit consent
- **API Key Hierarchy**: System-wide → User-wide → Channel-wide key inheritance

### Idea Processing Workflow
1. **User adds fragmented idea** (simple text, no title required)
2. **Background AI elaboration** generates detailed report
3. **Automatic email delivery** to all confirmed channel recipients
4. **Status tracking**: New → Elaborating → Elaborated → Sent → (or Failed)

### AI Elaboration
- **Background Processing**: Non-blocking, Blazor Server real-time updates
- **Configurable Instructions**: Default system instructions or custom per channel
- **Rate Limiting**: Daily limits per user, configurable by admin
- **Multi-tier API Keys**: System key (default) → User key (override) → Channel key (specific)

### Email Distribution
- **SMTP Configuration**: Per-channel email sender settings
- **Recipient Confirmation**: Double opt-in (system-wide + per-channel consent)
- **Automatic Sending**: Emails sent immediately after elaboration completes
- **Retry Logic**: Failed sends are retried with exponential backoff

### Admin Features
- **System Logging**: Comprehensive audit trail viewable on admin dashboard
- **User Limit Management**: Adjust rate limits per user
- **Health Monitoring**: Track AI service status, SMTP failures
- **Smart Notifications**: Critical error alerts with throttling to avoid spam

## Technology Stack

- **.NET 10.0**
- **Blazor Server** (real-time UI updates via SignalR)
- **SQLite** for data storage (users/channels metadata, per-channel entry history)
- **Entity Framework Core** for data access
- **ASP.NET Core Identity** for authentication
- **GitHub Models / Azure OpenAI** for AI elaboration (configurable)
- **SMTP** for email delivery
- **xUnit** for testing

## Scope

### In Scope (v0.1)
- User registration with email confirmation
- Channel CRUD operations
- Recipient management with consent workflow
- Simple text idea submission
- Background AI elaboration
- Automatic email sending
- Rate limiting and API key hierarchy
- Admin dashboard for logging and user management
- Basic error handling and retry logic

### Out of Scope (v0.1)
- Multi-modal file uploads (extensibility considered, not implemented)
- IMAP integration (no reading recipient replies)
- Search/filter functionality
- Tagging or categorization
- Export features
- Idea editing (immutable once submitted)
- Idea drafts (submissions are immediately processed)

### Future Considerations
- Multi-modal idea inputs (images, documents for AI analysis)
- Redis caching for performance at scale
- PostgreSQL migration if user base grows
- Mobile app
- Webhook integrations

## Design Principles

1. **Start Small and Stupid**: Avoid scope creep, focus on core value
2. **Immutable Ideas**: Once submitted, ideas cannot be edited (like Twitter)
3. **AI-First Design**: AI does the heavy lifting (titles, elaboration, actionable outputs)
4. **Real-time Experience**: Blazor Server provides immediate feedback as elaboration progresses
5. **Consent-Driven**: Explicit recipient consent at system and channel level
6. **Admin Control**: Flexible rate limits and cost management for personal/family use
7. **Fail Gracefully**: Comprehensive logging and retry mechanisms for resilience

## Deployment Target

- **Azure App Service** (Blazor Server)
- **SQLite** (file-based, suitable for small-scale personal use)
- Expected user scale: ~10-100 users (personal/family tool)

## Non-Functional Requirements

- **Performance**: Background AI elaboration should not block UI
- **Reliability**: Retry failed AI calls and email sends with exponential backoff
- **Security**: Encrypted passwords, secure API key storage, email confirmation
- **Observability**: Comprehensive logging for admin troubleshooting
- **Cost Management**: Rate limiting and API key hierarchy to control AI usage costs
- **Simplicity**: Minimal configuration, intuitive UI, no unnecessary features

## Next Steps for Implementation

1. Set up Entity Framework Core models and Identity
2. Implement user registration and email confirmation
3. Build channel CRUD and recipient consent workflow
4. Implement background AI elaboration service
5. Integrate SMTP email sending with retry logic
6. Create Blazor Server UI with real-time updates
7. Build admin dashboard for logging and user management
8. Write unit tests for core logic
9. Deploy to Azure App Service

---

**Note**: This documentation set is designed to guide AI implementation. Each document provides detailed specifications for its respective domain.
