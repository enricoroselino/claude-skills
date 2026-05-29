---
name: csharp-modular-monolith
description: Enforce modular monolith architecture in C# (.NET 10) — strict module boundaries, per-module DI, Minimal APIs, Dapper, FluentValidation, MailKit, and the Result pattern.
version: 1.0.0
author: User
---

# C# Modular Monolith Architecture

## Purpose

Enforce modular monolith architecture in C# (.NET 10) projects. Every piece of code lives inside a module. Every module owns its domain. Cross-module communication follows strict rules. The goal: monolith deployment with microservice discipline — replaceable modules, no spaghetti.

---

# Core Principle

> A module is a self-contained business capability. It owns its data, its logic, and its public contract. No other module may reach into its internals.

Every module must be extractable into its own service **without rewriting business logic**. If you can't extract it cleanly, the boundary is broken.

---

# C# Naming Conventions

Follow the [.NET Runtime Design Guidelines](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/) and the [C# Coding Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions).

## Visibility

C# uses explicit access modifiers. No capitalization-trick.

- **`public`**: accessible everywhere.
- **`internal`**: accessible within the same assembly (project).
- **`private`**: accessible within the same type.
- **`protected`**: accessible within the type and its subclasses.

## Namespace and Assembly Naming

- **PascalCase** — `MyCompany.OrderManagement`, `MyCompany.UserService`.
- Assembly name matches the namespace root: `OrderManagement.dll` → `namespace OrderManagement`.
- **No** `Util`, `Common`, `Misc`, `Helpers`, `Base`, `Core` — meaningless name magnets.
- **`SharedKernel`** is the cross-cutting namespace at the root — Result, ApiResponse, DI extensions.

## Files

- One type per file (exception: small related types — request + response).
- File name matches the primary type name: `OrderService.cs`, `PlaceOrderRequest.cs`.
- Test files: `*Tests.cs` in a test project.

## Classes, Structs, Records

- **PascalCase** for types: `OrderService`, `PlaceOrderCommand`, `UserEntity`.
- **`I` prefix** for interfaces: `IOrderRepository`, `IEventHandler`.
- **No** `C` prefix, no Hungarian notation.
- **Abstract base classes**: `Base` prefix: `BaseEntity`, `BaseHandler`.
- **Records** for DTOs and value objects: `public record OrderDto(string Id, string Status)`.

## Methods

- **PascalCase**: `PlaceOrderAsync`, `GetCustomerById`.
- Async methods: **`Async` suffix**: `GetOrderAsync`, `InsertOrderAsync`.
- Boolean-returning methods: `Is`, `Has`, `Can` prefix — `IsActive()`, `HasPermission()`, `CanEdit()`.
- Factory methods: no special prefix; just name what they create.

## Properties

- **PascalCase**: `UserId`, `OrderTotal`, `CreatedAt`.
- Boolean properties: `Is`, `Has` prefix — `IsActive`, `HasErrors`, `IsDeleted`.
- Private backing fields: **`_camelCase`**: `private readonly IOrderRepository _orderRepo;`.

## Constants

- **PascalCase**: `public const int MaxRetries = 3;`.
- Group related constants in a static class: `public static class OrderConstants`.

## Enums

- **PascalCase** for type and members: `enum OrderStatus { Pending, Confirmed, Shipped }`.
- No `Enum` suffix on the type name.
- Use singular: `OrderStatus` not `OrderStatuses`.

## Variables and Parameters

- **camelCase**: `orderId`, `maxRetries`, `httpClient`.
- Short scope = short name: loop variable `i`, lambda parameter `x`.
- No abbreviations except universally known (Id, Url).

## DI Registration Methods

- Extension method on `IServiceCollection` per module: `AddOrderModule()`, `AddUserModule()`.

---

# Module Definition

## What Makes a Module

A module is a **folder** inside a single web project. One solution, one `.csproj`, one deployment unit. Modules are folder boundaries, not assembly boundaries. No separate class libraries.

```
src/                            # Single ASP.NET Core project
  Program.cs
  appsettings.json

  SharedKernel/                 # Cross-cutting — Result, ApiResponse, DI extensions
  Modules/
    Order/                      # Business capability — folder, not project
      Contracts/
      Domain/
      Application/
      Infrastructure/
      Api/
      Module.cs
    Payment/                    # Business capability
    User/                       # Business capability
```

**Module = business capability.** Not a layer. Not a utility drawer. Not a separate `.csproj`.

```
src/
  Controllers/                # ❌ technical layer, not a module
  Services/                   # ❌ technical layer, not a module
  Repositories/               # ❌ technical layer, not a module
```

## Module Name Rules

- **Singular** noun describing the business capability: `Order`, `User`, `Payment`, `Notification`.
- Namespace: `MyApp.Modules.Order`.
- Folder name (if single-project): `src/modules/Order/`.

---

# Module Internal Structure

## Ports & Adapters (Hexagonal Architecture)

Every module follows ports & adapters. Contracts are interfaces, adapters are implementations. Strict dependency direction.

```
modules/Order/
  Contracts/                  # Interfaces — the module's contracts
    IOrderService.cs           # Inbound port — use cases
    IOrderRepository.cs        # Outbound port — data access
    Key.cs                     # DI key constants (optional, for named registrations)
  Domain/                     # Pure domain — zero dependencies
    OrderEntity.cs             # DB-mapped entity (Dapper — no attributes needed)
    OrderDto.cs                # Transfer object — no DB concerns
    OrderErrors.cs             # Domain error types
    OrderEvents.cs             # OrderPlaced, OrderCancelled
    Constants.cs               # Status enums, outbox type keys
  Application/                # Business logic
    OrderService.cs            # Implements IOrderService
  Infrastructure/             # Adapters
    OrderRepository.cs         # Dapper implementation of IOrderRepository
    MailHandler.cs             # MailKit implementation
    OrderPlacedHandler.cs     # Outbox handler
  Api/                         # Primary adapter
    OrderEndpoints.cs          # Minimal API endpoints
    PlaceOrderRequest.cs       # Request DTO + FluentValidation
    OrderResponse.cs           # Response DTO
  Module.cs                    # DI — AddOrderModule(this IServiceCollection)
```

## Dependency Direction

```
Api/ → Contracts/ → Domain/
                       ↑
Infrastructure/ ───────┘
```

- `Api/` depends on `Contracts/` and `Domain/` (calls service interface, uses DTOs).
- `Application/` depends on `Contracts/` and `Domain/` (implements service).
- `Infrastructure/` depends on `Domain/` (uses entities) and `Contracts/` (implements repos).
- `Domain/` depends on **nothing** — pure C#, no NuGet except shared kernel types.
- `Module.cs` depends on everything inside the module — it wires them.

## Type Separation

| File | Contains | Visibility |
|------|---------|-----------|
| `Domain/OrderEntity.cs` | DB-mapped class, no attributes | `internal` |
| `Domain/OrderDto.cs` | Transfer object, no tags | `internal` to Application; `public` if Contracts returns it |
| `Domain/Constants.cs` | Status enums, outbox type keys | `public` |
| `Domain/OrderErrors.cs` | Domain error types | `public` |
| `Domain/OrderEvents.cs` | Event record types | `public` |
| `Contracts/IOrderService.cs` | Service interface | `public` |
| `Contracts/IOrderRepository.cs` | Repository interface | `internal` (module-only) |
| `Contracts/Key.cs` | DI key constants for named services | `public` |
| `Api/PlaceOrderRequest.cs` | Request DTO, FluentValidation validator | `public` |
| `Api/OrderResponse.cs` | Response DTO | `public` |
| `Api/OrderEndpoints.cs` | Minimal API endpoint handlers | `public static` |
| `Infrastructure/OrderRepository.cs` | Dapper queries | `internal` |
| `Module.cs` | DI wiring | `public static` |

---

# The Result Pattern

## Principle

Go returns `(value, error)` tuples. C# can't do multi-return. Instead, a simple `Result<T>` encodes success or failure in one type without exceptions for expected errors.

**No union types.** No OneOf. No third-party result libraries. A single generic + non-generic pair of lightweight structs.

## Shared Result Types — `SharedKernel/Result.cs`

```csharp
// SharedKernel/Result.cs
namespace SharedKernel;

/// <summary>
/// Represents the outcome of an operation: either success with a value, or failure with an error.
/// Domain errors are expected and safe to expose. Exception-based failures are caught by middleware.
/// </summary>
public readonly record struct Result<T>
{
    public T? Value { get; }      // null on failure
    public Error? Error { get; }  // null on success
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;

    private Result(T value)
    {
        Value = value;
        Error = null;
        IsSuccess = true;
    }

    private Result(Error error)
    {
        Value = default;
        Error = error;
        IsSuccess = false;
    }

    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<Error, TResult> onFailure)
        => IsSuccess ? onSuccess(Value!) : onFailure(Error!);

    public static implicit operator Result<T>(T value) => new(value);
    public static implicit operator Result<T>(Error error) => new(error);
}

public readonly record struct Error
{
    public string Code { get; }
    public string Message { get; }

    public Error(string code, string message)
    {
        Code = code;
        Message = message;
    }

    public override string ToString() => $"[{Code}] {Message}";
}
```

## Usage in Services

```csharp
// Order.Application/OrderService.cs
public async Task<Result<OrderDto>> PlaceOrderAsync(PlaceOrderInput input, CancellationToken ct)
{
    var entity = input.ToEntity();

    await _uow.ExecuteAsync(async tx =>
    {
        await _repo.InsertOrderAsync(tx, entity);
        var msg = OutboxMessage.Create(Constants.OutboxTypeOrderPlaced, new OrderPlacedPayload(entity));
        await _outboxRepo.InsertAsync(tx, msg);
    });

    return entity.ToDto();
}
```

Service returns `Result<OrderDto>`. The handler (Minimal API endpoint) calls `Match` to decide the HTTP response.

## Error Creation

```csharp
// Domain errors — safe to expose
public static class OrderErrors
{
    public static Error NotFound(string orderId) =>
        new("order.not_found", $"Order {orderId} not found");

    public static Error AlreadyCancelled =>
        new("order.already_cancelled", "Order is already cancelled");

    public static Error InsufficientStock =>
        new("order.insufficient_stock", "Not enough stock to fulfill the order");
}
```

## Why Not Exceptions for Expected Errors

Exceptions are for truly exceptional cases — DB connection failures, out-of-memory, external service timeouts. Domain errors (not found, validation, duplicate, business rule violation) are expected paths. Result removes try/catch from the happy path and makes the error contract explicit.

---

# UUID Identifiers

## Principle

Every entity ID is a `Guid`. No auto-increment integers.

```csharp
// Domain/OrderEntity.cs
namespace Modules.Order.Domain;

internal sealed class OrderEntity
{
    public Guid Id { get; set; }             // Guid — assigned at creation, never auto-generated by DB
    public Guid UserId { get; set; }
    public string Status { get; set; } = "pending";
    public decimal Total { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

## Guid Creation

- `Guid.CreateVersion7()` (.NET 9+) or `Guid.NewGuid()` (V4) for sensitive IDs.
- Always create the Guid in application code, never let the DB generate it.

## DB Storage

```sql
-- SQL Server
CREATE TABLE orders (
    id UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
    user_id UNIQUEIDENTIFIER NOT NULL,
    status NVARCHAR(50) NOT NULL,
    total DECIMAL(18,2) NOT NULL,
    created_at DATETIME2 NOT NULL,
    updated_at DATETIME2 NOT NULL
);
```

No `NEWSEQUENTIALID()` as default — the application assigns the ID.

Dapper column mapping — add this once in `Program.cs`:

```csharp
Dapper.DefaultTypeMap.MatchNamesWithUnderscores = true;
```

Now `user_id` column auto-maps to `UserId` property. No `[Column]` attributes needed.

---

# Database — Timestamps

## Principle

Every entity table has `CreatedAt` and `UpdatedAt`. No exceptions. Always use UTC — local time is a presentation concern.

Infrastructure tables (`outbox`, `system_alerts`) are append-only — `CreatedAt` only.

## IClock — Abstract the System Clock

`DateTime.UtcNow` in domain code makes tests non-deterministic and prevents timezone swaps. Inject `IClock` everywhere.

```csharp
// SharedKernel/IClock.cs
namespace SharedKernel;

public interface IClock
{
    DateTime UtcNow { get; }
}
```

```csharp
// SharedKernel/SystemClock.cs
namespace SharedKernel;

internal sealed class SystemClock : IClock
{
    public DateTime UtcNow => DateTime.UtcNow;
}
```

```csharp
// SharedKernel/LocalClock.cs — swap in for local-time environments
namespace SharedKernel;

internal sealed class LocalClock : IClock
{
    public DateTime UtcNow => DateTime.Now; // local, not UTC
}
```

Registration — one line, swap implementation per environment:

```csharp
// Program.cs
builder.Services.AddSingleton<IClock>(new SystemClock());  // production — UTC
// builder.Services.AddSingleton<IClock>(new LocalClock()); // dev/local — machine time
```

In tests — deterministic time:

```csharp
// SharedKernel/FakeClock.cs (test project)
public sealed class FakeClock : IClock
{
    public DateTime UtcNow { get; set; } = new DateTime(2026, 1, 1, 0, 0, 0, DateTimeKind.Utc);

    public void Advance(TimeSpan delta) => UtcNow = UtcNow.Add(delta);
}

var clock = new FakeClock();
var entity = new OrderEntity { Id = Guid.NewGuid(), Clock = clock };
entity.MarkUpdated();
entity.UpdatedAt.Should().Be(clock.UtcNow);  // deterministic
```

## BaseEntity

```csharp
// SharedKernel/BaseEntity.cs
namespace SharedKernel;

public abstract class BaseEntity
{
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }

    // Caller passes clock — no hidden DateTime.UtcNow dependency
    public void MarkUpdated(IClock clock) => UpdatedAt = clock.UtcNow;
}
```

All entities inherit from `BaseEntity`. Set `CreatedAt` at construction via clock, call `MarkUpdated(clock)` on every mutation.

**Never call `DateTime.UtcNow` directly in domain or service code.** Always go through `IClock`.

## Dapper — Read Timestamps Back

```csharp
// Infrastructure/OrderRepository.cs
public async Task<OrderEntity?> GetByIdAsync(SqlConnection conn, Guid id, SqlTransaction? tx = null)
{
    return await conn.QuerySingleOrDefaultAsync<OrderEntity>(
        "SELECT id, user_id, status, total, created_at, updated_at FROM orders WHERE id = @Id",
        new { Id = id },
        transaction: tx);
}
```

Timestamps are columns like any other — read them back. UpdatedAt naturally from DB via `GETUTCDATE()`:

```sql
UPDATE orders SET status = @Status, updated_at = GETUTCDATE() WHERE id = @Id
```

---

# Minimal API — Endpoints

## Principle

Minimal APIs in .NET 10 replace controllers. No `[ApiController]`, no `[Route]` attributes. Endpoints are static methods registered via extension methods on `RouteGroupBuilder`.

Each module exposes an `AddOrderEndpoints()` extension that registers its routes. The host project calls one method per module.

## Route Registration — Inside the Module

```csharp
// Order/Api/OrderEndpoints.cs
namespace Modules.Order.Api;

