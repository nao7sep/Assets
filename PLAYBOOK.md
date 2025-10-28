# The AI Coding Playbook

## 1. Introduction

This document is your primary instruction set for all coding and code review tasks. You are an AI assistant, and this playbook is designed to be your single source of truth. Adhering to these guidelines is mandatory. The goal is to produce clean, maintainable, and high-quality code. Remember, coding should be fun, and following these rules will help us achieve that by creating a predictable and professional codebase.

## 2. Language and Communication

Clarity in communication, both in code and in conversation, is paramount.

* **User Interaction**: All conversations with the user (this chat) must be in English.
* **Prompts**: All prompts, which are often user-facing settings or instructions, must be in English. They are closely related to code, which is in English, and this ensures better understanding and performance from AI assistants.
* **Source Code Comments**: All generated source code comments (e.g., `//`, `/// <summary>`) must be in **Japanese**. This is because the primary human developer is a native Japanese speaker. These comments are static and can be easily auto-translated if needed in the future.
* **Internal and User-Facing Text**: All other text, including log messages, exception messages, console output, and internal metadata, must be in **English**. This is because this text can be exposed to non-Japanese speaking users or system administrators.

## 3. Architecture and Design Principles

This section defines the high-level architecture that guides all implementation.

### 3.1. Separation of Concerns (SoC)

This is the most important principle. Each class, method, and module must have a single, well-defined responsibility. If a component handles multiple, unrelated concerns, it must be split into more focused units. Do not hesitate to create many small, single-purpose types. The cost of creating more types is low with AI assistance, and the long-term benefits of high cohesion and low coupling are significant.

### 3.2. Domain-First (Anti-Corruption Layer)

Our architecture is built on a strict separation between the internal **Domain (business logic)** and the external **Infrastructure (data sources)**. This is achieved by separating Domain Models from Data Transfer Objects (DTOs) using an **Anti-Corruption Layer**.

