---
name: csharp-modular-monolith
description: Enforce modular monolith architecture in C# (.NET 10) — strict module boundaries, per-module DI, Minimal APIs, Dapper, FluentValidation, MailKit, and the Result pattern.
version: 2.0.0
author: User
---

# C# Modular Monolith Architecture

## Purpose

Enforce modular monolith architecture in C# (.NET 10) projects. Every piece of code lives inside a module. Every module owns its domain. Cross-module communication follows strict rules. The goal: monolith deployment with microservice discipline — replaceable modules, no spaghetti.

---

# Core Principle

> A module is a self-contained business capability. It owns its data, its logic, and its public contract via `Contracts/`. No other module may reach into its internals.

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

### Rules

| Rule | Why |
|------|-----|
| PascalCase for types, `I` prefix for interfaces | .NET convention — consistent across ecosystem |
| `_camelCase` for private fields | Distinguishes locals from fields at a glance |
| `Async` suffix on async methods | Caller knows it's awaitable |
| No `Util`, `Common`, `Misc`, `Helpers` | Name magnets — everything ends up there |
| Records for DTOs | Immutable by default, value-based equality |
| `Base` prefix for abstract base classes | Not `AbstractEntity` — `BaseEntity` is shorter |

---

# Module Definition

## What Makes a Module

A module is a **folder** inside a single web project. One solution, one `.csproj`, one deployment unit. Modules are folder boundaries, not assembly boundaries.

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

**Module = business capability.** Not a layer. Not a utility drawer.

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

## Package-by-Package

### `Domain/` — Pure Business. No Dependencies.

```csharp
// Modules/Order/Domain/OrderEntity.cs
namespace Modules.Order.Domain;

internal sealed class OrderEntity
{
    public Guid Id { get; set; }
    public Guid UserId { get; set; }
    public string Status { get; set; } = "pending";
    public decimal Total { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

- Pure C#. No attributes, no NuGet, no framework references.
- Inherits from `BaseEntity` (SharedKernel) for timestamps.
- DTOs, Errors, Constants, Events all live here.

### `Contracts/` — What the Module Promises and Needs.

```csharp
// Modules/Order/Contracts/IOrderService.cs
namespace Modules.Order.Contracts;

public interface IOrderService
{
    Task<Result<OrderDto>> PlaceOrderAsync(PlaceOrderInput input, CancellationToken ct);
    Task<Result<OrderDto>> GetOrderAsync(Guid id, CancellationToken ct);
    Task<Result<List<OrderDto>>> ListByUserAsync(Guid userId, CancellationToken ct);
}
```

```csharp
// Modules/Order/Contracts/IOrderRepository.cs
namespace Modules.Order.Contracts;

internal interface IOrderRepository
{
    Task<OrderEntity?> GetByIdAsync(SqlConnection conn, Guid id, SqlTransaction? tx = null);
    Task InsertOrderAsync(SqlConnection conn, SqlTransaction tx, OrderEntity order);
}
```

- **Public** interfaces (`IOrderService`) = other modules can call.
- **Internal** interfaces (`IOrderRepository`) = module-only, other modules never see.

### `Application/` — Business Logic.

```csharp
// Modules/Order/Application/OrderService.cs
namespace Modules.Order.Application;

internal sealed class OrderService : IOrderService
{
    private readonly IOrderRepository _repo;
    private readonly IUnitOfWork _uow;
    private readonly IClock _clock;
    private readonly ILogger<OrderService> _logger;

    public OrderService(IOrderRepository repo, IUnitOfWork uow, IClock clock, ILogger<OrderService> logger)
    {
        _repo = repo;
        _uow = uow;
        _clock = clock;
        _logger = logger;
    }

    public async Task<Result<OrderDto>> PlaceOrderAsync(PlaceOrderInput input, CancellationToken ct)
    {
        var entity = new OrderEntity
        {
            Id = Guid.CreateVersion7(),
            UserId = input.UserId,
            Status = "pending",
            Total = input.Items.Sum(i => i.Price * i.Quantity),
            CreatedAt = _clock.UtcNow,
            UpdatedAt = _clock.UtcNow
        };

        await _uow.ExecuteAsync(async (conn, tx) =>
        {
            await _repo.InsertOrderAsync(conn, tx, entity);
            // Outbox message in same transaction
            var msg = OutboxMessage.Create("order.placed", new { entity.Id, entity.UserId }, _clock);
            await _outboxRepo.InsertAsync(conn, tx, msg);
        }, ct);

        _logger.LogInformation("[OrderService.PlaceOrder] Order {OrderId} placed", entity.Id);
        return entity.ToDto();
    }
}
```

- Implements the `Contracts/` interface.
- Owns the transaction scope via `IUnitOfWork`.
- Returns `Result<T>` — never throws for domain errors.

### `Infrastructure/` — Adapters (Dapper, MailKit, etc.).

```csharp
// Modules/Order/Infrastructure/OrderRepository.cs
namespace Modules.Order.Infrastructure;

internal sealed class OrderRepository : IOrderRepository
{
    public async Task<OrderEntity?> GetByIdAsync(SqlConnection conn, Guid id, SqlTransaction? tx = null)
    {
        return await conn.QuerySingleOrDefaultAsync<OrderEntity>(
            """SELECT Id, UserId, Status, Total, CreatedAt, UpdatedAt FROM orders WHERE Id = @Id""",
            new { Id = id }, transaction: tx);
    }

    public async Task InsertOrderAsync(SqlConnection conn, SqlTransaction tx, OrderEntity order)
    {
        await conn.ExecuteAsync(
            """INSERT INTO orders (Id, UserId, Status, Total, CreatedAt, UpdatedAt) VALUES (@Id, @UserId, @Status, @Total, @CreatedAt, @UpdatedAt)""",
            order, transaction: tx);
    }
}
```

- Receives `SqlConnection` + `SqlTransaction?` — never owns the connection.
- Raw SQL. No query builders, no ORM attributes on entities.

### `Api/` — Primary Adapter (Minimal API).

```csharp
// Modules/Order/Api/OrderEndpoints.cs
namespace Modules.Order.Api;