public static class OrderEndpoints
{
    public static RouteGroupBuilder MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/v1/orders")
            .WithTags("Orders")
            .RequireAuthorization();  // JWT auth on all order endpoints

        group.MapPost("/", PlaceOrderAsync);
        group.MapGet("/{id:guid}", GetOrderAsync);
        group.MapPost("/{id:guid}/cancel", CancelOrderAsync);
        group.MapGet("/", ListOrdersAsync);

        return group;
    }

    private static async Task<IResult> PlaceOrderAsync(
        PlaceOrderRequest request,
        IOrderService orderService,
        CancellationToken ct)
    {
        var result = await orderService.PlaceOrderAsync(request.ToInput(), ct);
        return result.Match(
            onSuccess: dto => Results.Ok(ApiResponse.Success(dto)),
            onFailure: error => error.ToResult()
        );
    }

    private static async Task<IResult> GetOrderAsync(
        Guid id,
        IOrderService orderService,
        CancellationToken ct)
    {
        var result = await orderService.GetOrderAsync(id, ct);
        return result.Match(
            onSuccess: dto => Results.Ok(ApiResponse.Success(dto)),
            onFailure: error => error.ToResult()
        );
    }
}
```

## URL Convention

```
/v1/orders              # NOT /api/v1/orders
/v1/orders/{id}         # GET specific order
/v1/auth/login          # NOT /api/v1/auth/login
```

`/api` prefix is noise. Versioning via `/v1/` already distinguishes API routes.

## Wiring in Program.cs

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Infrastructure
builder.Services.AddDatabase(builder.Configuration);
builder.Services.AddOutbox();

// Modules — one call per module
builder.Services.AddOrderModule(builder.Configuration);
builder.Services.AddPaymentModule(builder.Configuration);
builder.Services.AddNotificationModule(builder.Configuration);

var app = builder.Build();

app.UseMiddleware<RequestIdMiddleware>();
app.UseMiddleware<RecovererMiddleware>();
app.UseMiddleware<RealIpMiddleware>();
app.UseRateLimiter();
app.UseAuthentication();
app.UseAuthorization();

// Map endpoints — one call per module
app.MapOrderEndpoints();
app.MapPaymentEndpoints();
app.MapNotificationEndpoints();

app.Run();
```

One `.AddXxxModule()` per module (DI) + one `.MapXxxEndpoints()` per module (routing). `Program.cs` is a flat list — no knowledge of module internals.

---

# FluentValidation — Request Validation

## Principle

Every request type has a corresponding `AbstractValidator<T>`. Endpoints call it explicitly (or via filter). Validation lives in `Api/` alongside the request type — never in the service.

```csharp
// Order/Api/PlaceOrderRequest.cs
namespace Modules.Order.Api;

// Install-Package FluentValidation
public sealed class PlaceOrderRequest
{
    public Guid UserId { get; init; }
    public List<OrderItemRequest> Items { get; init; } = [];

    public PlaceOrderInput ToInput() => new(UserId, Items.ConvertAll(i => i.ToDto()));
}

public sealed class OrderItemRequest
{
    public Guid ProductId { get; init; }
    public int Quantity { get; init; }
}

public sealed class PlaceOrderRequestValidator : AbstractValidator<PlaceOrderRequest>
{
    public PlaceOrderRequestValidator()
    {
        RuleFor(x => x.UserId).NotEmpty();
        RuleFor(x => x.Items).NotEmpty().WithMessage("At least one item is required");
        RuleForEach(x => x.Items).SetValidator(new OrderItemRequestValidator());
    }
}

public sealed class OrderItemRequestValidator : AbstractValidator<OrderItemRequest>
{
    public OrderItemRequestValidator()
    {
        RuleFor(x => x.ProductId).NotEmpty();
        RuleFor(x => x.Quantity).GreaterThan(0).LessThanOrEqualTo(100);
    }
}
```

