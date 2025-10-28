# The Play Harder Playbook

This document is the companion to your primary `AI Coding Playbook`. The first playbook defines our foundational C# and architectural rules (Domain vs. DTO, etc.). This document defines the specific rules for *building* applications (e.g., UI, Web API, Console).

When we are working on a specific application type, the rules from both playbooks are in effect.

## 1. Reference: API Design (REST vs. RPC vs. Hybrid)

This is a reference for our "hybrid and logical" approach.

* **Textbook REST (Nouns):** The API is a collection of resources. You use HTTP verbs (GET, POST, PUT, DELETE) to manipulate them (e.g., `POST /api/orders` to create an order).
* **Textbook RPC (Verbs):** The API is a list of actions. The endpoint name is the function (e.g., `POST /api/buyBook`).
* **Our "Real-World" Hybrid Approach:**
    1.  We will use **REST** for simple, resource-based CRUD operations (e.g., `GET /api/books`, `POST /api/books`).
    2.  We will use **RPC** for complex, specific business actions that aren't simple CRUD (e.g., `POST /api/orders/123/cancel`).
* **Security:** This choice is about URL design and has **zero impact on security**. All business logic (checking funds, permissions, etc.) **must** be on the server. The server must never trust the client.

## 2. Dependency Injection (DI)

* **Framework:** All applications must be built using the `Microsoft.Extensions.DependencyInjection` host.
* **Method:** All dependencies (services, repositories) must be injected via **constructor injection**.
* **Registration:** Services must be registered using their interfaces (e.g., `services.AddScoped<IBookRepository, SqlBookRepository>();`).
* **Lifetimes (Default):**
    * **Singleton:** For services that are stateless and shared globally (e.g., `ILogger`).
    * **Scoped:** For services that should live for the duration of a single operation (e.g., one HTTP request). This is the default for most services.
    * **Transient:** For lightweight, stateless services that are created and destroyed frequently.

## 3. Configuration and Logging

* **Configuration:** All settings (connection strings, API keys) must be read via the `IConfiguration` interface (injected via DI). No hard-coded strings.
* **Logging:** All logging must be done via the `ILogger<T>` interface (injected via DI). Do not use `Console.WriteLine()` for logging in services.

## 4. Application-Level Error Handling

* **Throwing:**
    1.  Let built-in .NET exceptions bubble up naturally.
    2.  If code would "fail silently" but is logically wrong (like the "Path Validation" rule in the core playbook), explicitly `throw` a new exception.
    3.  When throwing, *first* try to use a suitable, common, built-in .NET exception (e.g., `InvalidOperationException`, `ArgumentNullException`).
    4.  Only create a new custom exception class if no .NET exception accurately describes the error.
* **Catching (The Boundary):** Exceptions must only be caught at the *boundary* of an application, when it is time to notify the user or evaluate a completed operation.
    * **Avalonia:** The "boundary" is the **ViewModel**.
    * **Console:** The "boundary" is a top-level `try/catch` in `Program.cs`.
    * **Web API:** The "boundary" is the **Middleware**.
* **Web API Middleware:**
    * This is the superior implementation of the catching rule for Web APIs.
    * A global `ExceptionHandlingMiddleware` must be implemented.
    * This middleware will be the single `try/catch` for the application, log the exception via `ILogger`, and map the exception to a standard JSON error response and HTTP status code.

## 5. Data Access (Repository Pattern)

* **Pattern:** All data access must be abstracted using the **Repository Pattern**. This applies to databases, local files, or any external data source.
* **Interface:** The repository interface (e.g., `IBookRepository`) must be defined in the `Domain` layer.
* **Implementation:** The concrete implementation (e.g., `SqlBookRepository`) must be in the `Infrastructure` layer.
* **Metaphor:** The `IRepository` interface is the **counter**.
    * The "customer" (your service layer) can only ask the "bartender" (the interface) for what's on the menu (e.g., `GetByIdAsync`, `AddAsync`).
    * The "customer" cannot see or go *behind* the counter.
    * The implementation is the **back room**, which is completely invisible to the customer and contains all the data-access logic (e.g., `_dbContext.Books.Add`).