public static class OrderEndpoints
{
    public static RouteGroupBuilder MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/v1/orders")
            .WithTags("Orders")
            .RequireAuthorization();

        group.MapPost("/", PlaceOrderAsync).Validate<PlaceOrderRequest>();
        group.MapGet("/{id:guid}", GetOrderAsync);
        group.MapGet("/", ListOrdersAsync);

        return group;
    }

    private static async Task<IResult> PlaceOrderAsync(
        PlaceOrderRequest request, IOrderService service, CancellationToken ct)
    {
        var result = await service.PlaceOrderAsync(request.ToInput(), ct);
        return result.ToResult();
    }
}
```

- Static methods. No controllers.
- Request DTO + FluentValidation validator live here.
- Calls `Match` on `Result<T>` to decide HTTP response.

### `Module.cs` — Composition Root.

```csharp
// Modules/Order/Module.cs
// Dependencies: SharedKernel, User.Contracts, Payment.Contracts
namespace Modules.Order;

public static class OrderModule
{
    public static IServiceCollection AddOrderModule(this IServiceCollection services, IConfiguration config)
    {
        // Infrastructure
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddTransient<IOutboxHandler, OrderPlacedHandler>();

        // Application
        services.AddScoped<IOrderService, OrderService>();

        // Validators
        services.AddValidatorsFromAssemblyContaining<PlaceOrderRequestValidator>(ServiceLifetime.Singleton);

        return services;
    }
}
```

One method. Everything the module needs. Nothing leaks.

### Type Separation

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

### Multi-Instance Adapters — Same Interface, Different Configuration

#### The Problem

A module might connect to two databases (primary + read replica) or two SMTP servers (transactional + marketing). The `IMailSender` interface is the same — only the config differs.

#### Strategy: Keyed Registration (.NET 8+)

```csharp
// Contracts/Key.cs
namespace Modules.Order.Contracts;

public static class OrderKeys
{
    public const string ReadReplica = "order_read_replica";
}
```

```csharp
// SharedKernel/DatabaseKeys.cs
namespace SharedKernel;

public static class DatabaseKeys
{
    public const string Primary = "primary";
    public const string ReadReplica = "read_replica";
}
```

```csharp
// Program.cs — keyed singletons
builder.Services.AddKeyedSingleton<ISqlConnectionFactory>(
    DatabaseKeys.Primary,
    new SqlConnectionFactory(config.GetConnectionString("Primary")!));

builder.Services.AddKeyedSingleton<ISqlConnectionFactory>(
    DatabaseKeys.ReadReplica,
    new SqlConnectionFactory(config.GetConnectionString("ReadReplica")!));
```

Usage — inject the right instance via `[FromKeyedServices]`:

```csharp
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

#### Rules

| Rule | Why |
|------|-----|
| Key constants in the module's `Contracts/` | Public — other modules can use them |
| `Instanced[T]` wrapper for multiple instances | One DI registration, multiple named configs |
| Key pattern: `<type>.<role>` | Greppable, consistent |
| Same interface, different config | Adapter swap without changing the contract |

### Data Flow

```
Client Request
     │
     ▼
  Middleware Pipeline          ┌──────────────────┐
  (RequestId → Recoverer →     │    SharedKernel   │
   RealIp → RateLimit → Auth)  │  Result, ApiResp  │
     │                         │  BaseEntity, IClock│
     ▼                         │  ISqlConnFactory   │
  Api/OrderEndpoints ────────► │  IUnitOfWork      │
     │                         └──────────────────┘
     ▼
  Application/OrderService ───► Domain/OrderEntity
     │
     ├──► Infrastructure/OrderRepository ──► SQL Server
     └──► OutboxMessage ──────────────────────► outbox table
                                                        │
                                                        ▼
                                              OutboxWorker ──► Handler
```

---

# The Result Pattern

## Principle

A simple `Result<T>` encodes success or failure in one type without exceptions for expected errors. Success wraps an `Outcome<T>` record with a status code. Failure wraps a `Failure` record with an HTTP status, title, and detail.

**No union types.** No OneOf. No third-party libraries. Three lightweight types.

## Types — `Shared/Results/`

### `Result<T>` — Discriminated Union

```csharp
namespace api.Shared.Results;

public readonly record struct Result<T>
{
    public Outcome<T>? Value { get; }
    public Failure? Failure { get; }
    public bool IsSuccess => Value is not null;
    public bool IsFailure => Failure is not null;

    private Result(Outcome<T> outcome) { Value = outcome; Failure = null; }
    private Result(Failure failure) { Value = null; Failure = failure; }

    public TResult Match<TResult>(Func<Outcome<T>, TResult> onSuccess, Func<Failure, TResult> onFailure)
        => IsSuccess ? onSuccess(Value!) : onFailure(Failure!);

    public static implicit operator Result<T>(Outcome<T> outcome) => new(outcome);
    public static implicit operator Result<T>(Failure failure) => new(failure);
}
```

### `Outcome<T>` — Success Value + Status Code

```csharp
namespace api.Shared.Results;

public sealed record Outcome<T>(T Value, int StatusCode = 200)
{
    public static Outcome<T> Created(T value) => new(value, 201);
    public static Outcome<T> Accepted(T value) => new(value, 202);
}
```

### `Outcome` — Static Factory

```csharp
namespace api.Shared.Results;

public static class Outcome
{
    public static Result<T> Success<T>(T value) => new Outcome<T>(value);
    public static Result<T> Created<T>(T value) => Outcome<T>.Created(value);
    public static Result<T> Accepted<T>(T value) => Outcome<T>.Accepted(value);
    public static Result<T> Failed<T>(Failure failure) => failure;
}
```

### `Failure` — Standardized Error Record

