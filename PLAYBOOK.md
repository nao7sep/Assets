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

### 7.1. What is a "Segment"?

A **segment** is a cohesive group of related types that share a common concern or responsibility. Examples:
- All classes for line handling (`LineHandler`, `LeadingWhitespaceProcessor`, `TrailingWhitespaceProcessor`)
- All classes for word processing (`WordSplitter`, `WordValidator`, `WordNormalizer`)
- All classes for a specific feature domain (authentication, file storage, email delivery)

A segment typically represents what would go in a subdirectory/namespace grouping.

### 7.2. When User Requests "Bit by Bit" or "Step by Step"

1. **Identify all segments**: Break down the full implementation into logical, cohesive segments
2. **Present segment overview**: Show the list of all segments and their relationships
3. **Detail one segment at a time**: For the first segment, show the **complete picture**:
   - List all types that will be in this segment
   - Show their responsibilities and relationships
   - Explain why these types form a clearly separated concern
4. **Wait for evaluation**: User reviews whether this segment is properly isolated
5. **Implement the segment**: Only after approval, implement all types in that segment
6. **Move to next segment**: Repeat steps 3-5 for each remaining segment

**Example flow:**
- "I have a plan for 15 classes across 3 segments: [1] LineHandling, [2] WordHandling, [3] FileProcessing. Should we start with LineHandling? It would include: `LineReader`, `LineNormalizer`, `EmptyLineFilter` - these handle all line-level text operations. Here's the detailed design..."

### 7.3. Proactive Segmentation

Even without user prompting, if a task involves multiple loosely-coupled groups of types, propose segmentation:

**Example:** "This implementation involves 12 classes. I can organize them into 3 segments: [1] Domain models, [2] Data access layer, [3] Business services. Should I show you the LineHandling segment design first?"

### 7.4. Segment Design Quality Check

**Good segment:** Types within it are highly related to each other, loosely coupled to other segments, and share a clear unified purpose.

**Bad segment (red flag):** Segment A depends on segment B, AND segment B depends on segment A → circular dependency detected. Restructure the design before implementing.

---

## Summary

These patterns ensure:
- Clear separation between business logic and external systems
- Loose coupling through dependency injection
- Predictable error handling boundaries
- Organized, maintainable file structure
- Pragmatic balance between consistency and productivity
- Thoughtful implementation planning for complex work