* **Unit of Work:**
    * For database operations, the `IUnitOfWork` interface shall be used to manage transactions (e.g., `SaveChangesAsync()`).
    * It is injected into services that need to *write* data, allowing multiple repository actions to be saved as a single atomic transaction.

## 6. Database Schema Management

* **Framework:** The standard for data access is **Entity Framework Core (EF Core)**.
* **Method:** All database schema changes must be managed using EF Core's **Code-First Migrations** feature.
* **Process:** Any change to a domain model that is persisted to the database must be accompanied by a new, named migration file.

## 7. Application-Level Validation

* **Web API (DTOs):**
    * **Library:** We will use **FluentValidation** for all complex validation of incoming DTOs.
    * **Execution:** Validation rules will be defined for DTOs and triggered automatically by the ASP.NET Core pipeline *before* the controller action is executed.
* **Avalonia UI (ViewModels):**
    * **Responsibility:** ViewModels are responsible for validating user input.
    * **Interface:** ViewModels must implement the `INotifyDataErrorInfo` interface to provide real-time validation feedback to the View (e.g., "This field is required" in red).

## 8. Authentication & Authorization

* **Standard (External):** For standard external providers (e.g., Google, Microsoft), we will use **OAuth** support built into ASP.NET Core.
* **Custom (Internal):** For custom email/password, email-link login, and account confirmation, we will use **ASP.NET Core Identity**.
* **Cross-Application (Enterprise):** When a single identity is required across multiple applications within an organization, we will use **Microsoft Entra ID**.

## 9. UI Architecture (Avalonia)

* **Pattern:** The **MVVM (Model-View-ViewModel)** pattern must be strictly followed to ensure separation of concerns.
* **Model:** The pure Domain Model from the `Domain` layer (e.g., `Book`).
* **View:** The `.axaml` file. It must contain **zero** C# code-behind logic (e.g., no `Button_Click` event handlers). All interaction is handled through data binding.
* **ViewModel:** The `*ViewModel.cs` class (e.g., `BookViewModel`). It contains all UI logic and state. It must *never* directly reference a View.

## 10. UI Communication (Avalonia)

* **View-to-ViewModel (Commands):**
    * This is **not** CQRS and is **mandatory** for MVVM.
    * All View actions (e.g., `Button.Click`) must be handled by binding to an `ICommand` property on the ViewModel.
    * We will use the `RelayCommand` from the `CommunityToolkit.Mvvm` library to implement these commands.
* **ViewModel-to-ViewModel (Messaging):**
    * ViewModels must **not** have direct references to each other.
    * Communication between ViewModels (e.g., selecting an item in a list to show it in a detail view) must be done using a decoupled **Event Aggregator / Message Bus** (e.g., `IMessenger` from `CommunityToolkit.Mvvm`).

## 11. Web API Architecture

* **Role:** API Controllers must be **"thin."** Their only job is to handle the HTTP request, call a single service, and return an HTTP response.
* **Logic:** Controllers must **never** contain business logic. All logic lives in the service layer.
* **Data Contract:** Controllers must **always** accept and return **DTOs** (per the Core Playbook). They must **never** expose internal Domain Models to the public.

## 12. Caching Strategy

* **In-Memory:** Use `IMemoryCache` (injected) for simple, non-critical, server-specific data.
* **Distributed:** Use `IDistributedCache` (injected) for data that must be shared and consistent across multiple server instances. **Redis** is the default implementation for this.
* **Principle:** Caching is an `Infrastructure` concern. Cached data is not business logic and should not be stored in the main application database.

## 13. Note: CQRS vs. MVVM ICommand

* **CQRS (Command Query Responsibility Segregation):** This is the "enterprise" pattern of creating formal `Command` objects (e.g., `CreateBookCommand`) that are processed by `CommandHandler` classes. We have explicitly decided **not** to use this, as it is overkill for our project scale.
* **MVVM `ICommand`:** This is a *completely different* pattern for UI. It is a simple property on a ViewModel (e.g., `public ICommand BuyBookCommand { get; }`) that a XAML `Button` can bind to. This is **mandatory** for our MVVM architecture. They are not related, despite the similar name.