## Validation Filter — DRY Endpoint Validation

A single endpoint filter applies FluentValidation automatically:

```csharp
// SharedKernel/ValidationFilter.cs
namespace SharedKernel;

public static class ValidationFilter
{
    public static RouteHandlerBuilder Validate<T>(this RouteHandlerBuilder builder)
        where T : class
    {
        builder.AddEndpointFilter(async (context, next) =>
        {
            var request = context.Arguments.OfType<T>().FirstOrDefault();
            if (request is null)
                return Results.BadRequest(ApiResponse.BadRequest("Invalid request body"));

            var validator = context.HttpContext.RequestServices.GetRequiredService<IValidator<T>>();
            var result = await validator.ValidateAsync(request);

            if (result.IsValid)
                return await next(context);

            var errors = result.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(g => g.Key, g => g.First().ErrorMessage);

            return Results.UnprocessableEntity(ApiResponse.ValidationError(errors));
        });

        return builder;
    }
}
```

Usage in endpoint registration:

```csharp
group.MapPost("/", PlaceOrderAsync)
    .Validate<PlaceOrderRequest>();
```

## FluentValidation Registration in Module.cs

```csharp
// Order/Module.cs
public static IServiceCollection AddOrderModule(this IServiceCollection services, IConfiguration config)
{
    // Validators
    services.AddValidatorsFromAssemblyContaining<PlaceOrderRequestValidator>(ServiceLifetime.Singleton);

    // ... rest of DI
}
```

---

# Structured HTTP Responses

## ApiResponse Envelope

Every HTTP response uses a single envelope type. No free-form JSON.

```csharp
// SharedKernel/ApiResponse.cs
namespace SharedKernel;

public sealed class ApiResponse
{
    public bool Success { get; init; }
    public int Code { get; init; }
    public string Message { get; init; } = string.Empty;
    public object? Data { get; init; }
    public PaginationMetadata? Pagination { get; init; }
    public object? Errors { get; init; }
    public string? TraceId { get; init; }

    // 2xx
    public static ApiResponse Success(object? data = null, string message = "Success")
        => new() { Success = true, Code = 200, Message = message, Data = data };

    public static ApiResponse Created(object? data = null, string message = "Created")
        => new() { Success = true, Code = 201, Message = message, Data = data };

    public static ApiResponse Paginated(object data, PaginationMetadata meta)
        => new() { Success = true, Code = 200, Message = "Success", Data = data, Pagination = meta };

    // 4xx
    public static ApiResponse BadRequest(string message = "Bad request")
        => new() { Success = false, Code = 400, Message = message };

    public static ApiResponse ValidationError(object errors)
        => new() { Success = false, Code = 422, Message = "Validation failed", Errors = errors };

    public static ApiResponse Unauthorized(string message = "Unauthorized")
        => new() { Success = false, Code = 401, Message = message };

    public static ApiResponse Forbidden(string message = "Forbidden")
        => new() { Success = false, Code = 403, Message = message };

    public static ApiResponse NotFound(string message = "Not found")
        => new() { Success = false, Code = 404, Message = message };

    // 5xx
    public static ApiResponse InternalError()
        => new() { Success = false, Code = 500, Message = "Internal Server Error" };
}
```

| Field | When Present | Description |
|-------|-------------|-------------|
| `success` | Always | `true` for 2xx, `false` otherwise |
| `code` | Always | Mirrors HTTP status code |
| `message` | Always | Human-readable summary. **5xx: hardcoded generic. Never expose internals.** |
| `data` | 2xx with body | Payload — object or array |
| `pagination` | 2xx paginated | Pagination metadata |
| `errors` | 4xx validation | Field-level errors as key-value dict |
| `traceId` | Non-2xx | Correlation ID for debugging |

## Pagination

```csharp
// SharedKernel/Pagination.cs
namespace SharedKernel;

public sealed record PaginationParams(int Page = 1, int PageSize = 20)
{
    public int Page { get; } = Math.Max(1, Page);
    public int PageSize { get; } = Math.Clamp(PageSize, 1, 100);
    public int Offset => (Page - 1) * PageSize;
}

public sealed record PaginationMetadata(int Page, int PageSize, int TotalItems)
{
    public int TotalPages => (int)Math.Ceiling((double)TotalItems / PageSize);
}
```

Repository receives `(int pageSize, int offset)` — raw SQL primitives. Service calls `params.Offset()`. Handler converts to `PaginationMetadata`.

```csharp
// Dapper repository
public async Task<(List<OrderEntity> items, int total)> SearchAsync(
    SqlConnection conn, Guid userId, int pageSize, int offset)
{
    var sql = """
        SELECT Id, UserId, Status, Total, CreatedAt, UpdatedAt
        FROM orders WHERE UserId = @UserId
        ORDER BY CreatedAt DESC
        OFFSET @Offset ROWS FETCH NEXT @PageSize ROWS ONLY;

        SELECT COUNT(*) FROM orders WHERE UserId = @UserId;
        """;

    using var multi = await conn.QueryMultipleAsync(sql, new { UserId = userId, PageSize = pageSize, Offset = offset });
    var items = (await multi.ReadAsync<OrderEntity>()).ToList();
    var total = await multi.ReadSingleAsync<int>();
    return (items, total);
}
```

## Mapping Errors to HTTP Responses

```csharp
// SharedKernel/ErrorExtensions.cs
namespace SharedKernel;

public static class ErrorExtensions
{
    public static IResult ToResult(this Error error) => error.Code switch
    {
        "order.not_found" or "user.not_found" => Results.NotFound(ApiResponse.NotFound(error.Message)),
        "order.already_cancelled" or "conflict" => Results.Conflict(ApiResponse.BadRequest(error.Message)),
        _ => Results.BadRequest(ApiResponse.BadRequest(error.Message))
    };

    public static IResult ToValidationResult(this IDictionary<string, string> errors)
        => Results.UnprocessableEntity(ApiResponse.ValidationError(errors));
}
```

## Empty List → `[]`, Never `null`

```csharp
// Wrong
h.Success(null);               // → {"data": null}

// Correct
var orders = await _service.ListAsync(ct);
h.Success(orders ?? []);       // → {"data": []}
```

Every list field defaults to an empty collection, never null. Pagination is the only valid null.

---

# Module Errors

## Error Design

Errors come from `Domain/` as `Error` values. Services return `Result<T>` — success with value, failure with `Error`. The handler maps the `Error.Code` to an HTTP status code.

```csharp
// Domain/OrderErrors.cs
namespace Modules.Order.Domain;

public static class OrderErrors
{
    public static Error NotFound(Guid orderId) =>
        new("order.not_found", $"Order {orderId} not found.");

    public static Error AlreadyCancelled(Guid orderId) =>
        new("order.already_cancelled", $"Order {orderId} is already cancelled.");

    public static Error InsufficientStock(string productId, int requested, int available) =>
        new("order.insufficient_stock",
            $"Product {productId}: requested {requested} but only {available} available.");
}
```

```csharp
// Application/OrderService.cs
public async Task<Result<OrderDto>> CancelOrderAsync(Guid orderId, Guid userId, CancellationToken ct)
{
    var order = await _repo.GetByIdAsync(conn, orderId);
    if (order is null) return OrderErrors.NotFound(orderId);
    if (order.Status == "cancelled") return OrderErrors.AlreadyCancelled(orderId);

    order.Status = "cancelled";
    order.MarkUpdated();
    await _repo.UpdateAsync(conn, order);

    return order.ToDto();
}
```

## Rules

| Rule | Why |
|------|-----|
| Domain errors as `Error` structs | Lightweight. No exception overhead for expected paths. |
| Error codes follow `domain.reason` convention | Greppable. `order.not_found` tells you the module + what happened. |
| Result<T> forces the caller to handle failure | No implicit `null` returns. The compiler won't let you ignore the error case. |
| Service never throws for domain errors | Return errors via Result. Let middleware catch only unexpected exceptions. |
| `Error.Code` drives the HTTP status | The handler has a switch/map — no if-chain. |

---

# Dependency Rules

## The One Rule

**Modules may only depend on:**
1. `SharedKernel` (shared types, Result, ApiResponse, Pagination)
2. Other modules' `Contracts/` (public interfaces, public DTOs)
3. Standard library + NuGet packages

**Modules may NOT:**
1. Import `Infrastructure/` or `Application/` from another module
2. Import from a sibling module at all unless it's declared as an upstream dependency
3. Create circular dependencies

## Dependency Direction

```
┌─────────────────────────────────────┐
│           SharedKernel               │
│  (Result<T>, Error, ApiResponse,     │
│   BaseEntity, Pagination)            │
└─────────────────────────────────────┘
         ↑             ↑
    ┌────┘         └────┐
┌───┴───┐         ┌────┴──────┐
│ Order │ ←──────→│ Payment   │  (via Contracts/ only)
└───────┘         └───────────┘
```

## Dependency Declaration

Each `Module.cs` starts with a comment listing its dependencies:

```csharp
// Order/Module.cs
// Dependencies: SharedKernel, User.Contracts, Payment.Contracts
public static class OrderModule
{
    public static IServiceCollection AddOrderModule(this IServiceCollection services, IConfiguration config)
    {
        // ...
    }
}
```

---

# Shared Kernel (`SharedKernel/`)

## What Belongs in Shared

- `Result<T>` and `Error` — the universal return type
- `ApiResponse` + `PaginationMetadata` — HTTP envelope
- `BaseEntity` — CreatedAt/UpdatedAt contract
- `IClock` + `SystemClock` — abstracted system clock (UTC by default)
- `IUnitOfWork` — transaction management interface
- `IEventBus` — in-process pub/sub
- `ILogger<T>` (.NET built-in — Serilog implements it)