```csharp
namespace api.Shared.Results;

public sealed record Failure(int StatusCode, string Title, string Detail)
{
    public static Failure NotFound(string detail) =>
        new(404, "Not Found", detail);
    public static Failure Conflict(string detail) =>
        new(409, "Conflict", detail);
    public static Failure BadRequest(string detail) =>
        new(400, "Bad Request", detail);
    public static Failure ValidationError(string detail, object? errors = null)
        => new(422, "Validation Failed", detail) { Errors = errors };
    public static Failure Internal(string detail = "An unexpected server error occurred.") =>
        new(500, "Internal Server Error", detail);
    public static Failure Unauthorized(string detail = "Authentication is required.") =>
        new(401, "Unauthorized", detail);
    public static Failure Forbidden(string detail = "You do not have permission.") =>
        new(403, "Forbidden", detail);

    public object? Errors { get; init; }
}
```

## Usage in Services

```csharp
// Success — explicit (recommended)
return Outcome.Success(items);         // 200 OK
return Outcome.Created(dto);           // 201 Created
return Outcome.Accepted(dto);          // 202 Accepted

// Failure — via static factories
return Outcome.Failed(Failure.NotFound("Order not found"));
return Outcome.Failed(Failure.Conflict("Already cancelled"));
return Outcome.Failed(Failure.Internal());
```

No per-module error classes needed — `Failure.NotFound()`, `Failure.Conflict()`, `Failure.Internal()` cover all cases.

## Mapping to HTTP in Endpoints

```csharp
var result = await service.GetAllAsync(ct);
return result.ToResult();  // single call — delegates to Outcome or Failure path
```

The `ToResult()` extension handles both paths:
- `Outcome<T>` → `ResponseResult` → `ApiResponse.ForSuccess(code, data)`
- `Failure` → `ResponseResult` → `ApiResponse.ForFailure(failure)`

`ResponseResult` is a custom `IResult` that stamps `TraceId` on failures and serializes as JSON.

### Rules

| Rule | Why |
|------|-----|
| `Result<T>` for all service methods that can fail | No exceptions for expected paths |
| `Outcome.Success()` / `Outcome.Created()` explicit factories | Clear intent, no magic |
| `Failure` static factories instead of module error classes | Single source of truth, no duplication |
| `Failure.Internal()` always returns generic 500 | Never leaks exception details to client |
| `result.ToResult()` in endpoints | One call, both paths handled |

---

# UUID Identifiers

## Principle

Every entity ID is a `Guid`. No auto-increment integers.

```csharp
internal sealed class OrderEntity
{
    public Guid Id { get; set; }             // Guid — assigned at creation, never auto-generated
    public Guid UserId { get; set; }
    public string Status { get; set; } = "pending";
    public decimal Total { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

## Guid Creation

- `Guid.CreateVersion7()` (.NET 9+) — time-sorted, clustered index friendly.
- `Guid.NewGuid()` (V4) — for public-facing IDs where timing should not leak.
- Always create the Guid in application code, never let the DB generate it.

## DB Storage — SQL Server

```sql
CREATE TABLE orders (
    id UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
    user_id UNIQUEIDENTIFIER NOT NULL,
    status NVARCHAR(50) NOT NULL,
    total DECIMAL(18,2) NOT NULL,
    created_at DATETIME2 NOT NULL,
    updated_at DATETIME2 NOT NULL
);
```

Dapper column mapping — add once in `Program.cs`:

```csharp
Dapper.DefaultTypeMap.MatchNamesWithUnderscores = true;
```

Now `user_id` column auto-maps to `UserId` property. No `[Column]` attributes.

### Rules

| Rule | Why |
|------|-----|
| `Guid.CreateVersion7()` by default | Time-sorted — avoids index fragmentation from V4 randomness |
| `Guid.NewGuid()` for public-facing IDs | Prevents timing-based enumeration |
| No `NEWSEQUENTIALID()` | Application assigns IDs, not the DB |
| No auto-increment `int` | Prevents merge conflicts, enumeration attacks, and entity leak |

---

# Database — Timestamps

## Principle

Every entity table has `CreatedAt` and `UpdatedAt`. Always UTC — local time is a presentation concern. Never call `DateTime.UtcNow` directly in domain or service code — go through `IClock`.

## IClock — Abstract the System Clock

```csharp
namespace SharedKernel;

public interface IClock { DateTime UtcNow { get; } }

internal sealed class SystemClock : IClock
{
    public DateTime UtcNow => DateTime.UtcNow;
}

internal sealed class LocalClock : IClock
{
    public DateTime UtcNow => DateTime.Now; // local, not UTC
}
```

Registration — one line:

```csharp
builder.Services.AddSingleton<IClock>(new SystemClock());  // production — UTC
```

## BaseEntity

```csharp
namespace SharedKernel;

public abstract class BaseEntity
{
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }

    public void MarkUpdated(IClock clock) => UpdatedAt = clock.UtcNow;
}
```

All entities inherit from `BaseEntity`. Set `CreatedAt` at construction via clock, call `MarkUpdated(clock)` on every mutation.

## Deterministic Tests

```csharp
public sealed class FakeClock : IClock
{
    public DateTime UtcNow { get; set; } = new DateTime(2026, 1, 1, 0, 0, 0, DateTimeKind.Utc);
    public void Advance(TimeSpan delta) => UtcNow = UtcNow.Add(delta);
}
```

### Rules

| Rule | Why |
|------|-----|
| `IClock` everywhere — never `DateTime.UtcNow` | Deterministic tests, timezone swap without code change |
| BaseEntity.MarkUpdated(clock) | Caller passes clock explicitly |
| UTC in DB, convert at presentation | No timezone ambiguity |
| Infrastructure tables: CreatedAt only | Append-only — no mutation |

---

# Route Registration — Inside the Module

```csharp
namespace Modules.Order.Api;

public static class OrderEndpoints
{
    public static RouteGroupBuilder MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/v1/orders")
            .WithTags("Orders")
            .RequireAuthorization();

