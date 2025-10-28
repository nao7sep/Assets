# The Play Harder Playbook

This document is the companion to the `AI Coding Playbook`. The first playbook defines foundational C# and architectural rules (e.g., Domain vs. DTO). This document defines the specific, mandatory rules for *building* application "heads" (e.g., UI, Web API, Console).

When we are working on a specific application type, the rules from **both** playbooks are in effect and must be followed.

## 1. API Design: Hybrid (REST/RPC) Approach

This section defines the mandatory design model for all web-facing APIs. This is a pragmatic hybrid, not a "pure" academic approach.

* **Textbook REST (Nouns):** The API is a collection of resources. You use HTTP verbs (GET, POST, PUT, DELETE) to manipulate them (e.g., `POST /api/orders` to create an order).
* **Textbook RPC (Verbs):** The API is a list of actions. The endpoint name is the function (e.g., `POST /api/buyBook`).
* **Our "Real-World" Hybrid Approach (Mandatory):**
    1.  **Rule:** You **must** use the **REST** style for simple, resource-based CRUD operations (e.g., `GET /api/books`, `POST /api/books`, `GET /api/books/123`).
    2.  **Rule:** You **must** use the **RPC** style for complex, state-changing business actions that are not simple CRUD (e.g., `POST /api/orders/123/cancel`, `POST /api/users/456/activate`).
* **Security (Non-Negotiable):**
    * **Rationale:** The choice between REST and RPC is a URL naming convention *only*. It has **zero impact on security**.
    * **Rule:** All business logic (checking permissions, validating funds, confirming state) **must** be on the server. The server must **never** trust the client.

## 2. Dependency Injection (DI)

* **Framework:** All applications **must** be built using the `Microsoft.Extensions.DependencyInjection` host (e.g., `Host.CreateDefaultBuilder()`). This ensures DI is available everywhere.
* **Method:** All dependencies (services, repositories) **must** be injected via **constructor injection**.
    * **Rationale:** This makes all class dependencies explicit and self-documenting. It is critical for testability and loose coupling.
* **Registration:** Services **must** be registered using their interfaces.
    * **Example:** `services.AddScoped<IBookRepository, SqlBookRepository>();`
    * **Rationale:** This is the core mechanism that enables the `Domain` / `Infrastructure` separation from the core playbook. The service layer depends on `IBookRepository` (Domain) and is completely unaware of `SqlBookRepository` (Infrastructure).
* **Lifetimes (Default Rules):**
    * **Singleton:** For services that are truly stateless and shared globally (e.g., `ILogger`, `IConfiguration`).
    * **Scoped:** This is the **default lifetime for most services** (e.g., repositories, unit of work, business services).
        * **Rationale:** The service instance persists for the duration of a single "unit of work," such as one HTTP request in an API or one complex user action in a UI. This allows services within that operation to share state (like a `DbContext`) safely.
    * **Transient:** For lightweight, stateless services that are created and destroyed frequently. Use this sparingly.

## 3. Configuration and Logging

* **Configuration:**
    * **Rule:** All settings (connection strings, API keys) **must** be read via the `IConfiguration` interface (injected via DI).
    * **Anti-Pattern:** Hard-coded strings for configuration are forbidden.
    * **Rationale:** `IConfiguration` abstracts the *source* of the configuration (e.g., `appsettings.json`, Environment Variables, Azure Key Vault), allowing the application to be environment-agnostic.
* **Logging:**
    * **Rule:** All logging **must** be performed via the `ILogger<T>` interface (injected via DI).
    * **Anti-Pattern:** Using `Console.WriteLine()` for logging in services or controllers is forbidden.
    * **Rationale:** `ILogger<T>` abstracts the *destination* of the logs (e.g., Console, File, Application Insights). This allows logging to be configured for different environments without changing application code.

## 4. Application-Level Error Handling