## What Does NOT Belong in Shared

- Any business entity (`User`, `Order`)
- Any module-specific DTO or enum
- Any repository or service interface
- Any validation rule

---

# Logging — Serilog via ILogger

## Principle

.NET has `Microsoft.Extensions.Logging.ILogger<T>`. Use it directly — no custom wrapper. **Serilog** plugs in as the provider behind `ILogger<T>`. All code logs to the interface; Serilog handles sinks, enrichment, and formatting. Swap Serilog for OpenTelemetry or console without touching a single service.

## Serilog Setup — appsettings.json

Serilog reads from config; no code-based `LoggerConfiguration`. See `# Configuration — appsettings.json` for the full file.

## Serilog Setup — Program.cs (Minimal)

```csharp
// Install-Package Serilog
// Install-Package Serilog.Extensions.Logging
// Install-Package Serilog.Settings.Configuration  // reads appsettings.json
// Install-Package Serilog.Sinks.Console
// Install-Package Serilog.Sinks.File
// Install-Package Serilog.Enrichers.Environment
// Install-Package Serilog.Enrichers.Thread
// Install-Package Serilog.Enrichers.Process
// Install-Package Serilog.Exceptions

using Serilog;

var builder = WebApplication.CreateBuilder(args);

// Read Serilog config from appsettings.json — no code-based LoggerConfiguration
builder.Host.UseSerilog((context, config) =>
    config.ReadFrom.Configuration(context.Configuration));

try
{
    // ... DI, modules ...

    var app = builder.Build();
    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
}
finally
{
    Log.CloseAndFlush();
}
```

Sink configuration lives in `appsettings.json`. `Program.cs` is one line: `ReadFrom.Configuration()`. Swap sinks per environment via `appsettings.Production.json` — zero code change.

## Request Logging Middleware

```csharp
// SharedKernel/RequestLoggingMiddleware.cs
using Serilog.Context;

public sealed class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;

    public RequestLoggingMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        // Push per-request enrichment — every log in this request gets these properties
        using (LogContext.PushProperty("CorrelationId", context.TraceIdentifier))
        using (LogContext.PushProperty("ClientIp", context.Connection.RemoteIpAddress?.ToString()))
        {
            var sw = Stopwatch.StartNew();
            await _next(context);
            sw.Stop();

            Log.Information(
                "{Method} {Path} responded {StatusCode} in {ElapsedMs}ms",
                context.Request.Method, context.Request.Path, context.Response.StatusCode, sw.ElapsedMilliseconds);
        }
    }
}
```

## Service Logging — ILogger<T> (No Serilog Reference)

Services never reference `Serilog` namespace. They log to `ILogger<T>`. Serilog enriches behind the scenes.

```csharp
// Application/OrderService.cs
internal sealed class OrderService : IOrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger) => _logger = logger;

    public async Task<Result<OrderDto>> PlaceOrderAsync(PlaceOrderInput input, CancellationToken ct)
    {
        // Structured log — Serilog captures {UserId} and {ItemCount} as properties
        _logger.LogInformation("Placing order for user {UserId} with {ItemCount} items",
            input.UserId, input.Items.Count);

        // Error log with exception — Serilog writes ExceptionDetails automatically
        try
        {
            // ...
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "[OrderService.PlaceOrder] Failed to insert order {OrderId}", entity.Id);
            throw;
        }
    }
}
```

## Log Message Convention

- **`[Type.Method]` prefix**: `[OrderService.PlaceOrder]` — greppable, stack-free.
- **Structured data as named placeholders**: `_logger.LogInformation("Order {OrderId} placed", order.Id)` — not `$"Order {order.Id} placed"`. Message templates are compile-time analyzable.
- **Never interpolate in the message template**: `$"Order {id}"` hides the property name from Serilog. Use `"Order {OrderId}"` with `order.Id` as argument.
- `{@Object}` serializes as JSON — use for complex payloads: `_logger.LogWarning("Invalid payload {@Payload}", request)`.

## Serilog Sinks — Recommended Set

| Sink | Package | Purpose |
|------|---------|---------|
| Console | `Serilog.Sinks.Console` | Local dev, Docker logs |
| File | `Serilog.Sinks.File` | Persistent, low-cost audit trail |
| Seq | `Serilog.Sinks.Seq` | Structured log viewer — search + dashboard |
| OpenTelemetry | `Serilog.Sinks.OpenTelemetry` | Export to OTLP collector |

Start with Console + File. Add Seq/OTel when needed.

## Serilog Rules

| Rule | Why |
|------|-----|
| `builder.Host.UseSerilog()` not `AddSerilog()` | Replaces `ILoggerFactory` provider. `AddSerilog()` stacks on top — you get double logging. |
| `.Enrich.WithExceptionDetails()` | Stack traces + inner exceptions as structured JSON. No more scanning multi-line text dumps. |
| `LogContext.PushProperty()` per request | CorrelationId, UserId, TenantId — every log in the request inherits them. No manual passing. |
| Override noisy namespaces to `Warning` | `Microsoft.AspNetCore`, `HttpClient` — healthy systems are noisy at Info level. Suppress. |
| `Log.CloseAndFlush()` in `finally` | Drains pending log events before exit. Avoids truncated final lines. |

## Database Log Sink — Capture WARN+ERROR to DB

Console and file sinks cover operational monitoring. But for production, you need WARN+ERROR persisted to the database — searchable, queryable, with context. Serilog has no built-in DB sink that does this cleanly, so write a custom one.

### AlertEntity + Table

```csharp
// SharedKernel/AlertEntity.cs
namespace SharedKernel;

public sealed class AlertEntity
{
    public Guid Id { get; init; }
    public string Level { get; init; } = string.Empty;       // WARN | ERROR
    public string Message { get; init; } = string.Empty;
    public string? Context { get; init; }                     // JSON — key=value pairs
    public string? Trace { get; init; }                       // Stack (ERROR only)
    public string? SourceMethod { get; init; }                // [Type.Method]
    public DateTime CreatedAt { get; init; }
}
```

```sql
CREATE TABLE system_alerts (
    Id UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
    Level NVARCHAR(10) NOT NULL,
    Message NVARCHAR(MAX) NOT NULL,
    Context NVARCHAR(MAX),              -- JSON — queryable with JSON_VALUE
    Trace NVARCHAR(MAX),                -- Stack trace on ERROR only
    SourceMethod NVARCHAR(255),
    CreatedAt DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
CREATE INDEX IX_SystemAlerts_Level ON system_alerts (Level);
CREATE INDEX IX_SystemAlerts_SourceMethod ON system_alerts (SourceMethod);
CREATE INDEX IX_SystemAlerts_CreatedAt ON system_alerts (CreatedAt);
```

### DB Sink — Channel-Based, Non-Blocking

Serilog sinks are called synchronously during a log write. A DB INSERT inside a sink would stall every log call. Instead: sink pushes to a bounded channel, background worker drains it.

```csharp
// SharedKernel/AlertSink.cs
using Serilog.Core;
using Serilog.Events;
using System.Threading.Channels;

namespace SharedKernel;

internal sealed class AlertSink : ILogEventSink, IDisposable
{
    private readonly ChannelWriter<AlertEntity> _writer;

    public AlertSink(ChannelWriter<AlertEntity> writer) => _writer = writer;

    public void Emit(LogEvent logEvent)
    {
        // Only WARN and above hit the DB
        if (logEvent.Level < LogEventLevel.Warning)
            return;

        // Extract source method from message template or properties
        string? sourceMethod = null;
        if (logEvent.Properties.TryGetValue("SourceContext", out var ctx) &&
            ctx is ScalarValue sv)
            sourceMethod = sv.Value?.ToString();

        // Collect all properties as key=value JSON
        var fields = new Dictionary<string, object?>();
        foreach (var prop in logEvent.Properties)
        {
            if (prop.Key is "SourceContext" or "CorrelationId" or "RequestId")
                continue; // already extracted or noise
            fields[prop.Key] = prop.Value.ToString();
        }
        var contextJson = fields.Count > 0
            ? System.Text.Json.JsonSerializer.Serialize(fields)
            : null;

        // Capture stack trace on ERROR
        var trace = logEvent.Level >= LogEventLevel.Error
            ? logEvent.Exception?.ToString()
            : null;

        var alert = new AlertEntity
        {
            Id = Guid.CreateVersion7(),
            Level = logEvent.Level.ToString(),
            Message = logEvent.RenderMessage(),
            Context = contextJson,
            Trace = trace,
            SourceMethod = sourceMethod,
            CreatedAt = DateTime.UtcNow
        };

        // Non-blocking — drop if channel full (logging must never stall)
        _writer.TryWrite(alert);
    }

    public void Dispose() => _writer.TryComplete();
}
```

Key behaviors:
- **Level filter** — only WARN and ERROR reach DB. DEBUG/INFO go to console/file only.
- **Source method** — `SourceContext` from `ILogger<T>` gives `Namespace.ClassName`.
- **Stack trace on ERROR only** — saves column space. WARN gets null trace.
- **`TryWrite`** — non-blocking. Channel full = drop. Logging never stalls business flow.

### Channel + Sink Registration