        group.MapGet("/{id:guid}", GetOrderAsync);
        group.MapPost("/", PlaceOrderAsync).Validate<PlaceOrderRequest>();
        group.MapPost("/{id:guid}/cancel", CancelOrderAsync);
        group.MapGet("/", ListOrdersAsync);
        return group;
    }

    private static async Task<IResult> GetOrderAsync(
        Guid id, IOrderService service, CancellationToken ct)
    {
        var result = await service.GetOrderAsync(id, ct);
        return result.ToResult();
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
var builder = WebApplication.CreateBuilder(args);

// Infrastructure
builder.Services.AddDatabase(builder.Configuration);
builder.Services.AddOutbox();

// Modules — one call per module
builder.Services.AddOrderModule(builder.Configuration);
builder.Services.AddPaymentModule(builder.Configuration);

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

app.Run();
```

One `.AddXxxModule()` per module (DI) + one `.MapXxxEndpoints()` per module (routing). `Program.cs` is a flat list — no knowledge of module internals.

---

# Structured HTTP Responses

## ApiResponse Envelope

Every HTTP response uses a single envelope type. No free-form JSON.

```csharp
namespace SharedKernel;

public sealed class ApiResponse
{
    public bool Success { get; init; }
    public int Code { get; init; }
    public string Message { get; init; } = string.Empty;
    public object? Data { get; init; }
    public PageInfo? Pagination { get; init; }
    public object? Errors { get; init; }
    public string? TraceId { get; init; }

    // 2xx
    public static ApiResponse Success(object? data = null, string message = "Success")
        => new() { Success = true, Code = 200, Message = message, Data = data };
    public static ApiResponse Created(object? data = null, string message = "Created")
        => new() { Success = true, Code = 201, Message = message, Data = data };
    public static ApiResponse Paginated(object data, PageInfo meta)
        => new() { Success = true, Code = 200, Message = "Success", Data = data, Pagination = meta };

    // 4xx
    public static ApiResponse BadRequest(string message = "Bad request")
        => new() { Success = false, Code = 400, Message = message };
    public static ApiResponse ValidationError(object errors)
        => new() { Success = false, Code = 422, Message = "Validation failed", Errors = errors };
    public static ApiResponse NotFound(string message = "Not found")
        => new() { Success = false, Code = 404, Message = message };

    // 5xx — generic, never expose internals
    public static ApiResponse InternalError()
        => new() { Success = false, Code = 500, Message = "Internal Server Error" };
}
```

## Why `success` Instead of `status`

The HTTP status code already says 200/400/500. The `success` field is the response-level indicator: `true` for 2xx, `false` otherwise. It prevents the client from checking `code >= 200 && code < 300`.

## Pagination

```csharp
namespace SharedKernel;

public sealed record PaginationParams(int Page = 1, int PageSize = 20)
{
    public int Page { get; } = Math.Max(1, Page);
    public int PageSize { get; } = Math.Clamp(PageSize, 1, 100);
    public int Offset => (Page - 1) * PageSize;
}

public sealed record PageInfo(int Page, int PageSize, int TotalItems)
{
    public int TotalPages => (int)Math.Ceiling((double)TotalItems / PageSize);
}
```

### Pagination Flow — Repository → Service → Handler

**Repository** receives raw primitives:

```csharp
public async Task<(List<OrderEntity> items, int total)> SearchAsync(
    SqlConnection conn, Guid userId, int pageSize, int offset, SqlTransaction? tx = null)
{
    // Multi-result query: SELECT ... OFFSET @Offset FETCH NEXT @PageSize ROWS ONLY; SELECT COUNT(*)
    using var multi = await conn.QueryMultipleAsync(sql, new { UserId = userId, PageSize = pageSize, Offset = offset }, transaction: tx);
    var items = (await multi.ReadAsync<OrderEntity>()).AsList();
    var total = await multi.ReadSingleAsync<int>();
    return (items, total);
}
```

**Service** calls `params.Offset()`:

```csharp
var (items, total) = await _repo.SearchAsync(conn, userId, params.PageSize, params.Offset);
return (items.ConvertAll(e => e.ToDto()), total);
```

**Handler** converts to `PageInfo`:

```csharp
group.MapGet("/", async ([AsParameters] PaginationParams p, IOrderService service, CancellationToken ct) =>
{
    var result = await service.ListByUserAsync(p, ct);
    return result.Match(
        onSuccess: t => Results.Ok(new ApiResponse
        {
            Success = true,
            Code = 200,
            Data = t.items,
            Pagination = new PageInfo(p.Page, p.PageSize, t.total)
        }),
        onFailure: failure => failure.ToError());
});
```

### Pagination Rules

| Rule | Why |
|------|-----|
| Transport-agnostic `PaginationParams` in SharedKernel | Works for any HTTP framework |
| Repository: raw `int pageSize, int offset` | No params types in the query layer |
| `Page` defaults to 1, `PageSize` defaults to 20 | Sensible defaults, no required params |
| `PageSize` clamped to [1, 100] | Prevents abuse (no `?pageSize=999999`) |

## Result to HTTP — ResponseResult

The `ResponseResult` class is a custom `IResult` that wraps `ApiResponse` and stamps `TraceId` on failures. Extensions on `Outcome<T>` and `Failure` create it:

```csharp
// Shared/Extensions/ResultExtensions.cs
namespace api.Shared.Extensions;

public static class OutcomeExtensions
{
    public static IResult ToResult<T>(this Outcome<T> outcome)
        => new ResponseResult(outcome.StatusCode, outcome.Value);
}

public static class FailureExtensions
{
    public static IResult ToError(this Failure failure)
        => new ResponseResult(failure);
}

public static class ResultExtensions
{
    public static IResult ToResult<T>(this Result<T> result)
        => result.Match(
            onSuccess: outcome => outcome.ToResult(),
            onFailure: failure => failure.ToError());
}

internal sealed class ResponseResult : IResult
{
    private readonly int _statusCode;
    private readonly ApiResponse _response;

    public ResponseResult(Failure failure)
    {
        _response = ApiResponse.ForFailure(failure);
        _statusCode = failure.StatusCode;
    }

    public ResponseResult(int statusCode, object? data)
    {
        _response = ApiResponse.ForSuccess(statusCode, data);
        _statusCode = statusCode;
    }

    public async Task ExecuteAsync(HttpContext httpContext)
    {
        if (!_response.Success)
            _response.TraceId = httpContext.TraceIdentifier;

        httpContext.Response.StatusCode = _statusCode;
        httpContext.Response.ContentType = "application/json";
        await JsonSerializer.SerializeAsync(httpContext.Response.Body, _response, JsonOptions);
    }
}
```

`ApiResponse` factories build the envelope:

```csharp
// Shared/Models/ApiResponse.cs
namespace api.Shared.Models;

public sealed class ApiResponse
{
    public bool Success { get; init; }
    public int Code { get; init; }
    public string? Message { get; init; }
    public object? Data { get; init; }
    public object? Errors { get; init; }
    public string? TraceId { get; set; }

    public static ApiResponse ForSuccess(int statusCode, object? data) => new()
    {
        Success = true, Code = statusCode, Data = data
    };

    public static ApiResponse ForFailure(Failure failure) => new()
    {
        Success = false, Code = failure.StatusCode,
        Message = failure.Detail, Errors = failure.Errors
    };
}
```

All endpoints use `result.ToResult()` — no manual matching, no repeated response construction.

## Empty List → `[]`, Never `null`

```csharp
// ❌ Wrong — returns {"data": null}
h.Success(null);

// ✅ Correct — returns {"data": []}
h.Success(orders ?? []);
```

## Handler Pattern

```
1. Decode request      ASP.NET binds from body/query/path
2. Validate request    FluentValidation (via .Validate<T>() filter or explicit)
3. Call service        service.PlaceOrderAsync(input, ct) → Result<T>
4. Call ToResult       result.ToResult() → IResult
5. Return IResult      handled by ASP.NET → ResponseResult stamps TraceId on failures
```

### Handler Implementation Convention

```csharp
private static async Task<IResult> PlaceOrderAsync(
    PlaceOrderRequest request,
    IOrderService orderService,
    CancellationToken ct)
{
    var result = await orderService.PlaceOrderAsync(request.ToInput(), ct);
    return result.ToResult();
}
```

### Request Validation — `.Validate<T>()` Filter

```csharp
// Shared/Filters/ValidationFilter.cs
namespace api.Shared.Filters;

public static class ValidationFilter
{
    public static RouteHandlerBuilder Validate<T>(this RouteHandlerBuilder builder) where T : class
    {
        builder.AddEndpointFilter(async (context, next) =>
        {
            var request = context.Arguments.OfType<T>().FirstOrDefault();
            if (request is null)
                return HttpResults.BadRequest(ApiResponse.ForFailure(Failure.BadRequest("Invalid request body")));

            var validator = context.HttpContext.RequestServices.GetRequiredService<IValidator<T>>();
            var result = await validator.ValidateAsync(request);
            if (result.IsValid) return await next(context);

            var errors = result.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage).ToArray());

            return Results.UnprocessableEntity(ApiResponse.ValidationError(errors));
        });
        return builder;
    }
}
```

Usage in endpoint registration:

```csharp
group.MapPost("/", PlaceOrderAsync).Validate<PlaceOrderRequest>();
```

## Swagger / OpenAPI Annotations

.NET 10 has built-in OpenAPI via `Microsoft.AspNetCore.OpenApi`. Minimal APIs auto-generate schema from request/response types. Annotate with attributes on request/response DTOs — not on endpoints.

### Setup — Program.cs

```csharp
builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();  // serves /openapi/v1.json
}
```

### Annotate Request/Response DTOs

```csharp
using System.ComponentModel;
using System.Text.Json.Serialization;

