# The AI Coding Playbook

This document defines the core patterns and rules for all coding tasks. These are mandatory architectural and organizational principles that ensure clean, maintainable code.

---

## 1. Architectural Patterns

### 1.1. Separation of Concerns

Each class, method, and module must have a single, well-defined responsibility. If a component handles multiple unrelated concerns, split it into focused units. Create many small, single-purpose types rather than fewer large multi-purpose ones.

### 1.2. Domain-First with Anti-Corruption Layer

Maintain strict separation between internal business logic and external systems through three distinct layers:

**Domain Model:**
- Represents core business concepts (e.g., `Book`, `User`)
- Contains business logic and validation rules
- Must be a pure POCO with no external system attributes (no `[JsonPropertyName]`, `[XmlElement]`, `[Table]`)
- Uses the simplest, cleanest name reflecting the business concept

**Data Transfer Object (DTO):**
- Data-only class matching the exact structure of an external contract (API, file format, database schema)
- Must be specific to each external source (no shared DTOs across different vendors)
- The only place where serialization/data attributes should exist
- Contains zero business logic
- Uses clear suffixes: `*Dto`, `*Request`, `*Response`

**Mapper:**
- Static class or method that translates between DTO and Domain Model
- The only component that knows about both DTO and Domain Model
- Performs validation and transformation during translation
- Contains the "messy" translation logic by design

**Dependency Rule:**
- Data flows one-way: `DTO` → `Mapper` → `Domain Model`
- Domain Model never references DTOs, not even through interfaces
- Values needed by domain logic must be copied onto Domain Model properties during mapping

---

## 2. Dependency Injection

- Use constructor injection for all dependencies
- Register services via their interfaces, never concrete types (e.g., `services.AddScoped<IBookRepository, SqlBookRepository>()`)
- This enables Domain/Infrastructure separation by allowing domain layer to depend on abstractions

---

## 3. Error Handling

**In Service/Business Layers:**
- Allow built-in .NET exceptions to bubble up naturally
- Throw exceptions when code would fail silently or behave incorrectly
- Prefer built-in .NET exceptions (e.g., `InvalidOperationException`, `ArgumentNullException`)
- Only create custom exceptions when no built-in exception accurately describes the error

**At Application Boundaries:**
Catch and handle exceptions only at the boundary where outcomes are communicated to users or external systems:

- **Web API:** Global exception middleware that logs and translates to HTTP responses
- **Desktop UI:** ViewModel layer (typically in command handlers)
- **Console Applications:** Context-dependent (command handlers for complex apps, top-level for simple apps)

---

## 4. File Organization

### 4.1. Type and File Rules

- **One type per file** (mandatory - do not create files with multiple types)
- File name must exactly match the type name (e.g., class `MyClass` in `MyClass.cs`)
- No headers, timestamps, copyright notices, or "change assist" comments in source files

### 4.2. Namespace and Directory Structure

- Namespace structure must match folder structure
- For C# projects: Always set `<RootNamespace>` explicitly in `.csproj` files
- Project folder names should not contain dots (Mac executable compatibility)
- Root namespace uses dots for hierarchy (e.g., `Nekote.Sandbox`)

---

## 5. Scope-Based Pragmatism

**Principle:** Don't pursue perfection across the entire codebase. Each feature or module should be internally coherent and well-structured within its own boundaries.

**In Practice:**
- If working on authentication, that code should be clean and consistent
- If working on file processing, same standard applies
- These areas don't need to match each other's patterns exactly
- Finishing working features is more valuable than achieving uniformity across all files
- Ship quality code in sections rather than refactoring everything to match a global standard
- Move forward, not in circles

---

## 6. Code Formatting

- Wrap lines at meaningful places (statement ends, logical breaks), not at arbitrary character counts
- Modern screens are wide - prioritize readability over historical 80-character limits
- If code or comments become too long, developers will wrap them naturally

---

## 7. Incremental Implementation Pattern

### 7.1. When User Requests "Bit by Bit" or "Step by Step"

1. **Design first, implement later**: Create a structured segment plan where each segment is self-contained
2. **Dependency ordering**: Order segments so each either:
   - Compiles independently, OR
   - Only depends on previously implemented segments
3. **Present for review**: Show segment structure and dependencies before implementing
4. **Implement sequentially**: After approval, implement one segment at a time, ensuring each compiles successfully

### 7.2. Proactive Segmentation

Even without user prompting, if a task involves multiple loosely-coupled concerns or layers, propose segmentation:

**Example:** "I can implement this in 3 segments: [1] Domain models (compiles independently), [2] Repository interfaces (depends on #1), [3] Service layer (depends on #1 and #2). Proceed with this structure?"

### 7.3. Design Red Flag

If segment A depends on segment B, AND segment B depends on segment A → circular dependency detected. Restructure the design before implementing.

---

## Summary

These patterns ensure:
- Clear separation between business logic and external systems
- Loose coupling through dependency injection
- Predictable error handling boundaries
- Organized, maintainable file structure
- Pragmatic balance between consistency and productivity
- Thoughtful implementation planning for complex work