```csharp
// SharedKernel/DependencyInjection.cs — Logging section
using System.Threading.Channels;

public static IServiceCollection AddSharedKernel(this IServiceCollection services, IConfiguration config)
{
    // Alert channel — bounded, 1000 backlog
    var channel = Channel.CreateBounded<AlertEntity>(new BoundedChannelOptions(1000)
    {
        FullMode = BoundedChannelFullMode.DropWrite
    });
    services.AddSingleton(channel.Reader);
    services.AddSingleton(channel.Writer);

    // Add DB sink to Serilog
    Log.Logger = new LoggerConfiguration()
        .ReadFrom.Configuration(config)
        .WriteTo.Sink(new AlertSink(channel.Writer))
        .CreateLogger();

    services.AddHostedService<AlertWorker>();
    // ...
}
```

### AlertWorker — Persist Channel to DB

```csharp
// SharedKernel/AlertWorker.cs
namespace SharedKernel;

internal sealed class AlertWorker : BackgroundService
{
    private readonly ChannelReader<AlertEntity> _reader;
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<AlertWorker> _logger;

    public AlertWorker(
        ChannelReader<AlertEntity> reader,
        IServiceScopeFactory scopeFactory,
        ILogger<AlertWorker> logger)
    {
        _reader = reader;
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        // Reaper tick — every 12h
        using var reaperTimer = new PeriodicTimer(TimeSpan.FromHours(12));

        _ = Task.Run(async () =>
        {
            while (await reaperTimer.WaitForNextTickAsync(ct))
                await RunReaperAsync();
        }, ct);

        // Drain channel
        await foreach (var alert in _reader.ReadAllAsync(ct))
        {
            await using var scope = _scopeFactory.CreateAsyncScope();
            var conn = scope.ServiceProvider.GetRequiredService<ISqlConnectionFactory>()
                .CreateConnection();
            await conn.ExecuteAsync(
                @"INSERT INTO system_alerts (Id, Level, Message, Context, Trace, SourceMethod, CreatedAt)
                  VALUES (@Id, @Level, @Message, @Context, @Trace, @SourceMethod, @CreatedAt)",
                alert);
        }
    }

    private async Task RunReaperAsync()
    {
        await using var scope = _scopeFactory.CreateAsyncScope();
        var conn = scope.ServiceProvider.GetRequiredService<ISqlConnectionFactory>()
            .CreateConnection();
        var deleted = await conn.ExecuteAsync(
            "DELETE FROM system_alerts WHERE CreatedAt < DATEADD(DAY, -30, SYSUTCDATETIME())");
        if (deleted > 0)
            Log.Information("[AlertWorker] Cleaned {Count} alerts older than 30 days", deleted);
    }
}
```

### Wiring Summary

| Component | Role |
|-----------|------|
| `AlertSink` | Serilog `ILogEventSink` — filter WARN+ERROR → push to channel |
| `Channel<AlertEntity>` | Bounded (1000), `DropWrite` — logging never blocks |
| `AlertWorker` | `BackgroundService` — drain channel, INSERT into DB, reaper every 12h |
| `system_alerts` table | Queryable WARN+ERROR log with context, trace, source method |

Console + file for development. DB alerts for production incident response.

---

# Middleware

## Middleware Order

```csharp
// Program.cs — order matters
var app = builder.Build();

app.UseMiddleware<RequestIdMiddleware>();       // 1. Tag every request
app.UseMiddleware<RecovererMiddleware>();        // 2. Catch panics, log, return 500
app.UseMiddleware<RealIpMiddleware>();           // 3. Restore real client IP
app.UseRateLimiter();                            // 4. Rate limit by real IP
app.UseAuthentication();                         // 5. JWT validation
app.UseAuthorization();                          // 6. Policy enforcement

app.MapOrderEndpoints();
// ...
```

## Custom Exception Handler (Recoverer)

```csharp
// SharedKernel/RecovererMiddleware.cs
namespace SharedKernel;

public sealed class RecovererMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RecovererMiddleware> _logger;

    public RecovererMiddleware(RequestDelegate next, ILogger<RecovererMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex) when (ex is not OperationCanceledException)
        {
            _logger.LogError(ex, "Unhandled exception on {Method} {Path}",
                context.Request.Method, context.Request.Path);

            context.Response.StatusCode = 500;
            context.Response.ContentType = "application/json";
            await context.Response.WriteAsJsonAsync(ApiResponse.InternalError());
        }
    }
}
```

**Rules:**
- Never expose exception message to client — always `"Internal Server Error"`.
- Log full exception with `ILogger` — stack trace is captured automatically.
- Exclude `OperationCanceledException` — it's a cancelled request, not a bug.
- Include `TraceId` in the 500 response — client reports it, operator greps for it.

## Rate Limiter

.NET 10 built-in rate limiting with `FixedWindowLimiter`. Default partition key is client IP — works out of the box.

**But:** behind Cloudflare or any reverse proxy, `HttpContext.Connection.RemoteIpAddress` is always the proxy's IP, not the real client. Rate limiting without fixing this rate-limits the entire user base as a single IP.

### Real IP Middleware

```csharp
// SharedKernel/RealIpMiddleware.cs
namespace SharedKernel;

public sealed class RealIpMiddleware
{
    private readonly RequestDelegate _next;
    private static readonly string[] _trustedHeaders =
        ["CF-Connecting-IP", "X-Forwarded-For", "X-Real-IP"];

    public RealIpMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        foreach (var header in _trustedHeaders)
        {
            if (context.Request.Headers.TryGetValue(header, out var value))
            {
                if (System.Net.IPAddress.TryParse(value.ToString(), out var ip))
                {
                    context.Connection.RemoteIpAddress = ip;
                    break;
                }
            }
        }
        await _next(context);
    }
}
```

The header order matters: Cloudflare strips non-Cloudflare-set headers it forwards, so `CF-Connecting-IP` can't be spoofed by the client. `X-Forwarded-For` is last resort — it could contain the proxy chain, so take the leftmost IP.

### Named Policies

```csharp
// Program.cs
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("global", opt =>
    {
        opt.PermitLimit = 120;
        opt.Window = TimeSpan.FromMinutes(1);
    });
    options.AddFixedWindowLimiter("strict", opt =>
    {
        opt.PermitLimit = 5;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
    });
    options.AddFixedWindowLimiter("standard", opt =>
    {
        opt.PermitLimit = 30;
        opt.Window = TimeSpan.FromMinutes(1);
    });
    options.AddFixedWindowLimiter("relaxed", opt =>
    {
        opt.PermitLimit = 60;
        opt.Window = TimeSpan.FromMinutes(1);
    });
    options.RejectionStatusCode = 429;
});

// Global safety net — catches every unmatched route
app.UseRateLimiter();                              // applied globally

// Module routes — override per-endpoint
group.MapPost("/login", LoginAsync)
    .RequireRateLimiting("strict");
group.MapPost("/register", RegisterAsync)
    .RequireRateLimiting("strict");
group.MapGet("/events", GetEventsAsync)
    .RequireRateLimiting("relaxed");
```

### Rate Limiter Rules

| Rule | Why |
|------|-----|
| Global limiter as safety net | Catches unscoped endpoints, prevents missed configuration. |
| Strict for auth/sensitive endpoints | Login, register, password reset — 5 req/min. |
| `RealIpMiddleware` before `UseRateLimiter` | Rate limiter must see real client IP, not proxy IP. |
| Rejection status 429 | Standard — client can retry after `Retry-After`. |
| Rate limit by IP, not user ID | Works for unauthenticated endpoints. No side-channel from failed auth. |

---

# Configuration

## Single Config Source — appsettings.json

One file, all config. No scattered `builder.Configuration["key"]` calls. Every section binds to a strongly-typed options class via `IOptions<T>`.

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft.AspNetCore": "Warning",
        "System.Net.Http.HttpClient": "Warning"
      }
    },
    "Enrich": [
      "FromLogContext",
      "WithMachineName",
      "WithProcessId",
      "WithThreadId",
      "WithExceptionDetails"
    ],
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/app-.log",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 7,
          "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}"
        }
      }
    ]
  },

  "App": {
    "JwtSecret": "super-secret-key"
  },

  "ConnectionStrings": {
    "Primary": "Server=localhost;Database=mydb;...",
    "ReadReplica": "Server=read-replica;Database=mydb;...",
    "Reporting": "Server=reporting-db;Database=reports;..."
  },

  "Smtp": {
    "Host": "smtp.example.com",
    "Port": 587,
    "User": "noreply@example.com",
    "Pass": "app-password",
    "Sender": "MyApp <noreply@example.com>"
  },

  "Outbox": {
    "PollIntervalSeconds": 5,
    "BatchSize": 20
  }
}
```

```json
// appsettings.Development.json — overrides only what differs
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Debug"
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}"
        }
      }
    ]
  },
  "ConnectionStrings": {
    "Primary": "Server=localhost;Database=mydb_dev;Trusted_Connection=True;"
  }
}
```

## Strongly-Typed Options Classes

```csharp
// SharedKernel/AppConfig.cs
namespace SharedKernel;

public sealed class AppConfig
{
    public const string Section = "App";
    public string JwtSecret { get; init; } = string.Empty;
}
```

```csharp
// SharedKernel/SmtpConfig.cs
namespace SharedKernel;

public sealed class SmtpConfig
{
    public const string Section = "Smtp";
    public string Host { get; init; } = string.Empty;
    public int Port { get; init; } = 587;
    public string User { get; init; } = string.Empty;
    public string Pass { get; init; } = string.Empty;
    public string Sender { get; init; } = string.Empty;
}
```

```csharp
// SharedKernel/Outbox/OutboxOptions.cs
namespace SharedKernel.Outbox;