namespace Modules.Order.Api;

public sealed class PlaceOrderRequest
{
    /// <summary>User placing the order</summary>
    /// <example>3fa85f64-5717-4562-b3fc-2c963f66afa6</example>
    public Guid UserId { get; init; }

    /// <summary>Order line items</summary>
    [Required, MinLength(1)]
    public List<OrderItemRequest> Items { get; init; } = [];
}

public sealed class OrderItemRequest
{
    /// <summary>Product identifier</summary>
    public Guid ProductId { get; init; }

    /// <summary>Quantity to order</summary>
    /// <example>2</example>
    [Range(1, 100)]
    public int Quantity { get; init; }
}

public sealed class OrderResponse
{
    public Guid Id { get; init; }
    public string Status { get; init; } = string.Empty;
    public decimal Total { get; init; }
    public DateTime CreatedAt { get; init; }
}
```

### Endpoint Metadata — Response Types

```csharp
public static RouteGroupBuilder MapOrderEndpoints(this IEndpointRouteBuilder app)
{
    var group = app.MapGroup("/v1/orders").WithTags("Orders");

    group.MapPost("/", PlaceOrderAsync)
        .Produces<ApiResponse>(StatusCodes.Status200OK)
        .Produces<ApiResponse>(StatusCodes.Status400BadRequest)
        .Produces<ApiResponse>(StatusCodes.Status422UnprocessableEntity)
        .ProducesProblem(StatusCodes.Status500InternalServerError);
}
```

### Rules

| Rule | Why |
|------|-----|
| XML doc comments on DTOs (`/// <summary>`, `/// <example>`) | Swagger generates rich descriptions |
| `[Required]`, `[Range]`, `[MinLength]` on DTO properties | Schema constraints without Swagger-specific attributes |
| `.Produces<T>(statusCode)` on every endpoint | Swagger knows what each endpoint returns |
| Response types reference `ApiResponse`, not the inner DTO | Consistent envelope |
| No attributes on endpoint methods | DTOs are the schema — endpoints just register routes |
| `[JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]` on optional fields | Cleaner JSON output |

---

# Module Errors

## Two Error Categories

| Category | Type | HTTP Status | Example |
|----------|------|-------------|---------|
| Domain error | `Failure` record | 4xx | `Failure.NotFound("...")`, `Failure.Conflict("...")` |
| Internal error | Exception | 500 | DB connection failure, timeout |

## Failure Factories — No Per-Module Error Classes