* **Domain Model:**

    * **Role:** Represents a core business concept (e.g., `Book`, `User`). This class contains business logic and properties.
    * **Purity:** Must be a "pure" POCO (Plain Old C# Object). It must **never** contain any attributes related to external systems (e.g., no `[JsonPropertyName]`, `[XmlElement]`, `[Table]`).
    * **Naming:** Must have the simplest, cleanest name that reflects the business concept (e.g., `Book`), with no suffixes.

* **Data Transfer Object (DTO):**

    * **Role:** A "data-only" class whose *only* job is to match the exact data structure of an external contract (e.g., a JSON API, an XML file, a database table).
    * **Fidelity:** DTOs *must* be specific to each external source. For example, `PublicApiBookDto` and `LibraryFileBookDto` must be different classes that perfectly match their respective external specs. Do not try to find a common base class or interface for DTOs from different vendors.
    * **Logic:** This is the *only* place serialization/data attributes (e.g., `[JsonPropertyName]`) should exist. DTOs must **never** contain business logic.
    * **Naming:** Must use a clear suffix (e.g., `*Dto`, `*Request`, `*Response`).

* **Mapper (The "Translator"):**

    * **Role:** A static class (e.g., `CloudBookMapper`) or private method that acts as the "bridge" (Anti-Corruption Layer).
    * **Responsibility:** Its *only* job is to translate between a specific DTO and its corresponding Domain Model. This is the **only** place that should know about both the DTO and the Domain Model. This layer is *designed* to contain the "messy" translation logic.

* **The Strict Dependency Rule:**

    * Data flow is **one-way**: `DTO` -> `Mapper` -> `Domain Model`.
    * The Domain Model must **never** hold a reference to a DTO, not even through an interface.
    * If your domain logic needs a value from the DTO, that value must be **copied** onto a property of the Domain Model during mapping. The DTO is then discarded.

### 3.3. Directory Structure (Vertical Slice)

All files related to a single feature or external system must be grouped together in a "vertical slice." This maximizes cohesion and provides clear context. Use `Infrastructure` for all external-facing code.

```
/YourProject
  ├─ /Domain                        // Your pure Domain Models
  │   └─ Book.cs                    // (e.g., public class Book)
  │
  └─ /Infrastructure                // All external systems (DB, APIs, files)
      │
      ├─ /LocalLibrary              // "Vertical Slice" for one feature
      │   ├─ /Dtos
      │   │   └─ LibraryFileBookDto.cs
      │   ├─ LocalBookMapper.cs     // Mapper
      │   └─ LocalLibraryService.cs // Service that uses the DTO/Mapper
      │
      └─ /PublicBookApi             // "Vertical Slice" for another feature
          ├─ /Dtos
          │   └─ PublicApiBookDto.cs
          ├─ CloudBookMapper.cs
          └─ PublicBookApiService.cs
```

## 4. C# Implementation Details

### 4.1. File and Project Structure

* **One Type Per File**: Each C# file must contain only one public type.
* **File Naming**: The file name must exactly match the name of the public type it contains (e.g., class `MyClass` in `MyClass.cs`).
* **No Generated File Headers**: Do not add any headers to source files. This includes, but is not limited to, license information, author names, file paths, or any other metadata.
* **Test Project Structure**: The folder structure of a test project must mirror the folder structure of the project it is testing.

### 4.2. Code Organization

* **Namespaces**: Always use bracketed namespaces (`namespace MyNamespace { ... }`). Do not use file-scoped namespaces. The namespace must match the directory structure (e.g., `YourProject.Infrastructure.LocalLibrary.Dtos`).
* **Class Member Organization**: Position new class members in a logical and consistent order. A good default order is: public fields, private fields, constructors, public properties, public methods, and finally private methods.

## 5. Naming Conventions

* **Descriptive Names**: Avoid cryptic names like `dt`, `s`, or `str`. Use descriptive names like `dateTime`, `result`, or `connectionString`.
* **Path vs. Name**: Variable names must clearly distinguish between paths and names. Use the suffix 'Path' for paths (e.g., `directoryPath`) and 'Name' for names (e.g., `fileName`).
* **Object Type Clarity**: Variable names for complex objects should reflect the object's type. For example, use `FileInfo fileInfo` instead of `FileInfo file`.

## 6. Data Handling and Validation

* **String Validation**: Always use `string.IsNullOrWhiteSpace` for validation. While it is acceptable to use the word "empty" in identifiers and variable names for readability (e.g., `if (userInput.IsEmpty)`), the underlying implementation should use `string.IsNullOrWhiteSpace`. This ensures that strings containing only whitespace are correctly handled. The validation should happen before the string is trimmed and used.
* **Path Validation**: Use `Path.IsFullyQualified` to validate paths. However, this is not required for every method that accepts a path. Enforce validation in cases where an invalid path might be handled without an immediate exception, leading to incorrect behavior (e.g., `c:\dir1\d\dir2\file`). In cases where an invalid path would obviously cause an exception, explicit validation is not strictly necessary.
* **Null Semantics**: Use `null` to represent a value that is not set or is unclear. Use an empty string (`""`) to represent a value that is explicitly set as empty.
* **Enum Value Validation**: When an enum value is *used* (not just stored), it must be validated. Use a `switch` statement with a `default` case that throws an exception for undefined values. Do not assume that if a value is not `A`, it must be `B`.
* **Lazy Initialization**: Use `Lazy<T>` for delayed initialization when the initialized value may be `null`, or when the value can become `null` again after initialization.

## 7. Asynchronous Programming

* **ConfigureAwait**: All `await` calls must use `.ConfigureAwait(false)`.
* **CancellationToken**: Async methods should accept a `CancellationToken` parameter where cancellation is meaningful.
* **Asynchronous Operations**: I/O-bound and computationally intensive operations should be implemented asynchronously.

## 8. Documentation and Comments

* **XML Documentation Comments**:
    * Use `/// <summary>` for all public members (in **Japanese**).
    * Use the `<see>` tag to reference known types when they are mentioned in comments.
    * Do not use HTML formatting tags like `<b>` or `<i>`. Plain text is sufficient.
* **Comment Formatting**: Wrap long comment lines to maintain readability, with a soft limit of 120 characters.

## 9. Testing

* **Test Method Structure**: Test methods should follow the "Arrange, Act, Assert" pattern, with corresponding English section comments.
* **Edge Case Coverage**: Ensure that complex logic and boundary conditions have corresponding test coverage.
* **Testing Necessity**: Not all code requires rigorous testing. Focus on complex algorithms and edge cases.

## 10. Code Review

When asked to review code, use this playbook as your checklist. For each point, verify that the code complies with the standards outlined in this document. Be proactive in suggesting refactoring. Even if the code is well-designed, if you see an opportunity to improve separation of concerns (per Section 3) by splitting types, you should propose the refactoring. It is never too late to improve the code structure.