public sealed class OutboxOptions
{
    public int PollIntervalSeconds { get; init; } = 5;
    public int BatchSize { get; init; } = 20;
}
```

## Registration in Program.cs

```csharp
// Bind each section to its options class
builder.Services.Configure<AppConfig>(builder.Configuration.GetSection(AppConfig.Section));
builder.Services.Configure<SmtpConfig>(builder.Configuration.GetSection(SmtpConfig.Section));
builder.Services.Configure<OutboxOptions>(builder.Configuration.GetSection("Outbox"));
```

Modules access via `IOptions<T>` — never call `Configuration["key"]` directly.

---

# Database Connection Factory

Dapper needs a `SqlConnection` — but services and repositories don't care how it's created. A single factory creates connections from the connection string. Everything injects `SqlConnectionFactory`.

```csharp
// SharedKernel/SqlConnectionFactory.cs
namespace SharedKernel;

public interface ISqlConnectionFactory
{
    SqlConnection CreateConnection();
}

internal sealed class SqlConnectionFactory : ISqlConnectionFactory
{
    private readonly string _connectionString;

    public SqlConnectionFactory(string connectionString) => _connectionString = connectionString;

    public SqlConnection CreateConnection() => new(_connectionString);
}
```

Registration in `Program.cs`:

```csharp
// Single database
builder.Services.AddSingleton<ISqlConnectionFactory>(new SqlConnectionFactory(
    builder.Configuration.GetConnectionString("Default")!));
```

Or as an extension:

```csharp
// SharedKernel/DependencyInjection.cs
public static IServiceCollection AddDatabase(this IServiceCollection services, string connectionString)
{
    services.AddSingleton<ISqlConnectionFactory>(new SqlConnectionFactory(connectionString));
    services.AddScoped<IUnitOfWork, UnitOfWork>();
    return services;
}
```

Usage:

```csharp
// Program.cs
services.AddDatabase(builder.Configuration.GetConnectionString("Default")!);
```

## Multiple Database Connections

When modules need different databases (primary, read replica, legacy reporting), key each factory by name. The interface is identical — only the connection string differs.

```csharp
// SharedKernel/NamedConnectionFactory.cs
namespace SharedKernel;

public static class DatabaseKeys
{
    public const string Primary = "primary";
    public const string ReadReplica = "read_replica";
    public const string Reporting = "reporting";
}

// SharedKernel/DependencyInjection.cs
public static IServiceCollection AddDatabase(this IServiceCollection services, IConfiguration config)
{
    // Keyed registrations — .NET 8+ keyed DI
    services.AddKeyedSingleton<ISqlConnectionFactory>(
        DatabaseKeys.Primary,
        new SqlConnectionFactory(config.GetConnectionString("Primary")!));

    services.AddKeyedSingleton<ISqlConnectionFactory>(
        DatabaseKeys.ReadReplica,
        new SqlConnectionFactory(config.GetConnectionString("ReadReplica")!));

    services.AddKeyedSingleton<ISqlConnectionFactory>(
        DatabaseKeys.Reporting,
        new SqlConnectionFactory(config.GetConnectionString("Reporting")!));

    // Default UoW uses primary
    services.AddScoped<IUnitOfWork, UnitOfWork>();
    return services;
}
```

A module that needs the read replica injects via `[FromKeyedServices]`:

```csharp
// Order.Infrastructure/OrderRepository.cs
internal sealed class OrderReadRepository
{
    private readonly ISqlConnectionFactory _readFactory;

    public OrderReadRepository(
        [FromKeyedServices(DatabaseKeys.ReadReplica)] ISqlConnectionFactory readFactory)
    {
        _readFactory = readFactory;
    }

    public async Task<List<OrderEntity>> SearchAsync(...)
    {
        await using var conn = _readFactory.CreateConnection();
        // SELECT on read replica
    }
}
```

The `UnitOfWork` always uses primary. Read-only queries go directly through the keyed factory, no transaction wrapper needed.

Services and UoW inject `ISqlConnectionFactory`. Repositories receive `SqlConnection` + `SqlTransaction?` — they never call the factory. Connection lifetime is owned by the caller (UoW or worker).

---

# Unit of Work (Transaction Management)

## Principle

Transactions span business operations. Dapper needs explicit connection + transaction management. A shared `IUnitOfWork` wraps this.

```csharp
// SharedKernel/IUnitOfWork.cs
namespace SharedKernel;

public interface IUnitOfWork
{
    Task ExecuteAsync(Func<SqlConnection, SqlTransaction, Task> operation, CancellationToken ct = default);
    Task<T> ExecuteAsync<T>(Func<SqlConnection, SqlTransaction, Task<T>> operation, CancellationToken ct = default);
}
```

## Implementation

```csharp
// SharedKernel/UnitOfWork.cs
namespace SharedKernel;

internal sealed class UnitOfWork : IUnitOfWork
{
    private readonly ISqlConnectionFactory _factory;

    public UnitOfWork(ISqlConnectionFactory factory) => _factory = factory;

    public async Task ExecuteAsync(Func<SqlConnection, SqlTransaction, Task> operation, CancellationToken ct = default)
    {
        await using var conn = _factory.CreateConnection();
        await conn.OpenAsync(ct);
        await using var tx = conn.BeginTransaction();
        try
        {
            await operation(conn, tx);
            await tx.CommitAsync(ct);
        }
        catch
        {
            await tx.RollbackAsync(ct);
            throw;
        }
    }

    public async Task<T> ExecuteAsync<T>(Func<SqlConnection, SqlTransaction, Task<T>> operation, CancellationToken ct = default)
    {
        await using var conn = _factory.CreateConnection();
        await conn.OpenAsync(ct);
        await using var tx = conn.BeginTransaction();
        try
        {
            var result = await operation(conn, tx);
            await tx.CommitAsync(ct);
            return result;
        }
        catch
        {
            await tx.RollbackAsync(ct);
            throw;
        }
    }
}
```

## Service Usage

```csharp
// Application/OrderService.cs
public async Task<Result<OrderDto>> PlaceOrderAsync(PlaceOrderInput input, CancellationToken ct)
{
    OrderDto? result = null;

    await _uow.ExecuteAsync(async (conn, tx) =>
    {
        var entity = input.ToEntity();
        await _repo.InsertOrderAsync(conn, tx, entity);
        await _repo.InsertItemsAsync(conn, tx, entity.Items);

        var msg = OutboxMessage.Create(Constants.OutboxTypeOrderPlaced, new OrderPlacedPayload(entity));
        await _outboxRepo.InsertAsync(conn, tx, msg);

        result = entity.ToDto();
    });

    return result!;
}
```

Business write + outbox message in one transaction. Commit or rollback together.

---

# Dapper — Repository Pattern

```csharp
// Infrastructure/OrderRepository.cs
namespace Modules.Order.Infrastructure;

internal sealed class OrderRepository : IOrderRepository
{
    public async Task<OrderEntity?> GetByIdAsync(SqlConnection conn, Guid id, SqlTransaction? tx = null)
    {
        return await conn.QuerySingleOrDefaultAsync<OrderEntity>(
            """
            SELECT Id, UserId, Status, Total, CreatedAt, UpdatedAt
            FROM orders
            WHERE Id = @Id
            """,
            new { Id = id },
            transaction: tx);
    }

    public async Task InsertOrderAsync(SqlConnection conn, SqlTransaction tx, OrderEntity order)
    {
        await conn.ExecuteAsync(
            """
            INSERT INTO orders (Id, UserId, Status, Total, CreatedAt, UpdatedAt)
            VALUES (@Id, @UserId, @Status, @Total, @CreatedAt, @UpdatedAt)
            """,
            order,
            transaction: tx);
    }

    public async Task<(List<OrderEntity> items, int total)> SearchAsync(
        SqlConnection conn, Guid userId, int pageSize, int offset, SqlTransaction? tx = null)
    {
        var sql = """
            SELECT Id, UserId, Status, Total, CreatedAt, UpdatedAt
            FROM orders WHERE UserId = @UserId
            ORDER BY CreatedAt DESC
            OFFSET @Offset ROWS FETCH NEXT @PageSize ROWS ONLY;

            SELECT COUNT(*) FROM orders WHERE UserId = @UserId;
            """;

        using var multi = await conn.QueryMultipleAsync(sql,
            new { UserId = userId, PageSize = pageSize, Offset = offset },
            transaction: tx);

        var items = (await multi.ReadAsync<OrderEntity>()).AsList();
        var total = await multi.ReadSingleAsync<int>();
        return (items, total);
    }
}
```

**Rules:**
- Repository receives `SqlConnection` + `SqlTransaction?` — never owns the connection.
- No `using` on connection — the UoW or caller owns the lifetime.
- Raw SQL. No query builder. No ORM mapping attributes on entities.
- Column names match property names — Dapper maps by name.

---

# Cross-Module Communication

## Allowed Mechanisms

| Mechanism | When |
|-----------|------|
| **Direct call via `Contracts/` interface** | Synchronous — query user data from order module |
| **In-process EventBus** | Fire-and-forget — "order placed → update loyalty points" |
| **Outbox** | Durable — "order placed → send confirmation email" |

## Forbidden Mechanisms

| Mechanism | Why |
|-----------|-----|
| SQL joins across module tables | Breaks data ownership |
| Importing `Infrastructure/` from another module | Bypasses contracts |
| Sharing entity types | Couples schemas |
| Module-to-module HTTP calls in-process | Needless serialization overhead |

## EventBus

```csharp
// SharedKernel/IEventBus.cs
namespace SharedKernel;

public interface IEventBus
{
    void Subscribe<T>(Func<T, CancellationToken, Task> handler) where T : notnull;
    Task PublishAsync<T>(T @event, CancellationToken ct = default) where T : notnull;
}
```

Modules subscribe in `Module.cs`:

```csharp
// Order/Module.cs
public static IServiceCollection AddOrderModule(this IServiceCollection services, IConfiguration config)
{
    // ... register repos, services ...
    return services;
}