```csharp
// Shared/Results/Failure.cs
namespace api.Shared.Results;

public sealed record Failure(int StatusCode, string Title, string Detail)
{
    public static Failure NotFound(string detail) =>
        new(404, "Not Found", detail);
    public static Failure Conflict(string detail) =>
        new(409, "Conflict", detail);
    public static Failure BadRequest(string detail) =>
        new(400, "Bad Request", detail);
    public static Failure Internal(string detail = "An unexpected server error occurred.") =>
        new(500, "Internal Server Error", detail);

    public object? Errors { get; init; }
}
```

No per-module error classes. All modules use `Failure.NotFound()`, `Failure.Conflict()`, `Failure.Internal()` — consistent, greppable.

## Service — Return Failures, Wrap Unexpected Ones

```csharp
public async Task<Result<OrderDto>> CancelOrderAsync(Guid orderId, Guid userId, CancellationToken ct)
{
    var order = await _repo.GetByIdAsync(conn, orderId);
    if (order is null) return Outcome.Failed(Failure.NotFound($"Order {orderId} not found."));
    if (order.Status == "cancelled") return Outcome.Failed(Failure.Conflict($"Order {orderId} is already cancelled."));

    order.Status = "cancelled";
    order.MarkUpdated(_clock);
    await _repo.UpdateAsync(conn, order);

    return Outcome.Success(order.ToDto());
}
```

## Handler — One Pattern for All Modules

```csharp
private static async Task<IResult> CancelOrderAsync(
    Guid id, IOrderService service, CancellationToken ct)
{
    var result = await service.CancelOrderAsync(id, ct);
    return result.ToResult();
}
```

### Error Flow

```
Service returns Failure ──► FailureExtensions.ToError() ──► ResponseResult(failure)
                                 │
                                 ▼
                    ApiResponse.ForFailure(failure)
                                 │
                     ┌───────────┼──────────────┐
                     ▼           ▼               ▼
               NotFound     Conflict         Internal
              (404, data)  (409, data)     (500, generic)
                     ▼           ▼               ▼
              { success: false, code: 404/409/500, message: ..., traceId: ... }
```

### Rules

| Rule | Why |
|------|-----|
| Failures use `Failure` record, not custom error classes | No duplication — one type covers all modules |
| `Failure.Internal()` always returns generic 500 | Never leaks exception details to client |
| `Failure.NotFound()` / `Failure.Conflict()` for domain errors | Consistent error contract across modules |
| `Outcome.Failed(Failure)` to return from services | Explicit, readable |
| Service never throws for domain errors | Return errors via Result. Middleware catches only unexpected exceptions. |

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
│   BaseEntity, Pagination, IClock)    │
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
// Modules/Order/Module.cs
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
- `ApiResponse` + `PageInfo` + `PaginationParams` — HTTP envelope
- `BaseEntity` — CreatedAt/UpdatedAt contract
- `IClock` + `SystemClock` — abstracted system clock (UTC by default)
- `ISqlConnectionFactory` + `IUnitOfWork` — transaction management
- `IEventBus` — in-process pub/sub
- `ILogger<T>` (.NET built-in — Serilog implements it)

## What Does NOT Belong in Shared

- Any business entity (`User`, `Order`)
- Any module-specific DTO or enum
- Any repository or service interface
- Any validation rule

### Shared Kernel Rules

| Rule | Why |
|------|-----|
| High bar for inclusion | Every shared type is a coupling point |
| No business logic | Shared kernel is infrastructure, not domain |
| No module-specific types | Belongs in that module's `Domain/` or `Contracts/` |
| Value objects in SharedKernel | `Email`, `Money`, `Address` — cross-cutting domain primitives |

---

# FluentValidation — Request Validation

## Principle

Every request type has a corresponding `AbstractValidator<T>`. Validation lives in `Api/` alongside the request type — never in the service.

```csharp
namespace Modules.Order.Api;

public sealed class PlaceOrderRequest
{
    public Guid UserId { get; init; }
    public List<OrderItemRequest> Items { get; init; } = [];
    public PlaceOrderInput ToInput() => new(UserId, Items.ConvertAll(i => i.ToDto()));
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
```

## Validation Filter (.Validate<T>()) — in SharedKernel

```csharp
group.MapPost("/", PlaceOrderAsync).Validate<PlaceOrderRequest>();
```

Registered in `Module.cs`:

```csharp
services.AddValidatorsFromAssemblyContaining<PlaceOrderRequestValidator>(ServiceLifetime.Singleton);
```

### Rules

| Rule | Why |
|------|-----|
| Validator next to request DTO | Co-located — one file to change when the request changes |
| Validation in `Api/`, not `Application/` | HTTP concern — validation is protocol-specific |
| .Validate<T>() filter | DRY — one filter, all endpoints |
| Singleton validators | No per-request allocation overhead |

---

# Middleware

## Middleware Order

```csharp
var app = builder.Build();

app.UseMiddleware<RequestIdMiddleware>();       // 1. Tag every request
app.UseMiddleware<RecovererMiddleware>();        // 2. Catch panics, log, return 500
app.UseMiddleware<RealIpMiddleware>();           // 3. Restore real client IP
app.UseRateLimiter();                             // 4. Rate limit by real IP
app.UseAuthentication();                          // 5. JWT validation
app.UseAuthorization();                           // 6. Policy enforcement
```

## RecovererMiddleware

```csharp
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

### Middleware Rules

| Rule | Why |
|------|-----|
| Never expose exception message to client | Always `"Internal Server Error"` |
| Log full exception with `ILogger` | Stack trace is captured automatically |
| Exclude `OperationCanceledException` | Cancelled request, not a bug |
| `RealIpMiddleware` before `UseRateLimiter` | Rate limiter must see real IP, not proxy IP |

---

# Logging — Serilog via ILogger<T>

## Principle

.NET has `Microsoft.Extensions.Logging.ILogger<T>`. Use it directly — no custom wrapper. **Serilog** plugs in as the provider behind `ILogger<T>`. All code logs to the interface; Serilog handles sinks, enrichment, and formatting.

## Serilog Setup — Program.cs

```csharp
using Serilog;

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((context, config) =>
    config.ReadFrom.Configuration(context.Configuration));