* **Throwing (Service Layer):**
    1.  Allow built-in .NET exceptions (e.g., `DbUpdateException`, `IOException`) to bubble up naturally. Do not `catch` them in the service layer.
    2.  If code would "fail silently" but is logically wrong (like the "Path Validation" rule in the core playbook), you **must** explicitly `throw` a new exception.
    3.  When throwing, **first** try to use a suitable, common, built-in .NET exception (e.g., `InvalidOperationException` for bad state, `ArgumentNullException` for bad input).
    4.  Only create a new custom exception class if no .NET exception accurately describes the error.
* **Catching (The Boundary):**
    * **Core Principle:** Exceptions **must** only be caught at the *application boundary*. This is the top-most layer where the outcome of an operation is communicated to the user or calling system.
    * **Rationale:** Business services and repositories should not know *how* an error should be presented. A service shouldn't know if it's being called by a Web API (which needs a `404 Not Found` JSON response) or a Console App (which needs a console error message). By only catching at the boundary, the application "head" can translate the exception into the correct response for its context.
    * **The Boundaries Defined:**
        * **Avalonia:** The "boundary" is the **ViewModel** (e.g., in an `ICommand`'s `try/catch` block).
        * **Console:** The "boundary" is a top-level `try/catch` in `Program.cs`.
        * **Web API:** The "boundary" is **Middleware**.
* **Web API Middleware:**
    * **Rule:** This is the superior implementation of the "boundary" rule for Web APIs. A global `ExceptionHandlingMiddleware` **must** be implemented.
    * **Responsibility:** This middleware will be the **single `try/catch` block** for the entire application.
    * **Logic:** It must log the exception via `ILogger` and then map the exception to a standard JSON error response and the correct HTTP status code (e.g., `ArgumentException` -> `400 Bad Request`, `EntityNotFoundException` -> `404 Not Found`, all others -> `500 Internal Server Error`).

## 5. Data Access (Repository Pattern)

* **Pattern:** All data access (to databases, files, external APIs) **must** be abstracted using the **Repository Pattern**.
* **Interface (Domain Layer):** The repository interface (e.g., `IBookRepository`) **must** be defined in the `Domain` layer.
    * **Rationale:** This interface is the "contract" or "menu" of what data operations the domain logic is *allowed* to perform (e.g., `GetByIdAsync`, `AddAsync`).
* **Implementation (Infrastructure Layer):** The concrete implementation (e.g., `SqlBookRepository`) **must** be in the `Infrastructure` layer.
    * **Rationale:** This class contains the "how" (e.g., the `_dbContext` logic). It is an implementation detail that the rest of the application, per the Core Playbook, must not know about.
* **Abstraction Metaphor (The "Bartender"):**
    * **Purpose:** This metaphor is your guide for designing the abstraction.
    * The `IRepository` interface is the **counter**.
    * The "customer" (your service layer) can *only* ask the "bartender" (the interface) for what's on the menu (the methods like `GetByIdAsync`).
    * The "customer" (service) cannot see or go *behind* the counter.
    * The implementation (`SqlBookRepository`) is the **back room**. It's completely invisible to the customer and contains all the data-access logic (e.g., `_dbContext.Books.Add`). The customer doesn't know or care if the "back room" uses EF Core, Dapper, or a text file.
* **Unit of Work:**
    * **Rule:** For database operations, the `IUnitOfWork` interface **must** be used to manage transactions.
    * **Implementation:** It is typically injected into services that need to *write* data. It exposes the `SaveChangesAsync()` method.
    * **Rationale:** This allows a single business operation (e.g., "Create Order") to use *multiple* repositories (e.g., `IOrderRepository`, `IProductRepository`) and then save all changes as a single, atomic transaction by calling `_unitOfWork.SaveChangesAsync()` once at the end.

## 6. Database Schema Management

* **Framework:** The standard for all database interactions is **Entity Framework Core (EF Core)**.
* **Method:** All database schema changes **must** be managed using EF Core's **Code-First Migrations** feature.
    * **Anti-Pattern:** Manually editing the database schema (e.g., via an SQL GUI) is forbidden. Using "Database-First" is forbidden.
    * **Rationale:** The C# "Model" (defined in your `Infrastructure` DTOs or EF Core entities) is the **single source of truth**. Migrations provide a consistent, version-controlled, and repeatable way to evolve the database schema to match the code.
* **Process:** Any change to a model class that is persisted to the database **must** be accompanied by a new, named migration file (e.g., `dotnet ef migrations add AddUserEmail`).

## 7. Application-Level Validation

* **Web API (DTOs):**
    * **Library:** **FluentValidation** is the mandatory library for validating all incoming DTOs (Data Transfer Objects).
    * **Execution:** Validation rules **must** be defined in a validator class for the DTO. These rules will be triggered automatically by the ASP.NET Core pipeline *before* the controller action is executed.
    * **Rationale:** This enforces the "Separation of Concerns" principle from the core playbook. Validation is a distinct responsibility. By using this library, validation logic is kept out of the controller and out of the business service. The controller action should only execute *after* it knows the incoming data is syntactically valid.
* **Avalonia UI (ViewModels):**
    * **Responsibility:** ViewModels are solely responsible for validating user input.
    * **Interface:** ViewModels **must** implement the `INotifyDataErrorInfo` interface.
    * **Rationale:** This interface is the standard MVVM mechanism for providing real-time, property-level validation feedback to the View (e.g., "This field is required" appearing in red). This keeps all UI-related validation logic in the ViewModel, honoring the "zero code-behind" rule for the View.

## 8. Authentication & Authorization

This section defines the mandatory technology choices for identity.

* **Standard (External):** For standard external providers (e.g., Google, Microsoft, Twitter), you **must** use the built-in **OAuth** support in ASP.NET Core.
* **Custom (Internal):** For custom email/password, email-link login, and account confirmation, you **must** use **ASP.NET Core Identity**.
    * **Rationale:** Do not "roll your own" security. ASP.NET Core Identity is the battle-tested, secure, and maintained framework for handling user management, password hashing, and token generation.
* **Cross-Application (Enterprise):** When a single identity is required across multiple applications within an organization, you **must** use **Microsoft Entra ID** (formerly Azure Active Directory).

## 9. Caching Strategy

* **Principle:** Caching is an **Infrastructure** concern.
    * **Rationale:** Cached data is a performance optimization, not a source of business truth. It is a copy of data, and its management (e.g., cache invalidation) is a technical detail. It must be implemented within the `Infrastructure` layer, often as part of a Repository implementation (e.g., a `CachedBookRepository` decorator).
* **In-Memory:** Use `IMemoryCache` (injected) for simple, non-critical, server-specific data.
    * **Use Case:** Good for data that is expensive to get but can be slightly stale (e.g., a list of categories). This cache is local to each server instance.
* **Distributed:** Use `IDistributedCache` (injected) for data that must be shared and consistent across multiple server instances (e.g., in a load-balanced web farm).
    * **Default Implementation:** The default implementation for this is **Redis**.

## 10. UI Architecture (Avalonia)

* **Pattern:** The **MVVM (Model-View-ViewModel)** pattern **must** be strictly followed.
    * **Rationale:** This pattern is the ultimate expression of "Separation of Concerns" (from the core playbook) for a UI. It rigidly separates the *what it looks like* (View) from the *how it works* (ViewModel) and the *what it is* (Model).
* **Model:** This is the pure **Domain Model** from the `Domain` layer (e.g., `Book`). It contains business data and logic, but zero UI concerns.
* **View:**
    * **Definition:** The `.axaml` file.
    * **Rule:** The View **must** contain **zero** C# code-behind logic (e.g., no `Button_Click` event handlers, no manual manipulation of controls).
    * **Rationale:** The View's *only* job is to declare the UI layout and bind to properties on its ViewModel. All logic, state, and interaction are handled by the ViewModel. This makes the View "dumb" and easily replaceable.
* **ViewModel:**
    * **Definition:** The `*ViewModel.cs` class (e.g., `BookViewModel`).
    * **Rule:** This class contains *all* UI logic and state (e.g., "Is this button enabled?", "What happens when I click 'Save'?", "The text for this label is...").
    * **Rule:** The ViewModel **must never** have a direct reference to its View (e.g., no `using Avalonia.Controls;`).
    * **Rationale:** This decoupling allows the ViewModel to be tested "headless" (without a UI), which is critical for unit testing.

## 11. UI Communication (Avalonia)

* **View-to-ViewModel (Commands):**
    * **Rule:** All View actions (e.g., `Button.Click`) **must** be handled by data-binding to an `ICommand` property on the ViewModel.
    * **Implementation:** You **must** use the `RelayCommand` from the `CommunityToolkit.Mvvm` library to implement these commands.
    * **Rationale:** Using `ICommand` is the core mechanism that enables the "zero code-behind" rule for the View. It replaces old event handlers (like `Button_Click`) with a testable, bindable object on the ViewModel.
* **ViewModel-to-ViewModel (Messaging):**
    * **Anti-Pattern:** ViewModels **must not** have direct references to each other (e.g., `MainViewModel` must not have a property `public DetailViewModel MyDetailVm { get; }`).
    * **Rationale:** Direct references create high coupling, making the ViewModels difficult to test and reuse independently.
    * **Rule:** Communication between ViewModels (e.g., a "List" ViewModel telling a "Detail" ViewModel "The user selected item 5") **must** be done using a decoupled **Event Aggregator / Message Bus**.
    * **Implementation:** You **must** use the `IMessenger` interface from the `CommunityToolkit.Mvvm` library for this.

## 12. Web API Architecture

* **Role: "Thin Controllers"**
    * **Rule:** API Controllers must be **"thin."** Their *sole* responsibility is to handle the HTTP request/response cycle.
    * **Logic:** A controller action should typically only contain:
        1.  Deserialization of the incoming DTO (done by the framework).
        2.  A single call to a business service (e.g., `_bookService.CreateBookAsync(...)`).
        3.  Returning the HTTP response (e.g., `Ok()`, `CreatedAtRoute()`).
    * **Anti-Pattern:** Controllers **must never** contain business logic (e.g., `if/else` statements about business rules, database lookups, data manipulation).
    * **Rationale:** This is a critical enforcement of "Separation of Concerns." The Controller's concern is **HTTP**. The Service's concern is **Business Logic**.
* **Data Contract (Non-Negotiable):**
    * **Rule:** Controllers **must always** accept and return **DTOs** (per the Core Playbook, Sec 3.2).
    * **Rule:** Controllers **must never, under any circumstances,** expose internal **Domain Models** to the public.
    * **Rationale:** This is the most important application of the "Anti-Corruption Layer" from the core playbook. Exposing your internal Domain Model in an
        API:
        1.  **Breaks encapsulation:** It shows the outside world your internal business rules.
        2.  **Creates high coupling:** Any change to your internal domain model (e.g., renaming a property) now breaks your public API contract.
        3.  **Creates security risks:** It can inadvertently expose data that should be private (e.g., a `User` model's `PasswordHash` property).
    * The `Mapper` is responsible for translating between the `DTO` and the `Domain Model` inside the service or just before returning from the controller.

## 13. Pattern Clarification: CQRS vs. MVVM ICommand

* **Rationale:** The word "Command" is used in two completely different patterns. This section exists to prevent you from confusing them.
* **CQRS (Command Query Responsibility Segregation):**
    * **Definition:** The "enterprise" pattern of creating formal `Command` objects (e.g., `CreateBookCommand.cs`) that are processed by separate `CommandHandler.cs` classes, often using a "mediator" library.
    * **Rule:** The use of this full CQRS pattern (with mediator, separate command/handler classes) is **prohibited**. It is considered overkill for our project scale.
* **MVVM `ICommand`:**
    * **Definition:** A simple interface (e.g., `System.Windows.Input.ICommand`) used for UI data binding. It is a property on a ViewModel (e.g., `public ICommand SaveBookCommand { get; }`) that a XAML `Button` can bind to.
    * **Rule:** The use of the `ICommand` interface is **mandatory** for our MVVM architecture (per Section 11).
* **Summary:** They are not related. You **must** use `ICommand` for UI. You **must not** use the CQRS pattern for business logic.