// Called after all modules registered — wire events
public static void SubscribeEvents(IEventBus bus)
{
    bus.Subscribe<OrderPlaced>(async (e, ct) => { /* handle */ });
}
```

---

# Per-Module Dependency Injection

## Principle

Each module has one `AddXxxModule(this IServiceCollection, IConfiguration)` extension method. It registers everything the module owns. Nothing leaks.

```csharp
// Order/Module.cs
// Dependencies: SharedKernel, User.Contracts, Payment.Contracts
namespace Modules.Order;

public static class OrderModule
{
    public static IServiceCollection AddOrderModule(this IServiceCollection services, IConfiguration config)
    {
        // Domain — no registration needed (pure types)

        // Infrastructure
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddTransient<IOrderPlacedHandler, OrderPlacedHandler>();

        // Application
        services.AddScoped<IOrderService, OrderService>();

        // Validators
        services.AddValidatorsFromAssemblyContaining<PlaceOrderRequestValidator>(ServiceLifetime.Singleton);

        return services;
    }
}
```

## What NOT to put in Module.cs

- No domain logic
- No SQL queries
- No validation rules
- Business rules belong in `Application/` or `Domain/`

---

# Background Workers — `IHostedService`

## Principle

Workers are `BackgroundService` implementations. They run on a timer or long-poll loop. Belong in the module's `Infrastructure/`.

```csharp
// Order/Infrastructure/OrderReaperWorker.cs
namespace Modules.Order.Infrastructure;

internal sealed class OrderReaperWorker : BackgroundService
{
    private readonly ILogger<OrderReaperWorker> _logger;
    private readonly IServiceScopeFactory _scopeFactory;

    public OrderReaperWorker(ILogger<OrderReaperWorker> logger, IServiceScopeFactory scopeFactory)
    {
        _logger = logger;
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await using var scope = _scopeFactory.CreateAsyncScope();
                var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
                var conn = scope.ServiceProvider.GetRequiredService<ISqlConnectionFactory>().CreateConnection();

                var deleted = await repo.DeleteExpiredOrdersAsync(conn, retentionDays: 30);
                if (deleted > 0)
                    _logger.LogInformation("[OrderReaperWorker] Cleaned {Count} expired orders", deleted);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _logger.LogError(ex, "[OrderReaperWorker] Reaper cycle failed");
            }

            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }
}
```

**Rules:**
- Use `IServiceScopeFactory` — workers are singletons; scoped services (repos) need a scope per tick.
- Catch all exceptions — one failed cycle must not kill the worker.
- Respect `CancellationToken` — graceful shutdown.

## Registration

```csharp
// Order/Module.cs
services.AddHostedService<OrderReaperWorker>();
```

---

# Outbox Pattern

## Principle

Same pattern as Go version. Service writes to `outbox` table in the same transaction as the business write. An outbox worker polls and dispatches to handlers.

## Outbox Entity

```csharp
// SharedKernel/Outbox/OutboxMessage.cs
namespace SharedKernel.Outbox;

public sealed class OutboxMessage
{
    public Guid Id { get; init; }
    public string Type { get; init; } = string.Empty;
    public string Payload { get; init; } = string.Empty;  // JSON
    public OutboxStatus Status { get; set; } = OutboxStatus.Pending;
    public int RetryCount { get; set; }
    public int MaxRetries { get; set; } = 3;
    public string? LastError { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }

    public static OutboxMessage Create<T>(string type, T payload, IClock clock)
        => new()
        {
            Id = Guid.CreateVersion7(),
            Type = type,
            Payload = JsonSerializer.Serialize(payload),
            CreatedAt = clock.UtcNow
        };
}

public enum OutboxStatus { Pending = 0, Processing = 1, Sent = 2, Failed = 3 }
```

## Schema

```sql
CREATE TABLE outbox (
    Id UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
    Type NVARCHAR(100) NOT NULL,
    Payload NVARCHAR(MAX) NOT NULL,
    Status TINYINT NOT NULL DEFAULT 0,
    RetryCount INT NOT NULL DEFAULT 0,
    MaxRetries INT NOT NULL DEFAULT 3,
    LastError NVARCHAR(MAX) NULL,
    CreatedAt DATETIME2 NOT NULL,
    ProcessedAt DATETIME2 NULL
);
CREATE INDEX IX_Outbox_Status ON Outbox (Status);
CREATE INDEX IX_Outbox_Type ON Outbox (Type);
CREATE INDEX IX_Outbox_CreatedAt ON Outbox (CreatedAt);
```

## IOutboxHandler

```csharp
// SharedKernel/Outbox/IOutboxHandler.cs
namespace SharedKernel.Outbox;

public interface IOutboxHandler
{
    string Type { get; }
    Task ExecuteAsync(OutboxMessage message, CancellationToken ct);
}
```

## Module Usage — Write Side

```csharp
// Application/OrderService.cs
await _uow.ExecuteAsync(async (conn, tx) =>
{
    await _repo.InsertOrderAsync(conn, tx, entity);
    var msg = OutboxMessage.Create(Constants.OutboxTypeOrderPlaced, new { entity.Id, entity.UserId });
    await _outboxRepo.InsertAsync(conn, tx, msg);
});
```

## Module Usage — Handler Side

```csharp
// Order/Infrastructure/OrderPlacedHandler.cs
internal sealed class OrderPlacedHandler : IOutboxHandler
{
    public string Type => Constants.OutboxTypeOrderPlaced;

    private readonly IEmailSender _mailer;

    public OrderPlacedHandler(IEmailSender mailer) => _mailer = mailer;

    public async Task ExecuteAsync(OutboxMessage message, CancellationToken ct)
    {
        var payload = JsonSerializer.Deserialize<OrderPlacedPayload>(message.Payload)!;
        await _mailer.SendOrderConfirmationAsync(payload, ct);
    }
}
```

## Registration

```csharp
// Order/Module.cs — TryAddEnumerable prevents last-write-wins overwrite
services.TryAddEnumerable(ServiceDescriptor.Transient<IOutboxHandler, OrderPlacedHandler>());
```

```csharp
// Notification/Module.cs
services.TryAddEnumerable(ServiceDescriptor.Transient<IOutboxHandler, MailSentHandler>());
```

Multiple modules register `IOutboxHandler`. Plain `AddTransient`/`AddScoped`/`AddSingleton` all follow the same rule: **last registration wins**. `TryAddEnumerable` appends instead of replacing — the worker injects `IEnumerable<IOutboxHandler>` and gets every handler from every module.

```csharp
// SharedKernel/Outbox/OutboxWorker.cs
internal sealed class OutboxWorker : BackgroundService
{
    private readonly IOutboxRepository _repo;
    private readonly Dictionary<string, IOutboxHandler> _handlerMap;
    private readonly int _batchSize;
    private readonly TimeSpan _pollInterval;

    public OutboxWorker(IOutboxRepository repo, IEnumerable<IOutboxHandler> handlers,
        IOptions<OutboxOptions> options)
    {
        _repo = repo;
        _handlerMap = handlers.ToDictionary(h => h.Type);
        _batchSize = options.Value.BatchSize;
        _pollInterval = TimeSpan.FromSeconds(options.Value.PollIntervalSeconds);
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var messages = await _repo.FetchPendingAsync(_batchSize);
            foreach (var msg in messages)
            {
                if (_handlerMap.TryGetValue(msg.Type, out var h))
                    await h.ExecuteAsync(msg, ct);
            }
            await Task.Delay(_pollInterval, ct);
        }
    }
}
```

## Outbox Infrastructure Registration

```csharp
// SharedKernel/Outbox/DependencyInjection.cs
namespace SharedKernel.Outbox;

public static class OutboxServiceCollectionExtensions
{
    public static IServiceCollection AddOutbox(this IServiceCollection services, IConfiguration config)
    {
        services.Configure<OutboxOptions>(config.GetSection("Outbox"));
        services.AddScoped<IOutboxRepository, OutboxRepository>();
        services.AddHostedService<OutboxWorker>();
        return services;
    }
}
```

## Outbox Type Keys in Domain/Constants.cs

```csharp
// Order/Domain/Constants.cs
namespace Modules.Order.Domain;

public static class Constants
{
    public const string OutboxTypeOrderPlaced = "order.placed";
}
```

---

# MailKit — Email Sender

```csharp
// SharedKernel/Mail/IMailSender.cs
namespace SharedKernel.Mail;

public interface IMailSender
{
    Task SendAsync(string to, string subject, string htmlBody, CancellationToken ct = default);
}
```

```csharp
// SharedKernel/Mail/MailKitSender.cs
using MailKit.Net.Smtp;
using MimeKit;

namespace SharedKernel.Mail;

internal sealed class MailKitSender : IMailSender, IDisposable
{
    private readonly SmtpClient _client;
    private readonly AppConfig _config;
    private readonly ILogger<MailKitSender> _logger;

    public MailKitSender(IOptions<AppConfig> config, ILogger<MailKitSender> logger)
    {
        _config = config.Value;
        _client = new SmtpClient();
        _logger = logger;
    }