try
{
    var app = builder.Build();
    app.Run();
}
catch (Exception ex) { Log.Fatal(ex, "Application terminated unexpectedly"); }
finally { Log.CloseAndFlush(); }
```

One line: `ReadFrom.Configuration()`. Sink config lives in `appsettings.json`.

## Log Message Convention

- **`[Type.Method]` prefix**: `[OrderService.PlaceOrder]` — greppable.
- **Structured data as named placeholders**: `_logger.LogInformation("Order {OrderId} placed", order.Id)` — not `$"Order {order.Id} placed"`.
- **Never interpolate in the message template**: use `"Order {OrderId}"` with `order.Id` as argument.
- `{@Object}` serializes as JSON: `_logger.LogWarning("Invalid payload {@Payload}", request)`.

## Service Logging — No Serilog Reference

```csharp
internal sealed class OrderService : IOrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger) => _logger = logger;

    public async Task<Result<OrderDto>> PlaceOrderAsync(PlaceOrderInput input, CancellationToken ct)
    {
        _logger.LogInformation("[OrderService.PlaceOrder] Placing order for user {UserId} with {ItemCount} items",
            input.UserId, input.Items.Count);
        // ...
    }
}
```

## DB Sink — Capture WARN+ERROR to Channel

`AlertSink` pushes WARN+ERROR to a bounded channel. `AlertWorker` drains it and INSERTs into `system_alerts`. Non-blocking — `TryWrite` drops when full.

### Wiring

```csharp
// SharedKernel/DependencyInjection.cs
var channel = Channel.CreateBounded<AlertEntity>(new BoundedChannelOptions(1000)
{
    FullMode = BoundedChannelFullMode.DropWrite
});

Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(config)
    .WriteTo.Sink(new AlertSink(channel.Writer))
    .CreateLogger();

services.AddSingleton(channel.Reader);
services.AddSingleton(channel.Writer);
services.AddHostedService<AlertWorker>();
```

### Rules

| Rule | Why |
|------|-----|
| `builder.Host.UseSerilog()` not `AddSerilog()` | Replaces `ILoggerFactory` provider. `AddSerilog()` stacks on top. |
| `.Enrich.WithExceptionDetails()` | Stack traces + inner exceptions as structured JSON |
| `LogContext.PushProperty()` per request | CorrelationId, UserId — every log inherits them |
| Override noisy namespaces to `Warning` | `Microsoft.AspNetCore`, `HttpClient` — suppress |
| `Log.CloseAndFlush()` in `finally` | Drains pending log events before exit |
| DB sink: bounded channel + background worker | Logging never blocks business flow |

---

# Database

## Connection Factory — Single or Multiple

```csharp
namespace SharedKernel;

public interface ISqlConnectionFactory { SqlConnection CreateConnection(); }

internal sealed class SqlConnectionFactory : ISqlConnectionFactory
{
    private readonly string _connectionString;
    public SqlConnectionFactory(string connectionString) => _connectionString = connectionString;
    public SqlConnection CreateConnection() => new(_connectionString);
}
```

Multiple databases via keyed DI:

```csharp
builder.Services.AddKeyedSingleton<ISqlConnectionFactory>(
    DatabaseKeys.Primary, new SqlConnectionFactory(config.GetConnectionString("Primary")!));
builder.Services.AddKeyedSingleton<ISqlConnectionFactory>(
    DatabaseKeys.ReadReplica, new SqlConnectionFactory(config.GetConnectionString("ReadReplica")!));
```

## Unit of Work

```csharp
namespace SharedKernel;

public interface IUnitOfWork
{
    Task ExecuteAsync(Func<SqlConnection, SqlTransaction, Task> operation, CancellationToken ct = default);
    Task<T> ExecuteAsync<T>(Func<SqlConnection, SqlTransaction, Task<T>> operation, CancellationToken ct = default);
}
```

Service writes business data + outbox message in one transaction:

```csharp
await _uow.ExecuteAsync(async (conn, tx) =>
{
    await _repo.InsertOrderAsync(conn, tx, entity);
    var msg = OutboxMessage.Create("order.placed", new OrderPlacedPayload(entity), _clock);
    await _outboxRepo.InsertAsync(conn, tx, msg);
});
```

### Dapper Rules

| Rule | Why |
|------|-----|
| Repository receives `SqlConnection` + `SqlTransaction?` | Never owns the connection |
| No `using` on connection | UoW or caller owns the lifetime |
| Raw SQL. No query builder. No ORM attributes. | Dapper maps by name — `MatchNamesWithUnderscores` |
| Column names match property names | No `[Column]` attributes needed |

---

# Cross-Module Communication

## Allowed Mechanisms

| Mechanism | When | How |
|-----------|------|-----|
| **Direct call via `Contracts/` interface** | Synchronous — query user data from order module | Inject `IUserService` |
| **In-process EventBus** | Fire-and-forget — "order placed → update loyalty points" | `IEventBus.PublishAsync<T>()` |
| **Outbox** | Durable — "order placed → send confirmation email" | `OutboxMessage` in same transaction |

## Forbidden Mechanisms

| Mechanism | Why |
|-----------|-----|
| SQL joins across module tables | Breaks data ownership |
| Importing `Infrastructure/` from another module | Bypasses contracts |
| Sharing entity types | Couples schemas |
| Module-to-module HTTP calls in-process | Needless serialization overhead |

---

# Outbox Pattern

Write business data + outbox message in one DB transaction. A background worker polls and dispatches to handlers.

## Entity

```csharp
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
        => new() { Id = Guid.CreateVersion7(), Type = type, Payload = JsonSerializer.Serialize(payload), CreatedAt = clock.UtcNow };
}
```

## Handler Registration — Safe Multi-Module

Use `TryAddEnumerable` so multiple modules register handlers without overwriting each other:

```csharp
// Order/Module.cs — TryAddEnumerable appends, doesn't replace
services.TryAddEnumerable(ServiceDescriptor.Transient<IOutboxHandler, OrderPlacedHandler>());

// Notification/Module.cs — another handler, same interface
services.TryAddEnumerable(ServiceDescriptor.Transient<IOutboxHandler, MailSentHandler>());
```

### Outbox Rules

| Rule | Why |
|------|-----|
| `OutboxMessage` in same transaction as business write | Atomic — never lose an event |
| `IOutboxHandler` registered via `TryAddEnumerable` | Prevents last-write-wins: multiple modules, same interface |
| Bounded retry with `MaxRetries` | Prevents infinite loops on permanent failures |
| `ProcessedAt` timestamp | Observability — detect stuck messages |

---

# Background Workers — `BackgroundService`

```csharp
internal sealed class OrderReaperWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await using var scope = _scopeFactory.CreateAsyncScope();
                var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
                // ... work
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _logger.LogError(ex, "[OrderReaperWorker] Cycle failed");
            }
            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }
}
```

**Rules:** Use `IServiceScopeFactory` (workers are singletons; scoped services need a scope per tick). Catch all exceptions — one failed cycle must not kill the worker. Respect `CancellationToken`.

---

# Configuration

## Single Source — appsettings.json

```json
{
  "Serilog": { "MinimumLevel": { "Default": "Information" } },
  "ConnectionStrings": { "Primary": "Server=...", "ReadReplica": "Server=..." },
  "Outbox": { "PollIntervalSeconds": 5, "BatchSize": 20 }
}
```

## Strongly-Typed Options Classes

```csharp
public sealed class AppConfig
{
    public const string Section = "App";
    public string JwtSecret { get; init; } = string.Empty;
}

// Registration
builder.Services.Configure<AppConfig>(builder.Configuration.GetSection(AppConfig.Section));
```

Modules access via `IOptions<T>` — never call `Configuration["key"]` directly.

---

# Package Dependencies (NuGet)

| Package | Purpose |
|---------|---------|
| `Dapper` | Micro-ORM, SQL query/execute |
| `Microsoft.Data.SqlClient` | SQL Server ADO.NET driver |
| `FluentValidation` | Request validation |
| `FluentValidation.DependencyInjectionExtensions` | Auto-register validators from assembly |
| `MailKit` | SMTP email sending |
| `Serilog` | Structured logging core |
| `Serilog.Extensions.Logging` | Bridge `ILogger<T>` → Serilog |
| `Serilog.Sinks.Console` | Console sink |
| `Serilog.Sinks.File` | Rolling file sink |
| `Serilog.Enrichers.Environment` | Machine name, process ID enrichment |
| `Serilog.Enrichers.Thread` | Thread ID enrichment |
| `Serilog.Exceptions` | Rich exception details |
| `Microsoft.AspNetCore.OpenApi` | Swagger/OpenAPI (.NET 10 built-in) |
| `Microsoft.AspNetCore.Authentication.JwtBearer` | JWT auth |

## Framework-Provided (No NuGet Needed)

`System.Text.Json`, `Microsoft.Extensions.DependencyInjection`, `Microsoft.Extensions.Configuration`, `Microsoft.Extensions.Options`, `System.Threading.Channels`.

---

# Full Project Structure

```
src/
  Program.cs                           # Host builder, middleware, module registration
  appsettings.json
  appsettings.Development.json

  Shared/
    Models/
      ApiResponse.cs                   #   Envelope + ForSuccess/ForFailure factories
      Pagination.cs                    #   PaginationParams, PageInfo
    Results/
      Result.cs                        #   Result<T> discriminated union
      Outcome.cs                       #   Outcome<T> record + Outcome static factory
      Failure.cs                       #   Failure record with static factories
    Extensions/
      ResultExtensions.cs              #   ToResult(), ToError(), ResponseResult (IResult)
    Filters/
      ValidationFilter.cs              #   .Validate<T>() endpoint filter
    Middleware/
      ExceptionHandlerMiddleware.cs    #   Global catch-all → Failure.Internal()

  Modules/
    Order/
      Contracts/                       # IOrderService, IOrderRepository
      Domain/                          # OrderEntity, OrderDto, Constants
      Application/                     # OrderService
      Infrastructure/                  # OrderRepository, OrderPlacedHandler
      Api/                             # OrderEndpoints, PlaceOrderRequest, OrderResponse
      Module.cs                        # AddOrderModule()
    Payment/                           # same structure
    User/                              # same structure
```

Every module has the same shape. Adding a module = copying the folder structure + writing domain logic.

---

# Adding a New Module Checklist

1. [ ] Create `Modules/<Name>/` folder (PascalCase, singular, business capability).
2. [ ] Define `Contracts/` — public service interface, internal repository interface.
3. [ ] Define `Domain/` — entity, DTO, Constants, Errors, Events.
4. [ ] Define `Application/` — service implementation. Depends on Contracts + Domain only.
5. [ ] Define `Infrastructure/` — Dapper repository, outbox handlers, email senders.
6. [ ] Define `Api/` — Minimal API endpoints, request DTOs + FluentValidation, response DTOs.
7. [ ] Define `Module.cs` — `AddXxxModule(this IServiceCollection, IConfiguration)` with dependency comment.
8. [ ] Register in `Program.cs` — `services.AddXxxModule(config)` + `app.MapXxxEndpoints()`.
9. [ ] Verify: `Program.cs` has zero knowledge of module internals — no handler names, no connection strings, no route paths.

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
| Skip `Module.cs` — wire from `Program.cs` | Scatters DI knowledge | Each module has `AddXxxModule()` |
| Skip `Contracts/` — export from random files | No contract, boundaries rot | One folder per module for public API |

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
6. **`Result<T>` for all service methods that can fail** — no exceptions for expected errors.
7. **FluentValidation on every request** — no if-checks in the service.
8. **Dapper with raw SQL** — no query builder, no ORM.
9. **No shared database tables between modules** — each module copies what it needs, or calls the owning module's API.
10. When in doubt, ask: "If I extracted this module into its own service tomorrow, what would break?"