    public async Task SendAsync(string to, string subject, string htmlBody, CancellationToken ct = default)
    {
        var message = new MimeMessage();
        message.From.Add(new MailboxAddress("App", _config.SmtpSender ?? "noreply@example.com"));
        message.To.Add(new MailboxAddress(to, to));
        message.Subject = subject;
        message.Body = new TextPart("html") { Text = htmlBody };

        await _client.ConnectAsync(_config.SmtpHost, _config.SmtpPort, MailKit.Security.SecureSocketOptions.StartTls, ct);
        await _client.AuthenticateAsync(_config.SmtpUser, _config.SmtpPass, ct);
        await _client.SendAsync(message, ct);
        await _client.DisconnectAsync(true, ct);

        _logger.LogInformation("[MailKitSender] Sent email to {Recipient} with subject '{Subject}'", to, subject);
    }

    public void Dispose() => _client.Dispose();
}
```

**Rules:**
- `SmtpClient` is a singleton — one instance, thread-safe per MailKit docs.
- Reconnect per send — connections time out. Don't keep a persistent connection.
- `MimeMessage` constructed per call — never reuse.

---

# Package Dependencies (NuGet)

## Required Packages

| Package | Purpose |
|---------|---------|
| `Dapper` | Micro-ORM, SQL query/execute |
| `Microsoft.Data.SqlClient` | SQL Server ADO.NET driver |
| `FluentValidation` | Request validation |
| `FluentValidation.DependencyInjectionExtensions` | Auto-register validators from assembly |
| `MailKit` | SMTP email sending |
| `Serilog` | Structured logging core |
| `Serilog.Extensions.Logging` | Bridge `ILogger<T>` → Serilog |
| `Serilog.Sinks.Console` | Console sink — local dev, Docker logs |
| `Serilog.Sinks.File` | Rolling file sink — persistent audit trail |
| `Serilog.Enrichers.Environment` | Machine name, process ID enrichment |
| `Serilog.Enrichers.Thread` | Thread ID enrichment |
| `Serilog.Exceptions` | Rich exception details — full stack as structured JSON |
| `Microsoft.Extensions.Hosting` | `BackgroundService` for workers |
| `Microsoft.AspNetCore.OpenApi` | Swagger/OpenAPI generation (.NET 10 built-in) |
| `Microsoft.AspNetCore.Authentication.JwtBearer` | JWT authentication |

## Framework-Provided (No NuGet Needed)

| Namespace | Purpose |
|-----------|---------|
| `System.Text.Json` | JSON serialization |
| `Microsoft.Extensions.DependencyInjection` | DI container |
| `Microsoft.Extensions.Configuration` | Config binding |
| `Microsoft.Extensions.Options` | `IOptions<T>` |
| `System.Threading.Channels` | Channel-based worker queues |

---

# Full Project Structure

```
src/
  Program.cs                           # Host builder, middleware order, module registration
  appsettings.json                     # Serilog config, connection strings, outbox options
  appsettings.Development.json         # Dev overrides

  SharedKernel/                        # Cross-cutting — every module depends on this
    Result.cs                          #   Result<T>, Error
    ApiResponse.cs                     #   ApiResponse, PaginationMetadata, PaginationParams
    BaseEntity.cs                      #   CreatedAt, UpdatedAt, MarkUpdated(IClock)
    IClock.cs                          #   Clock interface + SystemClock + LocalClock
    FakeClock.cs                       #   Deterministic clock for tests
    ISqlConnectionFactory.cs           #   Interface + SqlConnectionFactory + DatabaseKeys
    IUnitOfWork.cs                     #   Interface + UnitOfWork
    IEventBus.cs                       #   In-process pub/sub
    DependencyInjection.cs             #   AddDatabase(), AddSharedKernel()
    RequestIdMiddleware.cs
    RecovererMiddleware.cs
    RealIpMiddleware.cs
    RequestLoggingMiddleware.cs
    ValidationFilter.cs                #   .Validate<T>() endpoint filter
    ErrorExtensions.cs                 #   Error → IResult mapper

    Alert/
      AlertEntity.cs                   #   DB-mapped alert record
      AlertSink.cs                     #   Serilog sink — WARN+ERROR → channel
      AlertWorker.cs                   #   BackgroundService — persist + reaper

    Mail/
      IMailSender.cs                   #   Interface
      MailKitSender.cs                 #   Implementation

    Outbox/
      OutboxMessage.cs                 #   Entity + OutboxStatus enum
      OutboxOptions.cs                 #   PollIntervalSeconds, BatchSize
      IOutboxRepository.cs             #   Repository interface
      OutboxRepository.cs              #   Dapper implementation
      IOutboxHandler.cs                #   Handler interface (modules implement)
      OutboxWorker.cs                  #   BackgroundService — polls + dispatches
      DependencyInjection.cs           #   AddOutbox()

  Modules/
    Order/
      Contracts/
        IOrderService.cs               #   Public — exposed to other modules
        IOrderRepository.cs            #   Internal — module only
      Domain/
        OrderEntity.cs                 #   internal — DB-mapped
        OrderDto.cs                    #   internal — transfer object
        Constants.cs                   #   Status enums, outbox type keys
        OrderErrors.cs                 #   Error factories
        OrderEvents.cs                 #   Event records
      Application/
        OrderService.cs                #   Implements IOrderService
      Infrastructure/
        OrderRepository.cs             #   Dapper — primary DB
        OrderPlacedHandler.cs          #   IOutboxHandler
      Api/
        OrderEndpoints.cs              #   MapOrderEndpoints()
        PlaceOrderRequest.cs           #   Request + FluentValidation validator
        OrderResponse.cs               #   Response DTO
      Module.cs                        #   AddOrderModule()

    Payment/
      ...
    Notification/
      ...
    User/
      ...
```

Every module has the same shape. Adding a module = copying the folder structure + writing domain logic.

---

# Full Module Example

```
modules/Order/
  Contracts/
    IOrderService.cs              # Public interface + DI key
    IOrderRepository.cs            # Internal — repository contract
  Domain/
    OrderEntity.cs                 # DB-mapped, internal
    OrderDto.cs                    # DTO, internal
    Constants.cs                   # Status enums, outbox type keys
    OrderErrors.cs                 # Error factories
    OrderEvents.cs                 # Event records
  Application/
    OrderService.cs                # Implements IOrderService
  Infrastructure/
    OrderRepository.cs             # Dapper implementation
    OrderPlacedHandler.cs         # Outbox handler
  Api/
    OrderEndpoints.cs              # Minimal API endpoints
    PlaceOrderRequest.cs           # Request DTO + FluentValidation validator
    OrderResponse.cs               # Response DTO
  Module.cs                        # AddOrderModule()
```

---

# Adding a New Module Checklist

1. [ ] Create `modules/<Name>/` folder (PascalCase, singular, business capability).
2. [ ] Define `Contracts/` — public interfaces. Service interface. Repository interface (internal).
3. [ ] Define `Domain/` — entity class, DTO record, Constants, Errors, Events.
4. [ ] Define `Application/` — service implementation. Depends on Contracts + Domain only.
5. [ ] Define `Infrastructure/` — Dapper repository, MailKit sender, outbox handlers.
6. [ ] Define `Api/` — Minimal API endpoints, request DTOs + FluentValidation validators, response DTOs.
7. [ ] Define `Module.cs` — `AddXxxModule(this IServiceCollection, IConfiguration)` extension.
8. [ ] Add dependency declaration comment at top of `Module.cs`.
9. [ ] Register in `Program.cs` — `services.AddXxxModule(config)` + `app.MapXxxEndpoints()`.
10. [ ] Verify: `Program.cs` has zero knowledge of module internals — no handler names, no connection strings, no route paths.
11. [ ] Verify: grep for cross-module imports — only `Contracts/` types referenced.

---

# What to NEVER Do

| Violation | Why It's Broken | Fix |
|-----------|----------------|-----|
| Import `Order.Infrastructure` from `Payment` module | Bypasses contracts | Use `IOrderService` from `Contracts/` |
| Query `Orders` table from `Payment` repository | Shared data ownership | Call order module's service |
| Throw exceptions for domain errors | Exception-as-control-flow, expensive | Return `Result<T>` |
| `Configuration["key"]` in a service | Hidden dependency | Inject `IOptions<T>` |
| One giant `Api/` folder with all endpoints | No module boundaries | Each module has its own `Api/` folder |
| `using` connection inside repository | Repository doesn't own the connection | UoW/caller provides connection |
| Skip `Contracts/` — export from random files | No contract, boundaries rot | One folder per module for public API |
| Skip `Module.cs` — wire from `Program.cs` | Scatters DI knowledge | Each module has `AddXxxModule()` |
| Global static state for module data | Hidden coupling | Constructor injection + DI |

---

# Extracting a Module to a Service

The test of good boundaries:

1. Copy the module directory to a new repo.
2. Replace `Contracts/` implementations with HTTP/gRPC clients.
3. In the monolith, replace the direct call with the client implementing the same interface.
4. **Nothing else should change.**

---

# Enforcement

When writing or reviewing C# code in this project:

1. **Every `.cs` file belongs to a module** — no loose files at project root (except `Program.cs`).
2. **Every module has `Contracts/`** — the single public contract.
3. **Every module has `Module.cs`** — `AddXxxModule(this IServiceCollection)`. Self-contained DI.
4. **No cross-module `Infrastructure/` imports** — grep for `Modules/X/Infrastructure` from `Modules/Y`. Fails.
5. **No wiring in `Program.cs`** — it's a flat list of `AddXxxModule()` + `MapXxxEndpoints()`.
6. **Result<T> for all service methods that can fail** — no exceptions for expected errors.
7. **FluentValidation on every request** — no if-checks in the service.
8. **Dapper with raw SQL** — no query builder, no ORM.
9. **No shared database tables between modules** — each module copies what it needs, or calls the owning module's API.
10. When in doubt, ask: "If I extracted this module into its own service tomorrow, what would break?"
