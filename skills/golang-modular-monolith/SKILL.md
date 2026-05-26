---
name: golang-modular-monolith
description: Enforce modular monolith architecture in Go — strict module boundaries, per-module DI, dependency direction, shared kernel, and anti-corruption patterns.
version: 2.0.0
author: User
---

# Go Modular Monolith Architecture

## Purpose

Enforce modular monolith architecture in Go projects. Every piece of code lives inside a module. Every module owns its domain. Cross-module communication follows strict rules. The goal: monolith deployment with microservice discipline — replaceable modules, no spaghetti.

---

# Core Principle

> A module is a self-contained business capability. It owns its data, its logic, and its public contract. No other module may reach into its internals.

Every module must be extractable into its own service **without rewriting business logic**. If you can't extract it cleanly, the boundary is broken.

---

# Go Naming Conventions

Follow [Effective Go](https://go.dev/doc/effective_go) and [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments). These rules apply to every `.go` file in the project — modules, shared kernel, `cmd/`, everything.

## Exported vs Unexported

Capitalization controls visibility. No `public`/`private` keywords.

- **Exported** (PascalCase): accessible outside the package.
- **Unexported** (camelCase): package-private.

This governs every identifier: types, functions, methods, variables, constants, struct fields, interface methods.

## Package and Directory Names

- **Lowercase only** — no underscores, no capitals, no mixed case.
- **Short** — one word preferred; two concatenated if unavoidable (`nethttp`, not `net_http`).
- **Singular** — `user`, not `users` (except plural-only concepts like `strings`, `bytes`, `errors`).
- **Not** `util`, `common`, `misc`, `helpers`, `base`, `core` — meaningless name magnets.
- **Avoid stutter** — `http.Server`, not `httpserver.Server` or `http.HTTPServer`.
- Directory name must match package name.
- Module directories follow the same rules (module = Go package): `user`, `order`, `payment`, `orderfulfillment`.

| Bad | Good |
|-----|------|
| `package util` | `package user` |
| `package common` | `package parser` |
| `package HTTPClient` | `package httpclient` |
| `package my_package` | `package mypackage` |
| `package controllers` | `package controller` |

## File Names

- **snake_case** — lowercase with underscores between words: `user_service.go`, `order_repository.go`.
- Describe the primary concern in the file.
- Test files: `*_test.go`. Platform suffixes: `*_linux.go`, `*_windows.go`.
- Generated files: `*_string.go`, `*.pb.go`.
- File name must describe what's inside — `handler.go`, not `stuff.go`.

## Variables

- **Unexported**: camelCase — `maxRetries`, `defaultTimeout`, `bufferSize`.
- **Exported**: PascalCase — `ErrNotFound`, `DefaultClient`.
- **Short scope = short name**. Loop indices: `i`, `j`, `k`. Single-use: `b` for bytes, `n` for count.
- **Longer scope = descriptive name**. Package-level vars earn a meaningful name.
- **Acronyms**: all-caps (exported) or all-lowercase (unexported) — never mixed. `UserID`, `userID`, `HTTPServer`, `httpServer`. Never `UserId` or `HttpServer`. Common acronyms: `ID`, `URL`, `HTTP`, `JSON`, `SQL`, `API`, `gRPC`, `TLS`, `UUID`, `JWT`, `HTML`, `CSV`, `XML`, `SMTP`, `SSH`, `DB`, `OS`.
- **Arrays/slices**: plural noun — `users`, `orderIDs`. Not `userList` or `itemsArray`.

```go
var maxRetries = 3
var httpClient *http.Client
var ErrNotFound = errors.New("not found")
var userID string   // unexported — all-lower acronym
var UserID string   // exported — all-caps acronym
```

## Constants

- Same casing as variables: camelCase unexported, PascalCase exported.
- **No** `SCREAMING_SNAKE_CASE` or `CONSTANT_CASE`. Go does not use this style.
- Build-time/env constants use PascalCase: `DefaultPort`, `MaxConnections`.
- Group related constants with `iota`.

```go
const (
    defaultPort    = 8080
    maxConnections = 100
)

const (
    StatusActive   = "active"
    StatusInactive = "inactive"
)

const (
    TypeJSON Type = iota + 1
    TypeXML
    TypeProtobuf
)
```

## Functions and Methods

- **camelCase** unexported, **PascalCase** exported.
- **No `Get` prefix** for simple getters. `Owner()` not `GetOwner()`. Exception: when the getter does I/O or remote work (`GetUserFromDB`).
- **`Set` prefix** allowed for setters: `SetOwner`, `SetDeadline`.
- **Constructors**: `New` + type name for exported (`NewClient`, `NewServer`); `new` + type name for unexported (`newParser`).
- **Boolean-returning**: descriptive names — `IsEmpty`, `HasPrefix`, `Contains`, `Closed()`. Don't force `Is` on every bool.

```go
func NewClient(addr string) *Client { ... }
func (c *Client) Do(req *Request) (*Response, error) { ... }
func newParser(r io.Reader) *parser { ... }
func (u *User) Name() string { return u.name }     // no Get prefix
func (u *User) SetName(name string) { u.name = name }
func (f *File) IsDir() bool { ... }
func (c *Conn) Closed() bool { ... }
```

## Types

- **PascalCase** exported: `Client`, `Server`, `ResponseWriter`.
- **camelCase** unexported: `parser`, `bufferPool`.
- **Single-method interface**: method name + `-er` suffix: `Reader`, `Writer`, `Closer`, `Stringer`.
- **Multi-method interface**: descriptive noun: `Handler`, `ReadWriter`.
- **No** `I` prefix for interfaces. **No** `Interface` suffix. `Reader`, not `IReader` or `ReaderInterface`.
- Type aliases: same casing as underlying type.

```go
type Client struct { ... }
type Reader interface { Read(p []byte) (n int, err error) }
type ReadWriter interface { Reader; Writer }
type parser struct { ... }
```

## Struct Fields

- **camelCase** unexported, **PascalCase** exported.
- Same acronym rules as variables.
- Field names read naturally with their type.

```go
type User struct {
    ID        string    // exported — PascalCase
    Email     string
    CreatedAt time.Time
    deleted   bool      // unexported — camelCase
    userID    string    // unexported, acronym all-lowercase
}
```

## Receivers

- **One or two letters**, short and consistent.
- Same receiver name across all methods of a type.
- First letter (lowercase) of the type, or first two for readability.
- **No** `this`, `self`, or `me`.

```go
func (c *Client) Do(req *Request) (*Response, error) { ... }
func (c *Client) Close() error { ... }
func (rw *ResponseWriter) Write(b []byte) (int, error) { ... }
```

## Errors

- **Sentinel errors**: `Err` + descriptive noun: `ErrNotFound`, `ErrTimeout`, `ErrClosed`.
- **Error types**: descriptive noun + `Error`: `ParseError`, `UnmarshalTypeError`.
- **Error messages**: lowercase, no trailing punctuation. `fmt.Errorf("user %q not found", id)` not `fmt.Errorf("User %q not found.", id)`.

```go
var ErrNotFound = errors.New("item not found")
var ErrPermission = errors.New("permission denied")

type ParseError struct { Line int; Msg string }
func (e *ParseError) Error() string { ... }

fmt.Errorf("parse config: %w", err)        // wrapping
fmt.Errorf("user %q not found", userID)    // context
```

## Test Functions

- **Test**: `Test` + type/method name: `TestParse`, `TestClient_Do`.
- **Benchmark**: `Benchmark` + name: `BenchmarkParse`.
- **Example**: `Example` + name: `ExampleParse`, `ExampleClient_Do`.
- **Fuzz**: `Fuzz` + name: `FuzzParse`.
- **Sub-behavior**: underscore-separated: `TestParser_Parse_MultipleTokens`.
- Black-box tests: `_test` package suffix in same directory.

```go
func TestParse(t *testing.T) { ... }
func TestClient_Do(t *testing.T) { ... }
func BenchmarkParse(b *testing.B) { ... }
```

## Quick Reference — Acronyms

| Acronym | Export Example | Unexported Example |
|---------|---------------|-------------------|
| ID | `UserID` | `userID` |
| URL | `ParseURL` | `baseURL` |
| HTTP | `HTTPClient` | `httpClient` |
| JSON | `ToJSON` | `toJSON` |
| SQL | `SQLDB` | `sqlDB` |
| API | `APIKey` | `apiKey` |
| gRPC | `GRPCServer` | `grpcServer` |
| UUID | `NewUUID` | `newUUID` |
| JWT | `JWTSigner` | `jwtSigner` |

---

# Module Definition

## What Makes a Module

A module is a directory at the project root (or `modules/` root) that represents a **business capability**, not a technical layer.

```
project/
  modules/
    user/           # ✅ business capability
    order/          # ✅ business capability
    payment/        # ✅ business capability
    notification/   # ✅ business capability
    shared/         # ✅ shared kernel (see below)
```

**Module = business capability.** Not a layer. Not a utility drawer.

```
  modules/
    handlers/       # ❌ technical layer, not a module
    models/         # ❌ technical layer, not a module
    services/       # ❌ technical layer, not a module
    database/       # ❌ technical layer, not a module
```

## Module Name Rules

Module directories follow [Package and Directory Names](#package-and-directory-names):
- **Singular** noun describing the business capability: `user`, `order`, `payment`, `catalog`.
- One word preferred; two concatenated if needed: `orderfulfillment`, not `order_fulfillment`.
- Package name must match directory name.

---

# Module Internal Structure

## Ports & Adapters (Hexagonal Architecture)

Every module follows ports & adapters. Three packages, strict dependency direction. Go-idiomatic naming — no frameworks, no code generation, just packages.

```
modules/order/
  ports/              # Interfaces — the module's contracts
  adapters/           # Implementations of those contracts
    http/             # Primary adapter — HTTP handlers, routes, request/response types
    postgres/         # Secondary adapter — DB repository
  domain/             # Pure domain — entities, DTOs, errors, events. Zero dependencies.
  module.go           # DI wiring — connects ports to adapters
```

## Dependency Direction

```
adapters → ports → domain
```

- `domain/` depends on **nothing in the module** — only `shared/` and stdlib
- `ports/` depends on `domain/` (uses entity, DTO, error types in interface signatures)
- `adapters/http/` depends on `ports/` (implements service interface)
- `adapters/postgres/` depends on `ports/` (implements repository interface)
- `adapters/` never depend on each other
- `module.go` depends on everything — it's the composition root for this module

## Package-by-Package

### `domain/` — Pure Business. No Dependencies.

Entities, DTOs, domain errors, events. No interfaces. No database imports. No HTTP imports. Pure Go structs and behavior.

```
modules/order/domain/
  entity.go           # orderEntity, orderItemEntity — DB tags OK (gorm/sql)
  dto.go              # orderDTO, orderItemDTO — no tags, plain structs
  errors.go           # orderError, InsufficientStockDetails — DomainError impl
  events.go           # OrderPlaced, OrderCancelled — domain events
```

```go
// modules/order/domain/entity.go
package domain

import "time"

type orderEntity struct {
    ID        string    `db:"id"`
    UserID    string    `db:"user_id"`
    Status    string    `db:"status"`
    Total     int64     `db:"total"` // cents
    CreatedAt time.Time `db:"created_at"`
}

type orderItemEntity struct {
    ID        string `db:"id"`
    OrderID   string `db:"order_id"`
    ProductID string `db:"product_id"`
    Quantity  int    `db:"quantity"`
    UnitPrice int64  `db:"unit_price"` // cents
}
```

```go
// modules/order/domain/dto.go
package domain

type orderDTO struct {
    id     string
    userID string
    status orderStatus
    items  []orderItemDTO
    total  shared.Money
}

type orderItemDTO struct {
    productID string
    quantity  int
    unitPrice shared.Money
}
```

### `ports/` — Contracts. What the Module Promises and Needs.

Interfaces only. The module's public surface. Depends only on `domain/` for types.

```
modules/order/ports/
  service.go          # Service — the module's public API (inbound port)
  repository.go       # Repository — what this module needs from its DB (outbound port)
```

```go
// modules/order/ports/service.go
package ports

import (
    "context"
    "project/modules/order/domain"
)

// Service is the module's inbound port. Adapters/http implements it.
// Other modules call this via the container.
type Service interface {
    PlaceOrder(ctx context.Context, input domain.orderDTO) (domain.orderDTO, error)
    GetOrder(ctx context.Context, orderID string) (domain.orderDTO, error)
    CancelOrder(ctx context.Context, orderID string) (domain.orderDTO, error)
}
```

```go
// modules/order/ports/service.go
package ports

// Key is the container key for this module's service.
// Other modules use this constant — no magic strings.
const Key = "order.ports"
```

```go
// modules/order/ports/repository.go
package ports

import (
    "context"
    "github.com/jmoiron/sqlx"
    "project/modules/order/domain"
)

// Repository is the module's outbound port. Adapters/postgres implements it.
type Repository interface {
    GetByID(ctx context.Context, tx *sqlx.Tx, id string) (domain.orderEntity, error)
    GetByUserID(ctx context.Context, tx *sqlx.Tx, userID string) ([]domain.orderEntity, error)
    InsertOrder(ctx context.Context, tx *sqlx.Tx, e domain.orderEntity) error
    InsertItems(ctx context.Context, tx *sqlx.Tx, items []domain.orderItemEntity) error
    UpdateStatus(ctx context.Context, tx *sqlx.Tx, id string, status string) error
}
```

### `adapters/http/` — Primary Adapter. Drives the Module.

Handlers, request/response types, route registration. Implements nothing in code — it calls `ports.Service`. Depends on `ports/`.

```
modules/order/adapters/http/
  handler.go          # OrderHandler — embeds httputil.BaseHandler
  request.go          # CreateOrderRequest, CancelOrderRequest — json + validate tags
  response.go         # OrderResponse, OrderItemResponse — json tags
  routes.go           # RegisterRoutes(r chi.Router, h *OrderHandler)
```

```go
// modules/order/adapters/http/handler.go
package http

import (
    "project/modules/order/ports"
    "project/modules/shared/httputil"
)

type OrderHandler struct {
    httputil.BaseHandler
    service ports.Service
}

func NewHandler(service ports.Service, log logger.Logger) *OrderHandler {
    return &OrderHandler{
        BaseHandler: httputil.BaseHandler{Log: log},
        service:     service,
    }
}

func (h *OrderHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        h.BadRequest(w, r, "Invalid request body")
        return
    }
    if errs := req.Validate(); len(errs) > 0 {
        h.ValidationError(w, r, "Validation failed", errs)
        return
    }

    order, err := h.service.PlaceOrder(r.Context(), req.toDTO())
    if err != nil {
        var domainErr shared.DomainError
        if errors.As(err, &domainErr) {
            h.BadRequest(w, r, err.Error())
            return
        }
        h.Error(r.Context(), "create order", err)
        h.InternalError(w, r)
        return
    }

    h.JSON(w, http.StatusCreated, NewCreatedResponse("Order created", fromDTO(order)))
}
```

### `adapters/postgres/` — Secondary Adapter. Driven by the Module.

Implements `ports.Repository`. Just SQL queries. Depends on `ports/` and `domain/`.

**Why `postgres/` and not `data/`**: The purpose of this package is to satisfy a specific port contract (`ports.Repository`) using PostgreSQL. Calling it `data/` obscures the implementation ("what kind of data? Flat file? In-memory map? Redis?"). `postgres/` is self-documenting — the reader knows exactly what backs the port. It also leaves room for a second implementation: when the module later needs a `redis/` adapter (caching, session store, rate-limit counters), it slots in as another secondary adapter at the same level with zero renaming.

```
modules/order/adapters/postgres/
  repository.go       # struct implementing ports.Repository
```

```go
// modules/order/adapters/postgres/repository.go
package postgres

import (
    "context"
    "github.com/jmoiron/sqlx"
    "project/modules/order/domain"
    "project/modules/order/ports"
)

type repository struct {
    db *sqlx.DB
}

func NewRepository(db *sqlx.DB) ports.Repository {
    return &repository{db: db}
}

func (r *repository) GetByID(ctx context.Context, tx *sqlx.Tx, id string) (domain.orderEntity, error) {
    var e domain.orderEntity
    err := tx.GetContext(ctx, &e, "SELECT * FROM orders WHERE id = ?", id)
    return e, err
}

func (r *repository) InsertOrder(ctx context.Context, tx *sqlx.Tx, e domain.orderEntity) error {
    _, err := tx.NamedExecContext(ctx,
        "INSERT INTO orders (id, user_id, status, total, created_at) VALUES (:id, :user_id, :status, :total, :created_at)", e)
    return err
}
```

### `module.go` — Composition Root

Wires ports to adapters. One exported function: `Register`. This is the only file that knows concrete implementations.

```go
// modules/order/module.go
package order

import (
    "project/modules/order/adapters/http"
    "project/modules/order/adapters/postgres"
    "project/modules/order/ports"
    "project/modules/shared"
    "project/modules/shared/db"
)

// Register wires the order module.
// Dependencies: shared, user.ports, payment.ports
// Provides: order.ports.Service
func Register(c *shared.Container, r chi.Router) {
    // Resolve dependencies by exported key constants. No magic strings.
    uow := db.NewUnitOfWork(c.DB)
    repo := postgres.NewRepository(c.DB)
    userAPI := shared.MustGet[userPorts.Service](c, userPorts.Key)
    paymentAPI := shared.MustGet[paymentPorts.Service](c, paymentPorts.Key)

    svc := newService(repo, uow, userAPI, paymentAPI, c.EventBus)

    c.Set(ports.Key, svc)
    c.EventBus.Subscribe(payment.PaymentCompleted{}, svc.handlePaymentCompleted)

    // Mount routes
    handler := httpAdapter.NewHandler(svc, c.Logger)
    httpAdapter.RegisterRoutes(r, handler)
}
```

## Full Module Layout

```
modules/order/
  ports/
    service.go          # Service interface — inbound port
    repository.go       # Repository interface — outbound port
  adapters/
    http/
      handler.go        # OrderHandler
      request.go        # CreateOrderRequest, CancelOrderRequest
      response.go       # OrderResponse, OrderItemResponse
      routes.go         # RegisterRoutes(r, h)
    postgres/
      repository.go     # SQL implementation of ports.Repository
  domain/
    entity.go           # orderEntity, orderItemEntity
    dto.go              # orderDTO, orderItemDTO
    errors.go           # Domain errors
    events.go           # OrderPlaced, OrderCancelled
  module.go             # Register(c) — wires ports → adapters
```

## Type Separation

| File | Package | Contains | Visibility |
|------|--------|---------|-----------|
| `domain/entity.go` | `domain` | DB-mapped structs with gorm/sql tags | unexported |
| `domain/dto.go` | `domain` | Internal transfer structs, no tags | unexported |
| `domain/errors.go` | `domain` | Error types + sentinels | exported |
| `domain/events.go` | `domain` | Event structs | exported |
| `ports/service.go` | `ports` | Service interface | exported |
| `ports/repository.go` | `ports` | Repository interface | exported |
| `adapters/http/request.go` | `http` | API input types, json+validate tags | exported |
| `adapters/http/response.go` | `http` | API output types, json tags | exported |
| `adapters/http/handler.go` | `http` | HTTP handler struct + methods | unexported struct, exported constructor |
| `adapters/http/routes.go` | `http` | Route registration | exported RegisterRoutes |
| `adapters/postgres/repository.go` | `postgres` | SQL queries | unexported struct, exported constructor |
| `module.go` | `order` | DI wiring | exported Register |

## Multi-Instance Adapters — Same Interface, Different Configuration

When a module needs the same adapter type wired with different configs (two PostgreSQL databases, two SMTP servers, two FTP clients, two Redis clusters), the interface is identical. Only the configuration differs. Don't duplicate code — instantiate the same adapter struct twice with different params.

### The Problem

```go
// One SMTP — simple
mailer := mailer.NewMailer(cfg.SMTPHost, cfg.SMTPPort, cfg.SMTPUsername, cfg.SMTPPassword)

// Two SMTPs — transactional email vs marketing email
// Same code, different credentials. How do you distinguish?
```

### Strategy: Labeled Constructor + Named Instances

Give each instance a **name**. The name is a string that identifies the instance for logging and DI registration. The adapter struct doesn't change — it just carries its name for identification.

```go
// modules/shared/mailer/mailer.go
type Mailer interface {
    Send(to []string, subject, body string) error
}

type smtpMailer struct {
    name     string   // "transactional", "marketing"
    host     string
    port     int
    username string
    password string
}

func New(name, host string, port int, username, password string) Mailer {
    return &smtpMailer{
        name:     name,
        host:     host,
        port:     port,
        username: username,
        password: password,
    }
}

func (m *smtpMailer) Send(to []string, subject, body string) error {
    // log with m.name so you know which instance sent it
    return nil
}
```

The `name` field is for **observability only** — logs, traces, metrics. It does not affect behavior. The interface is the same.

### DI: Container Keys With Qualifier Suffix

```go
// modules/shared/mailer/keys.go
package mailer

const (
    KeyTransactional = "mailer.transactional"
    KeyMarketing     = "mailer.marketing"
)
```

Two keys, same type, different instances. Registered at app startup:

```go
// internal/app/builder.go
func (b *builder) setupMailers() {
    b.transactionalMailer = mailer.New(
        "transactional",
        b.cfg.SMTPHost,
        b.cfg.SMTPPort,
        b.cfg.SMTPTransactionalUsername,
        b.cfg.SMTPTransactionalPassword,
    )
    b.marketingMailer = mailer.New(
        "marketing",
        b.cfg.SMTPHost,
        b.cfg.SMTPPort,
        b.cfg.SMTPMarketingUsername,
        b.cfg.SMTPMarketingPassword,
    )

    c.Set(mailer.KeyTransactional, b.transactionalMailer)
    c.Set(mailer.KeyMarketing, b.marketingMailer)
}
```

Modules pull the one they need:

```go
// modules/auth/module.go
func Register(c *shared.Container, r chi.Router) {
    transactionalMailer := shared.MustGet[mailer.Mailer](c, mailer.KeyTransactional)
    svc := newService(repo, uow, transactionalMailer, ...)
}

// modules/marketing/module.go
func Register(c *shared.Container, r chi.Router) {
    marketingMailer := shared.MustGet[mailer.Mailer](c, mailer.KeyMarketing)
    svc := newService(repo, marketingMailer, ...)
}
```

### Generic Pattern: `Instanced[T]` Wrapper

For avoiding the `name string` field in every adapter, use a thin generic wrapper:

```go
// modules/shared/container.go
type Instanced[T any] struct {
    Name     string
    Instance T
}
```

Each instance is explicitly wrapped with its identity. Logging libraries format this naturally as struct fields.

### Configuration: Flat Namespacing

```go
// modules/shared/config/config.go
type Config struct {
    // SMTP
    SMTPHost                   string
    SMTPPort                   int
    SMTPTransactionalUsername  string  // by function
    SMTPTransactionalPassword  string
    SMTPMarketingUsername      string
    SMTPMarketingPassword      string

    // FTP
    FTPExportHost       string
    FTPExportPort       int
    FTPExportUsername    string
    FTPExportPassword    string
    FTPImportHost       string
    FTPImportPort       int
    FTPImportUsername    string
    FTPImportPassword    string
}
```

Naming convention: `<PROTOCOL><Role><Attribute>`. The role (`Transactional`, `Marketing`, `Export`, `Import`) is the discriminator. Don't nest into sub-structs until you have 4+ instances of the same thing.

### Naming Convention for Keys

```
<type>.<role>
```

| Key | Type | Instance Role |
|-----|------|---------------|
| `mailer.transactional` | `mailer.Mailer` | Password resets, verification emails, receipts |
| `mailer.marketing` | `mailer.Mailer` | Newsletters, promotions, drip campaigns |
| `ftp.export` | `ftp.Client` | Export reports, data dumps |
| `ftp.import` | `ftp.Client` | Ingest upstream file feeds |
| `redis.cache` | `redis.Client` | Application cache |
| `redis.sessions` | `redis.Client` | Session store |
| `db.primary` | `*sqlx.DB` | Read-write |
| `db.replica` | `*sqlx.DB` | Read-only reporting |

### Two Separate Databases — Different Systems, Different Schemas

When the app connects to two completely different databases (e.g., main app DB + legacy system, or transactional DB + analytics warehouse), each gets its own connection pool, its own container key, and its own repository. They share nothing — different tables, different queries, different domains.

**Config — two full URLs:**

```go
// modules/shared/platform/config/config.go
type Config struct {
    DatabaseURL   string     // database A
    DatabaseBURL   string     // database B
    DatabasePool  PoolConfig // shared pool settings for both
    // ...
}
```

Name by what the database holds. Not `DatabaseURL1` / `DatabaseURL2`.

**Connection setup — two `*sqlx.DB` with named keys:**

```go
// modules/shared/container_keys.go
const (
    KeyDBA = "db.a"   // database A
    KeyDBB = "db.b"   // database B
)

// internal/app/builder.go
func (b *builder) setupDB() error {
    var err error
    b.dbA, err = db.Connect(b.cfg.DatabaseURL, poolCfg)
    if err != nil {
        return fmt.Errorf("connect db A: %w", err)
    }
    b.dbB, err = db.Connect(b.cfg.DatabaseBURL, poolCfg)
    if err != nil {
        return fmt.Errorf("connect db B: %w", err)
    }
    return nil
}

func (b *builder) registerInfra(c *shared.Container) {
    c.Set(shared.KeyDBA, b.dbA)
    c.Set(shared.KeyDBB, b.dbB)
}
```

**Module pulls the DB it needs:**

```go
// modules/order/module.go — needs only database A
func Register(c *shared.Container, r chi.Router) {
    dbA := shared.MustGet[*sqlx.DB](c, shared.KeyDBA)
    repo := postgres.NewRepository(dbA)
    uow := db.NewUnitOfWork(dbA)
    // ...
}

// modules/report/module.go — needs only database B
func Register(c *shared.Container, r chi.Router) {
    dbB := shared.MustGet[*sqlx.DB](c, shared.KeyDBB)
    repo := postgres.NewReportRepository(dbB)
    // ...
}
```

**Module that needs both — pulls both keys, two separate repos:**

```go
// modules/sync/module.go — reads from A, writes to B
func Register(c *shared.Container, r chi.Router) {
    dbA := shared.MustGet[*sqlx.DB](c, shared.KeyDBA)
    dbB := shared.MustGet[*sqlx.DB](c, shared.KeyDBB)

    sourceRepo := postgres.NewSourceRepository(dbA)  // queries DB A
    targetRepo := postgres.NewTargetRepository(dbB)  // queries DB B

    svc := newService(sourceRepo, targetRepo, outboxRepo)
    // ...
}
```

**Repository — one struct per database:**

```go
// modules/sync/adapters/postgres/source_repository.go
type sourceRepository struct {
    db *sqlx.DB  // database A
}
func NewSourceRepository(db *sqlx.DB) *sourceRepository { return &sourceRepository{db: db} }

// modules/sync/adapters/postgres/target_repository.go
type targetRepository struct {
    db *sqlx.DB  // database B
}
func NewTargetRepository(db *sqlx.DB) *targetRepository { return &targetRepository{db: db} }
```

Each repository struct wraps one `*sqlx.DB`. Never pass both DB handles into one repository struct — that's a god-object that joins across databases. Two repos, two concerns, both injected in `module.go`.

**Unit of Work — one per database:**

```go
func Register(c *shared.Container, r chi.Router) {
    dbA := shared.MustGet[*sqlx.DB](c, shared.KeyDBA)
    dbB := shared.MustGet[*sqlx.DB](c, shared.KeyDBB)

    uowA := db.NewUnitOfWork(dbA)
    uowB := db.NewUnitOfWork(dbB)

    // Two UoWs = no cross-DB transaction. Use saga/outbox for consistency.
    svc := newService(sourceRepo, targetRepo, uowA, uowB, outboxRepo)
}
```

A UnitOfWork wraps one database handle. If a service method must write to both databases atomically, it cannot use a single transaction — there is no distributed transaction across two independent databases. Use the outbox pattern: write to database A + outbox in one tx, then the outbox worker eventually writes to database B.

**Key naming convention for databases:**

```
db.<name>
```

Name describes what the database holds, not its role:

| Key | Example |
|-----|---------|
| `db.main` | Application data — users, orders, payments |
| `db.legacy` | Old system during migration — read-only, eventually decommissioned |
| `db.analytics` | Analytics warehouse — aggregated events, reports |
| `db.logs` | Standalone logging database |

### Rules

| Rule | Why |
|------|-----|
| Same interface, multiple instances — never duplicate code | Two identical `smtpMailer` structs is a DRY violation |
| Instance name for observability only | Log identity, not behavior. The interface doesn't change. |
| Container key discriminates by role | `mailer.transactional` vs `mailer.marketing` — type-safe + grep-friendly |
| Config flat by convention, not nested struct | Flat until 4+ instances. One struct = one env-var scan for ops. |
| Key pattern: `<type>.<role>` | Consistent. No `mailer_marketing_smtp_instance_final` chaos. |
| Module pulls exactly the instance it needs | Auth module never accidentally sends newsletters. Dependency clarity. |

## Data Flow

```
HTTP request
  → adapters/http/handler.go   (decode json → request.go)
    → adapters/http/request.go  (validate)
      → ports.Service.PlaceOrder()
        → [module.go wired: serviceImpl]
          → domain/dto.go           (business logic on DTOs)
            → ports.Repository.InsertOrder()
              → adapters/postgres/repository.go  (dto → entity → SQL)
                → DB

HTTP response
  ← adapters/http/handler.go   (json encode)
    ← adapters/http/response.go (dto → response struct)
      ← ports.Service (returns dto)
        ← domain/dto.go
          ← ports.Repository (returns dto)
            ← adapters/postgres/repository.go  (SQL → entity → dto)
```

Each boundary converts types. Request→DTO at the handler. Entity→DTO at the repository. No type crosses a port boundary unchanged.

---

# UUID Identifiers

## Principle

Always use UUIDs for primary keys and external identifiers. Never auto-increment integers. UUIDs survive sharding, merging, and extraction. Sequential IDs leak order-of-creation, invite enumeration attacks, and collide on multi-writer systems.

Use `github.com/google/uuid` — the standard Go UUID library.

## UUID V7 — Default for Most IDs

UUID V7 embeds a Unix timestamp in the first 48 bits, followed by random bits. Time-sorted and DB-index-friendly (reduces B-tree fragmentation). Use for all non-sensitive primary keys.

```go
id, err := uuid.NewV7()
```

**Use V7 for:** order IDs, event IDs, outbox message IDs, audit log IDs, payment IDs, notification IDs, internal table primary keys.

## UUID V4 — Sensitive Identifiers

UUID V4 is fully random. No timestamp embedded. An attacker cannot derive creation time or sequence from it. Use when the ID is exposed publicly and you want zero information leakage.

```go
id, err := uuid.NewV4()
```

**Use V4 for:** user IDs, API keys in URLs, session tokens, any identifier visible in public endpoints or query params.

## Why Not Auto-Increment

| Problem | UUID Fix |
|---------|----------|
| Sequential guessing (`/users/1001`, `/users/1002`) | Random/non-sequential — unguessable |
| Collision on DB merge or shard rebalance | Global uniqueness — no coordination |
| ID exhaustion (INT max) | 2^122 space — effectively infinite |
| Leaks insertion order and volume | V7 limits it to time-sort only; V4 leaks nothing |

The only cost is storage (16 bytes vs 8 bytes for BIGINT). That's not a real cost at any scale that uses a modular monolith.

## CHAR(36) Storage in MySQL

In MySQL, store UUIDs as `CHAR(36)`. Do not use `BINARY(16)`.

**Why `CHAR(36)` over `BINARY(16)`:**

- **Directly readable** — `SELECT * FROM users` shows `550e8400-e29b-41d4-a716-446655440000`, not binary garbage. No `HEX()` / `UNHEX()` needed during debugging, audit queries, or manual inspection.
- **No marshal friction** — `BINARY(16)` requires custom `sql.Scanner` / `driver.Valuer` implementation or hex conversion at every query. `CHAR(36)` works natively — the UUID string round-trips with zero conversion.
- **Tooling compatibility** — DB migration tools, backup scripts, ETL jobs, and BI tools understand string UUIDs out of the box. Binary UUIDs need special handling everywhere.
- **The storage cost is negligible** — 36 bytes vs 16 bytes. At 1 million rows, that's 20 MB difference. Not a real cost in a monolith. The debugging time saved exceeds the storage cost many times over.
- **Index performance is fine** — `CHAR(36)` BTREE indexes are well-optimized in modern MySQL. Unless there is a proven bottleneck, don't optimize for imaginary scale.

```sql
-- MySQL
CREATE TABLE orders (
    id CHAR(36) NOT NULL PRIMARY KEY,
    user_id CHAR(36) NOT NULL,
    -- ...
    INDEX idx_user_id (user_id)
);
```

```go
// With CHAR(36) — uuid.UUID string serializes directly
type orderEntity struct {
    ID     uuid.UUID `db:"id"`
    UserID uuid.UUID `db:"user_id"`
}
```

`uuid.UUID.String()` produces the canonical 36-char form (`550e8400-e29b-41d4-a716-446655440000`). sqlx binds it as a string. MySQL stores and reads it as `CHAR(36)`. Zero conversion at the application boundary.

For PostgreSQL, `UUID` native type is the equivalent — it also stores human-readable and the driver handles it natively:

```sql
-- PostgreSQL
CREATE TABLE orders (
    id UUID NOT NULL PRIMARY KEY,
    user_id UUID NOT NULL,
    -- ...
);
```

For MSSQL, use `UNIQUEIDENTIFIER` — maps natively to `uuid.UUID` via the `mssql` driver:

```sql
-- MSSQL
CREATE TABLE orders (
    id UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
    user_id UNIQUEIDENTIFIER NOT NULL,
    -- ...
);
```

## Rules

| Rule | Why |
|------|-----|
| UUID V7 as default for all primary keys | Time-sorted, index-friendly, globally unique |
| UUID V4 for user-facing IDs | Fully random — no timestamp, no sequence, no info leak |
| Never auto-increment integers | Sequential, guessable, collision-prone on merge |
| MySQL: `CHAR(36)` for UUIDs, never `BINARY(16)` | Readable without conversion. Tools work. Debugging doesn't need `HEX()`. |
| PostgreSQL: `UUID` native type | Driver handles natively. Same readability as `CHAR(36)`. |
| MSSQL: `UNIQUEIDENTIFIER` | Maps natively to `uuid.UUID`. |
| `uuid.UUID` type directly in entity structs | Serializes to string automatically via `db` tag |
| `uuid.NewV7()` at creation site | Service or entity constructor generates the ID, not the DB |
| `json:"id"` serializes UUID to hex string | `uuid.UUID` marshals to `"550e8400-e29b-41d4-a716-446655440000"` |

---


---

# Database — Timestamps

## Principle

Every table must have `created_at` and `updated_at`. No exceptions. An entity without timestamps is an unobservable black box. Timestamps enable debugging ("when did this happen?"), auditing ("what changed before the incident?"), and monitoring ("how fast are orders growing?").

## Entities Always Carry Timestamps

```go
// modules/order/domain/entity.go
type orderEntity struct {
    ID        uuid.UUID `db:"id"`
    UserID    uuid.UUID `db:"user_id"`
    Status    string    `db:"status"`
    Total     int64     `db:"total"`
    CreatedAt time.Time `db:"created_at"`
    UpdatedAt time.Time `db:"updated_at"`
}
```

Every entity struct has both. No opt-out. The DB tags match column names `created_at` and `updated_at`.

## Auto-Populate with `BaseEntity` (Shared)

Embed a shared base entity to avoid repeating `created_at`/`updated_at` setter logic:

```go
// modules/shared/entity.go
package shared

import "time"

// BaseEntity provides every entity with CreatedAt and UpdatedAt fields plus
// lifecycle helpers. Embed in entity structs for sqlx tag propagation.
type BaseEntity struct {
    CreatedAt time.Time `db:"created_at"`
    UpdatedAt time.Time `db:"updated_at"`
}

// SetCreate marks both timestamps to time.Now(). Call once before insert.
func (b *BaseEntity) SetCreate() {
    b.CreatedAt = time.Now()
    b.UpdatedAt = b.CreatedAt
}

// SetUpdate bumps UpdatedAt to time.Now(). Call before every update.
func (b *BaseEntity) SetUpdate() {
    b.UpdatedAt = time.Now()
}
```

Module entity embeds it:

```go
// modules/order/domain/entity.go
type orderEntity struct {
    shared.BaseEntity
    ID     uuid.UUID `db:"id"`
    UserID uuid.UUID `db:"user_id"`
    Status string    `db:"status"`
    Total  int64     `db:"total"`
}
```

sqlx resolves embedded structs. `db:"created_at"` and `db:"updated_at"` are picked up from `BaseEntity`. NamedExec maps `:created_at` and `:updated_at` from the embedded fields automatically.

## Usage — Construction and Mutations

```go
// Service — create
func (s *serviceImpl) PlaceOrder(ctx context.Context, input orderDTO) (orderDTO, error) {
    id, _ := uuid.NewV7()
    entity := orderEntity{
        ID:     id,
        UserID: input.userID,
        Status: "pending",
        Total:  input.total.AmountInCents,
    }
    entity.SetCreate() // sets CreatedAt + UpdatedAt

    err := s.uow.Execute(ctx, func(tx *sqlx.Tx) error {
        return s.repo.insertOrder(ctx, tx, entity)
    })
    // ...
}

// Service — update
func (s *serviceImpl) CancelOrder(ctx context.Context, orderID string) error {
    return s.uow.Execute(ctx, func(tx *sqlx.Tx) error {
        entity, err := s.repo.getByID(ctx, tx, orderID)
        if err != nil {
            return err
        }
        entity.Status = "cancelled"
        entity.SetUpdate() // bumps UpdatedAt only
        return s.repo.updateStatus(ctx, tx, entity)
    })
}
```

`.SetCreate()` on creation. `.SetUpdate()` on mutation. Nothing else touches these fields.

## Repository — Timestamps Are Part of the Entity

Repository never sets timestamps manually. It just maps entity fields to named params:

```go
func (r *repository) insertOrder(ctx context.Context, tx *sqlx.Tx, e domain.orderEntity) error {
    _, err := tx.NamedExecContext(ctx,
        "INSERT INTO orders (id, user_id, status, total, created_at, updated_at)
         VALUES (:id, :user_id, :status, :total, :created_at, :updated_at)", e)
    return err
}

func (r *repository) updateStatus(ctx context.Context, tx *sqlx.Tx, e domain.orderEntity) error {
    _, err := tx.NamedExecContext(ctx,
        "UPDATE orders SET status = :status, updated_at = :updated_at WHERE id = :id", e)
    return err
}
```

The `:updated_at` param is always included in UPDATE queries. The caller already called `.SetUpdate()`. Repository just binds the value.

## Schema

```sql
-- MySQL
CREATE TABLE orders (
    id CHAR(36) NOT NULL PRIMARY KEY,
    user_id CHAR(36) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    total BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    INDEX idx_user_id (user_id),
    INDEX idx_created_at (created_at)
);
```

```sql
-- PostgreSQL
CREATE TABLE orders (
    id UUID NOT NULL PRIMARY KEY,
    user_id UUID NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    total BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);
```

```sql
-- MSSQL
CREATE TABLE orders (
    id UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
    user_id UNIQUEIDENTIFIER NOT NULL,
    status NVARCHAR(50) NOT NULL DEFAULT 'pending',
    total BIGINT NOT NULL DEFAULT 0,
    created_at DATETIME2 NOT NULL,
    updated_at DATETIME2 NOT NULL
);
CREATE INDEX idx_orders_user_id ON orders (user_id);
CREATE INDEX idx_orders_created_at ON orders (created_at);
```

No DB defaults, no triggers, no `ON UPDATE`. The application is the sole owner of timestamp values. Every INSERT sets `created_at` and `updated_at` via `.SetCreate()`. Every UPDATE sets `updated_at` via `.SetUpdate()`. The repository always includes both columns in named params. If a code path forgets, the DB rejects the `NOT NULL` — a loud, immediate error, not a silent wrong value.

## Rules

| Rule | Why |
|------|-----|
| Every table has `created_at` and `updated_at` | Non-negotiable. Debugging, auditing, monitoring all depend on timestamps. |
| Embed `shared.BaseEntity` in entity structs | `.SetCreate()` and `.SetUpdate()` are one call each. Zero repetition. |
| `.SetCreate()` at construction | Sets both created_at and updated_at in one shot |
| `.SetUpdate()` on every mutation | Bumps updated_at. Caller must do this before the repo call. |
| Repository never sets timestamps | Timestamps are domain state, not persistence concern. Entity owns them. |
| Every UPDATE includes `updated_at` in named params | Reflects the entity's current state after `.SetUpdate()`. |
| DB defaults as safety net only | `CURRENT_TIMESTAMP` and `ON UPDATE` catch forgotten timestamps. App code is authoritative. |
| No `*time.Time` pointers | Zero value (`time.Time{}`) is 0001-01-01 — easily spotted as a bug. Nullable timestamps hide mistakes. |

---

## Cross-Module Access

Other modules import `modules/order/ports` — never `adapters/`, never `domain/`. Use the exported `Key` constant — no magic strings.

```go
// modules/notification/module.go
import (
    orderPorts "project/modules/order/ports"
)

func Register(c *shared.Container, r chi.Router) {
    orderAPI := shared.MustGet[orderPorts.Service](c, orderPorts.Key)
    // ...
}
```

## Route Registration — Inside the Module

Each module mounts its own routes. `module.go`'s `Register` function also registers routes. `cmd/` never constructs handlers or knows URL paths.

URLs group by module, never by transport layer. Version prefix goes directly on the resource path — no `/api` segment:

```
/v1/auth/login      # NOT /api/v1/auth/login
/v1/orders          # NOT /api/v1/orders
/v1/pvp/events      # NOT /api/v1/pvp/events
```

`/api` is noise. It tells the client nothing useful — every request is already an API call. Versioning via `/v1/` prefix already distinguishes API routes from static files or health checks. Two segments instead of three. If you need a separate API gateway path later, the reverse proxy can add `/api` at the edge without touching application code.

```go
// modules/order/module.go
package order

func Register(c *shared.Container, r chi.Router) {
    // Build dependencies — use exported Key constants, no magic strings
    uow := db.NewUnitOfWork(c.DB)
    repo := postgres.NewRepository(c.DB)
    userAPI := shared.MustGet[userPorts.Service](c, userPorts.Key)
    paymentAPI := shared.MustGet[paymentPorts.Service](c, paymentPorts.Key)

    svc := newService(repo, uow, userAPI, paymentAPI, c.EventBus)
    c.Set(ports.Key, svc)

    // Mount routes — this module owns its URL space
    handler := httpAdapter.NewHandler(svc, c.Logger)
    httpAdapter.RegisterRoutes(r, handler)

    // Subscribe to events
    c.EventBus.Subscribe(payment.PaymentCompleted{}, svc.handlePaymentCompleted)
}
```

```go
// modules/order/adapters/http/routes.go
package http

import "github.com/go-chi/chi/v5"

func RegisterRoutes(r chi.Router, h *OrderHandler) {
    r.Post("/api/orders", h.Create)
    r.Get("/api/orders", h.List)
    r.Get("/api/orders/{id}", h.GetByID)
    r.Post("/api/orders/{id}/cancel", h.Cancel)
}
```

## Wiring in `cmd/`

```go
// cmd/server/main.go
func main() {
    db := connectDB()
    cfg := config.Load()
    cfg.Validate()
    c := shared.NewContainer(db, cfg)

    r := chi.NewRouter()

    // Two calls per module. Route URLs live inside the module.
    // cmd/ knows nothing about paths, handlers, or handler construction.
    order.Register(c, r)
    payment.Register(c, r)
    notification.Register(c, r)

    http.ListenAndServe(":"+cfg.AppPort, r)
}
```

One call per module. Module decides its own URL space, handler wiring, everything. `cmd/` is just a flat list of `module.Register(c, r)`.

---

# Structured HTTP Responses

## ApiResponse Envelope

Every HTTP response uses a single envelope type. No free-form JSON. All handlers return the same shape.

```go
type ApiResponse struct {
    Success    bool                `json:"success"`
    Code       int                 `json:"code"`
    Message    string              `json:"message"`
    Data       any                 `json:"data,omitempty"`
    Pagination *PaginationMetadata `json:"pagination,omitempty"`
    Errors     any                 `json:"errors,omitempty"`
    TraceID    string              `json:"trace_id,omitempty"`
}
```

| Field | When Present | Description |
|-------|-------------|-------------|
| `success` | Always | `true` for 2xx, `false` otherwise. Derived automatically from code. |
| `code` | Always | Mirrors HTTP status code. |
| `message` | Always | Human-readable summary. **5xx: hardcoded generic message, never expose internal details.** |
| `data` | 2xx with body | Payload — object for single resource, array for list. Omitted on empty/no-content. |
| `pagination` | 2xx paginated list | Pagination metadata. Omitted for non-paginated responses. |
| `errors` | 4xx validation failures only | Field-level errors as key-value dict. Omitted for other error types. |
| `trace_id` | Error responses (non-2xx) only | Request ID injected from `middleware.GetReqID(ctx)` for debugging. Not exposed on success. |

## Why `success` Instead of `status`

A boolean `success` + numeric `code` is more useful than `status` alone:
- **Client-side**: `if (response.success)` is cleaner than `if (response.status >= 200 && response.status < 300)`.
- **No ambiguity**: `success: true` means "this is the happy path." A 201 or 204 is still success.
- **Derived automatically**: `NewResponse` computes `success` from the code range. Callers can't get it wrong.

## Pagination

Pagination flows through three layers. Each layer owns its part:

```
Handler:  extract query params → Params struct
Service:  call Params.Offset() → pass (pageSize, offset) to repo
Service:  call NewMetadata(page, pageSize, total) → Metadata struct
Handler:  convert Metadata → PaginationMetadata (HTTP)
Repo:     receives raw (pageSize int, offset int) — never Params
```

### Transport-Agnostic Types — `modules/shared/pagination/`

```go
// modules/shared/pagination/pagination.go
package pagination

// Params is transport-agnostic pagination input.
// Extracted by the handler layer from query string.
type Params struct {
    Page     int
    PageSize int
}

// Offset calculates the SQL OFFSET value.
// Called in the service layer — never in the handler, never in the repository.
func (p Params) Offset() int {
    if p.Page < 1 {
        return 0
    }
    return (p.Page - 1) * p.PageSize
}

// NewParams creates validated params with defaults.
func NewParams(page, pageSize int) Params {
    if page < 1 {
        page = 1
    }
    if pageSize < 1 {
        pageSize = 10
    }
    return Params{Page: page, PageSize: pageSize}
}

const DefaultMaxPages = 50

// Metadata is transport-agnostic pagination output.
// Built by the service layer.
type Metadata struct {
    Page       int
    PageSize   int
    TotalItems int
    TotalPages int
}

// NewMetadata calculates pagination info and applies the global safety cap.
func NewMetadata(page, pageSize, totalItems int) Metadata {
    totalPages := 0
    if pageSize > 0 {
        totalPages = (totalItems + pageSize - 1) / pageSize
    }
    if totalPages > DefaultMaxPages {
        totalPages = DefaultMaxPages
    }
    return Metadata{
        Page:       page,
        PageSize:   pageSize,
        TotalItems: totalItems,
        TotalPages: totalPages,
    }
}
```

### Handler — Extract Params from Query String

```go
// modules/shared/httputil/handler.go — on BaseHandler
func (h *BaseHandler) GetPaginationParams(r *http.Request) pagination.Params {
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    pageSize, _ := strconv.Atoi(r.URL.Query().Get("page_size"))

    params := pagination.NewParams(page, pageSize)

    // Safety caps
    if params.Page > 50 {
        params.Page = 50
    }
    if params.PageSize > 100 {
        params.PageSize = 100
    }
    return params
}
```

### Service — Call Offset(), Build Metadata

Offset calculation happens **in the service**, not the repository. The repository receives raw `(pageSize, offset)` ints — it doesn't know about pages.

```go
// modules/order/service.go
func (s *orderServiceImpl) List(ctx context.Context, p pagination.Params) ([]domain.Order, pagination.Metadata, error) {
    orders, total, err := s.repo.FindByUser(ctx, userID, p.PageSize, p.Offset())
    if err != nil {
        return nil, pagination.Metadata{}, err
    }
    return orders, pagination.NewMetadata(p.Page, p.PageSize, total), nil
}
```

### Repository — Raw ints, No Params Type

```go
// modules/order/ports/repository.go
type Repository interface {
    FindByUser(ctx context.Context, userID string, pageSize int, offset int) ([]domain.Order, int, error)
}
```

Repository signatures are `(pageSize int, offset int)`. Never pass `Params` to the repository — it's a transport concept. The repo deals in SQL primitives: `LIMIT ? OFFSET ?`.

### Handler — Convert to HTTP Envelope

```go
// modules/shared/httputil/response.go
type PaginationMetadata struct {
    Page       int `json:"page"`
    PageSize   int `json:"page_size"`
    TotalItems int `json:"total_items"`
    TotalPages int `json:"total_pages"`
}

func FromPaginationMetadata(m pagination.Metadata) PaginationMetadata {
    return PaginationMetadata{
        Page:       m.Page,
        PageSize:   m.PageSize,
        TotalItems: m.TotalItems,
        TotalPages: m.TotalPages,
    }
}
```

No `has_next` / `has_prev` in the HTTP envelope. The client derives: `page < totalPages` → more ahead. URL-based navigation (`next`/`prev` links) **not** included — client constructs its own URLs.

### Pagination Rules

| Rule | Why |
|------|-----|
| `Params` is transport-agnostic, in `modules/shared/pagination/` | Shared across all modules. Not tied to HTTP. |
| `Offset()` called in service, not repository | Repo deals in SQL primitives. Service owns the offset formula. |
| Repository receives `(pageSize int, offset int)` | Repo doesn't know about page numbers. Just `LIMIT ? OFFSET ?`. |
| `NewMetadata(page, pageSize, total)` in service | Service owns metadata construction. Repo returns `(rows, total, error)`. |
| Handler converts `Metadata` → `PaginationMetadata` | Transport concern. HTTP JSON tags live in `httputil/`. |
| Safety caps: max 50 pages, max 100 page_size | Prevents `page=99999` from hammering the DB. |
| Capped `TotalPages` at 50 | Attackers can't discover total row count by scanning page numbers. |

### 2xx Success

| Code | Constructor | `data` | `pagination` |
|------|------------|--------|-------------|
| `200` | `NewSuccessResponse(data)` | Resource or result | `omitempty` |
| `200` paginated | `NewSuccessResponse(data)` + set `Pagination` | Array | Set |
| `201` | `NewCreatedResponse(message, data)` | Created resource | `omitempty` |
| `200` message | `NewSuccessMessageResponse(msg)` | `omitempty` | `omitempty` |

### 4xx Client Error

| Code | Constructor | `errors` | Notes |
|------|------------|---------|-------|
| `400` | `NewBadRequestResponse(message)` | `omitempty` | Malformed body, bad syntax |
| `400` validate | `NewValidationErrorResponse(message, errs)` | Field errors dict | Semantic validation failure |
| `401` | `NewUnauthorizedResponse(message)` | `omitempty` | Missing/expired auth |
| `403` | `NewForbiddenResponse(message)` | `omitempty` | Authenticated but forbidden |
| `404` | `NewNotFoundResponse(message)` | `omitempty` | Resource not found |
| `405` | `NewMethodNotAllowedResponse()` | `omitempty` | Wrong HTTP method |

### 5xx Server Error

| Code | Constructor | `message` | `errors` |
|------|------------|-----------|---------|
| `500` | `NewInternalErrorResponse()` | `"Internal Server Error"` — **hardcoded** | `omitempty` |

**5xx rule**: `NewInternalErrorResponse()` takes **zero parameters**. The message is always `"Internal Server Error"`. Never expose stack traces, SQL errors, upstream hostnames, or panic messages to the client. Log the real error server-side with `h.Error(ctx, msg, err)`. The client sees only the generic message.

## Handler Pattern (with BaseHandler)

```go
type BaseHandler struct {
    Log logger.Logger
}

// JSON writes raw JSON — only for non-envelope responses (webhooks, callbacks).
func (h *BaseHandler) JSON(w http.ResponseWriter, status int, data any)

// SendResponse writes an ApiResponse and injects trace_id on errors.
func (h *BaseHandler) SendResponse(w http.ResponseWriter, r *http.Request, resp ApiResponse)

// Success helpers
func (h *BaseHandler) Success(w http.ResponseWriter, data any)
func (h *BaseHandler) PaginatedSuccess(w http.ResponseWriter, data any, meta PaginationMetadata)
func (h *BaseHandler) SuccessMessage(w http.ResponseWriter, message string)

// Error helpers
func (h *BaseHandler) BadRequest(w http.ResponseWriter, r *http.Request, message string)
func (h *BaseHandler) Unauthorized(w http.ResponseWriter, r *http.Request, message string)
func (h *BaseHandler) Forbidden(w http.ResponseWriter, r *http.Request, message string)
func (h *BaseHandler) NotFound(w http.ResponseWriter, r *http.Request, message string)
func (h *BaseHandler) MethodNotAllowed(w http.ResponseWriter, r *http.Request, message string)
func (h *BaseHandler) ValidationError(w http.ResponseWriter, r *http.Request, message string, errors any)
func (h *BaseHandler) InternalError(w http.ResponseWriter, r *http.Request)

// Error logs an error with request context (request_id injection).
func (h *BaseHandler) Error(ctx context.Context, message string, err error)
```

## Handler Implementation Convention

Every handler follows the same flow: **decode → validate → call service → check error type → respond**.

```go
// POST /api/orders
func (h *OrderHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        h.BadRequest(w, r, "Invalid request body")
        return
    }

    if errs := req.Validate(); len(errs) > 0 {
        h.ValidationError(w, r, "Validation failed", errs)
        return
    }

    order, err := h.service.PlaceOrder(r.Context(), req)
    if err != nil {
        // Custom error type with extra data?
        var stockErr *order.InsufficientStockError
        if errors.As(err, &stockErr) {
            h.BadRequest(w, r, stockErr.Error())
            return
        }
        // Plain domain error? Message is safe to show.
        h.BadRequest(w, r, err.Error())
        return
    }

    h.JSON(w, http.StatusCreated, NewCreatedResponse("Order created", order))
}
```

### Request Validation — `.Validate()` Pattern

Every request struct carries a `Validate()` method. The handler calls it after decode. Validation lives on the request, not in the service.

```go
// modules/order/adapters/http/request.go
package http

import "github.com/go-playground/validator/v10"

var validate = validator.New()

type CreateOrderRequest struct {
    UserID string            `json:"user_id" validate:"required,uuid"`
    Items  []OrderItemRequest `json:"items" validate:"required,min=1,dive"`
}

func (r CreateOrderRequest) Validate() map[string]string {
    err := validate.Struct(r)
    if err == nil {
        return nil
    }
    errs := make(map[string]string)
    for _, e := range err.(validator.ValidationErrors) {
        errs[e.Field()] = e.Tag()
    }
    return errs
}
```

**How it works:**
1. `go-playground/validator` reads struct tags (`validate:"required,uuid"`) and checks value constraints
2. `Validate()` calls `validate.Struct(r)` — one call validates every field
3. On failure, maps field names to failed rule tags: `{"user_id": "required", "items": "min"}`
4. Handler returns the map as `errors` in the `ApiResponse` envelope — client knows which field failed which rule

**Rules:**
- One `validator.New()` instance at package level — no per-request allocation
- `Validate()` returns `nil` on success, `map[string]string` on failure — nil means "passed"
- Only use struct tags for validation — never custom imperative checks in the handler
- `dive` tag validates nested slice elements: `[]OrderItemRequest with validate:"required,min=1,dive"` checks each item

## Swagger / OpenAPI Annotations

### Setup

Generated via [swaggo/swag](https://github.com/swaggo/swag). Annotations live **on the handler methods** in `adapters/http/handler.go` — nowhere else.

```go
// @title           Order Service API
// @version         1.0
// @description     Order management module
// @host            localhost:8080
// @BasePath        /
```

Module-level swagger header lives once per module, in `handler.go` above the handler struct. `cmd/` composes all modules' swagger headers when generating docs.

### Handler Annotation Rules

Every handler method gets a complete godoc block honoring this template:

```go
// Create godoc
// @Summary      Create a new order
// @Description  Validates items, checks stock, charges payment, creates the order.
// @Tags         orders
// @Accept       json
// @Produce      json
// @Param        request  body      CreateOrderRequest  true  "Order payload"
// @Success      201      {object}  httputil.ApiResponse{data=OrderResponse}
// @Failure      400      {object}  httputil.ApiResponse
// @Failure      401      {object}  httputil.ApiResponse
// @Failure      422      {object}  httputil.ApiResponse
// @Failure      500      {object}  httputil.ApiResponse
// @Router       /api/orders [post]
// @Security     BearerAuth
func (h *OrderHandler) Create(w http.ResponseWriter, r *http.Request) {
```

### Required Fields

| Field | Required? | Notes |
|-------|----------|-------|
| `@Summary` | Yes | Short, imperative mood. Under one line. |
| `@Description` | Yes | Behavior, not implementation. What it does, not how. |
| `@Tags` | Yes | Module name in lowercase. Group routes in Swagger UI. |
| `@Accept` | Yes | Always `json` for these APIs. |
| `@Produce` | Yes | Always `json` for these APIs. |
| `@Param` | Yes (if body) | `name` `kind` `type` `required` `"description"` |
| `@Param` (query) | Yes (if query) | `name  query  string  true  "param description"` |
| `@Param` (path) | Yes (if path) | `name  path  string  true  "param description"` |
| `@Success` | For 2xx | Include the response type — `{data=OrderResponse}` shows the inner shape |
| `@Failure` | For 4xx/5xx | List every failure code the handler returns. No shortcuts. |
| `@Router` | Yes | `path [method]` format |
| `@Security` | Yes (auth routes) | `BearerAuth` for JWT routes. Omit for public routes. |

### Response Type Notation

```
@Success 200 {object} httputil.ApiResponse{data=OrderResponse}
@Success 200 {object} httputil.ApiResponse{data=[]OrderResponse}
@Success 200 {object} httputil.ApiResponse{data=MessageResponse}
@Failure 400 {object} httputil.ApiResponse
@Failure 422 {object} httputil.ApiResponse
```

Always wrap in `httputil.ApiResponse`. The `{data=Xxx}` suffix tells swaggo the inner type. If `data` is omitted (`{object} httputil.ApiResponse`), Swagger shows no inner schema.

### Failures — List Every Code

Every `@Failure` the handler actually returns. Don't skip 500 just because "it could happen anywhere" — swaggo needs it.

```go
// @Failure 400 {object} httputil.ApiResponse  — bad JSON, malformed request
// @Failure 401 {object} httputil.ApiResponse  — missing/expired JWT
// @Failure 403 {object} httputil.ApiResponse  — insufficient permissions
// @Failure 404 {object} httputil.ApiResponse  — resource not found
// @Failure 409 {object} httputil.ApiResponse  — conflict (duplicate)
// @Failure 422 {object} httputil.ApiResponse  — validation error
// @Failure 500 {object} httputil.ApiResponse  — internal server error
```

### Handler with Query + Path Params

```go
// List godoc
// @Summary      List orders
// @Description  Returns paginated orders for the authenticated user.
// @Tags         orders
// @Accept       json
// @Produce      json
// @Param        page       query     int     false  "Page number (1-indexed)"  default(1)
// @Param        page_size  query     int     false  "Items per page"            default(20)
// @Param        status     query     string  false  "Filter by status"         Enums(pending,confirmed,cancelled)
// @Success      200        {object}  httputil.ApiResponse{data=[]OrderListResponse}
// @Failure      401        {object}  httputil.ApiResponse
// @Failure      500        {object}  httputil.ApiResponse
// @Router       /api/orders [get]
// @Security     BearerAuth
func (h *OrderHandler) List(w http.ResponseWriter, r *http.Request) {

// GetByID godoc
// @Summary      Get order by ID
// @Description  Returns a single order by its unique identifier.
// @Tags         orders
// @Accept       json
// @Produce      json
// @Param        id   path      string  true  "Order ID (UUID)"
// @Success      200  {object}  httputil.ApiResponse{data=OrderResponse}
// @Failure      401  {object}  httputil.ApiResponse
// @Failure      404  {object}  httputil.ApiResponse
// @Failure      500  {object}  httputil.ApiResponse
// @Router       /api/orders/{id} [get]
// @Security     BearerAuth
func (h *OrderHandler) GetByID(w http.ResponseWriter, r *http.Request) {
```

### Response Structs — `swaggertype` Hints

For structs used in `@Success` annotations, add type hints so swaggo resolves them correctly:

```go
// modules/order/adapters/http/response.go
package http

// OrderResponse is the API response for a single order.
type OrderResponse struct {
    ID        string              `json:"id"         example:"01J..."     description:"Order ID"`
    UserID    string              `json:"user_id"    example:"01J..."     description:"User ID"`
    Status    string              `json:"status"     example:"confirmed"  description:"Order status"`
    Items     []OrderItemResponse `json:"items"                            description:"Line items"`
    Total     string              `json:"total"      example:"12.50 USD"  description:"Formatted total"`
    CreatedAt string              `json:"created_at" example:"2026-01-15T08:30:00Z" description:"ISO 8601"`
}

// OrderItemResponse is a line item in an order response.
type OrderItemResponse struct {
    ProductID string `json:"product_id" example:"prod_abc"    description:"Product identifier"`
    Quantity  int    `json:"quantity"    example:"2"          description:"Quantity ordered"`
    UnitPrice string `json:"unit_price"  example:"6.25 USD"   description:"Formatted unit price"`
}
```

### Envelope Types in Shared

```go
// modules/shared/httputil/response.go

// ApiResponse is the standard JSON envelope for all responses.
// @Description Standard API response envelope.
type ApiResponse struct {
    Success    bool                `json:"success"                                     description:"True for 2xx responses"`
    Code       int                 `json:"code"         example:"200"                  description:"HTTP status code"`
    Message    string              `json:"message"      example:"Success"              description:"Human-readable summary"`
    Data       any                 `json:"data,omitempty"                               description:"Response payload"`
    Pagination *PaginationMetadata `json:"pagination,omitempty"                         description:"Pagination metadata"`
    Errors     any                 `json:"errors,omitempty"                             description:"Field-level validation errors"`
    TraceID    string              `json:"trace_id,omitempty" example:"req-abc123"     description:"Request trace ID (errors only)"`
}

// MessageResponse is the response for endpoints that return only a message.
type MessageResponse struct {
    Message string `json:"message" example:"Operation completed" description:"Status message"`
}
```

### File Header — Module-Level Swagger

Each module's `handler.go` starts with a swagger header block. `cmd/` or the main package aggregates these when generating docs:

```go
// modules/order/adapters/http/handler.go
package http

// @title Order Module
// @description Order management — create, view, cancel orders.
// @BasePath /
// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization
```

Or keep the `@securityDefinitions` in `cmd/server/main.go` and let each module only declare its routes.

### Rules

| Rule | Why |
|------|-----|
| Annotations only on handler methods | Single source of truth. No annotation duplication in service/repo. |
| Every failure code listed | Swagger UI shows consumers exactly what can go wrong. |
| `@Failure 500` always present | Internal errors are part of the contract. |
| Response structs have `example` + `description` tags | Swagger UI shows sample values and field docs. |
| `@Tags` match module name | Swagger UI groups by module. One tag per module. |
| `@Param` names match the Go variable name | No drift between `page` in godoc and `r.URL.Query().Get("page")`. |
| No godoc on unexported types | `private structs don't appear in swagger. Only exported response types. |
| `swag init` runs in CI | Generated docs diverge from code. CI catches stale docs. |

## Response Examples

**200 single resource:**
```json
{
    "success": true,
    "code": 200,
    "message": "Success",
    "data": { "id": "abc", "email": "a@b.com" }
}
```

**200 paginated list:**
```json
{
    "success": true,
    "code": 200,
    "message": "Success",
    "data": [ ... ],
    "pagination": { "page": 1, "page_size": 20, "total_items": 143, "total_pages": 8 }
}
```

**400 validation error:**
```json
{
    "success": false,
    "code": 400,
    "message": "Validation failed",
    "errors": { "email": "email is required", "age": "must be greater than 0" },
    "trace_id": "req-abc123"
}
```

**404 not found:**
```json
{
    "success": false,
    "code": 404,
    "message": "User not found",
    "trace_id": "req-abc123"
}
```

**500 internal error:**
```json
{
    "success": false,
    "code": 500,
    "message": "Internal Server Error",
    "trace_id": "req-abc123"
}
```

Note: 500 `message` is always `"Internal Server Error"` — identical in every response. The real error is in the server logs, keyed by `trace_id`.

## Where Response Types Live

```
modules/shared/httputil/
  response.go          # ApiResponse, PaginationMetadata, all NewXxx constructors
  handler.go           # BaseHandler with typed response helpers + GetPaginationParams()

modules/order/
  adapters/http/
    request.go         # CreateOrderRequest, CancelOrderRequest (module-specific)
    response.go        # OrderResponse, OrderItemResponse (module-specific — inner data types)
```

`ApiResponse` is the envelope — it lives in `shared/httputil/`. Each module's `adapters/http/response.go` defines only the inner `data` shape. Handlers compose module response types into the shared envelope.

## Empty List → `[]`, Never `null`

When `data` is an array type, an empty result must serialize as `[]`, never as `null`.

```go
// Wrong — nil slice serializes to null
orders, _ := h.service.List(r.Context())
if len(orders) == 0 {
    h.Success(w, nil)       // → {"data": null}
    return
}
h.Success(w, orders)

// Correct — empty slice serializes to []
orders, _ := h.service.List(r.Context())
if orders == nil {
    orders = []OrderResponse{} // zero-length, non-nil → "data": []
}
h.Success(w, orders)              // → {"data": []}
```

**Rule**: every list or array field in a response struct defaults to `[]` not `nil`. This applies to:

- Top-level `data` when it's an array
- Nested array fields inside a response struct (`items`, `tags`, `roles`)
- `pagination` is `null` when not paginated — that's the only valid null

```go
// Module response structs — initialize arrays
type OrderResponse struct {
    Items []OrderItemResponse `json:"items"` // always allocate in constructor
}

func fromDTO(d orderDTO) OrderResponse {
    items := make([]OrderItemResponse, 0, len(d.items))
    for _, item := range d.items {
        items = append(items, fromItemDTO(item))
    }
    return OrderResponse{Items: items}
}
```

Client expects `items: []`, never `items: null`. A `null` forces null checks in every consumer. An empty array requires none.

---

# Module Errors

## Two Error Categories

Every error returned by a service is one of two kinds:

| Kind | Created with | Safe to show client? | Handler does |
|------|-------------|---------------------|--------------|
| **Domain error** | `errors.New()` or custom type implementing `DomainError` | Yes | `h.BadRequest(w, r, err.Error())` |
| **Internal error** | `fmt.Errorf("[Type.Method] ...: %w", err)` | No | `h.Error(ctx, msg, err)` then `h.InternalError(w, r)` |

The handler must distinguish them. Use a marker interface.

## `shared.DomainError` — Marker Interface

```go
// modules/shared/errors.go
package shared

// DomainError marks an error as safe to expose to the client.
// The handler checks for this interface; if satisfied, err.Error()
// goes directly into the response message.
type DomainError interface {
    // DomainMarker is a no-op method that prevents accidental implementation.
    // Only errors explicitly designed for client exposure satisfy this.
    DomainMarker()
}
```

Interface only. No fields. No `Unwrap()`. The sole purpose: let the handler ask "is this safe to show?"

## Module `errors.go` — One File Per Module

Each module defines its domain errors in `errors.go`. Exported. Everything the module can tell a caller lives here.

```go
// modules/order/errors.go
package order

import "fmt"

// ── Plain domain errors ──

type orderError string

func (e orderError) Error() string    { return string(e) }
func (e orderError) DomainMarker() {} // satisfies shared.DomainError

// Sentinel-like domain errors. Handler uses errors.Is to match.
var (
    ErrOrderNotFound     = orderError("order not found")
    ErrAlreadyCancelled  = orderError("order already cancelled")
    ErrInsufficientStock = orderError("insufficient stock")
)

// ── Structured domain error (only when handler needs extra data) ──

type InsufficientStockDetails struct {
    ProductID string
    Requested int
    Available int
}

func (e *InsufficientStockDetails) Error() string {
    return fmt.Sprintf("insufficient stock for %s (requested %d, available %d)",
        e.ProductID, e.Requested, e.Available)
}
func (e *InsufficientStockDetails) DomainMarker() {} // satisfies shared.DomainError
```

Why a custom string type instead of bare `errors.New()`: it satisfies the `DomainError` interface. Plain `errors.New("msg")` does NOT — the handler treats it as internal, logs it, returns 500. The type system enforces the distinction.

## Service — Return Domain Errors, Wrap Internal Ones

```go
// modules/order/service.go
func (s *orderService) PlaceOrder(ctx context.Context, input orderDTO) (orderDTO, error) {
    // Domain check → domain error
    if !s.inventory.HasStock(input.productID) {
        return orderDTO{}, ErrInsufficientStock
    }

    // DB call fails → internal error, wrapped with [Type.Method]
    entity, err := s.repo.getByID(ctx, tx, input.orderID)
    if err != nil {
        return orderDTO{}, fmt.Errorf("[orderService.PlaceOrder] get order: %w", err)
    }

    // Upstream module call fails → internal error
    _, err = s.userAPI.GetEmail(ctx, input.userID)
    if err != nil {
        return orderDTO{}, fmt.Errorf("[orderService.PlaceOrder] get user email: %w", err)
    }

    // Business rule → domain error with structured data
    if entity.status == statusCancelled {
        return orderDTO{}, &InsufficientStockDetails{
            ProductID: input.productID,
            Requested: input.quantity,
            Available: stock,
        }
    }

    // ...
}
```

Key:
- **`return ErrXxx`** — domain error. Handler reads `err.Error()` → client sees it.
- **`return &StructError{...}`** — domain error with data. Handler `errors.As` → extracts fields.
- **`return fmt.Errorf("[Type.Method] ...: %w", err)`** — internal error. Handler logs + 500.

## Handler — One Pattern for All Modules

```go
func (h *OrderHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        h.BadRequest(w, r, "Invalid request body")
        return
    }
    if errs := req.Validate(); len(errs) > 0 {
        h.ValidationError(w, r, "Validation failed", errs)
        return
    }

    order, err := h.service.PlaceOrder(r.Context(), req)
    if err != nil {
        // 1. Structured domain error? Extract fields.
        var stockErr *order.InsufficientStockDetails
        if errors.As(err, &stockErr) {
            h.BadRequest(w, r, stockErr.Error())
            return
        }

        // 2. Domain error? Safe to show.
        var domainErr shared.DomainError
        if errors.As(err, &domainErr) {
            h.BadRequest(w, r, err.Error())
            return
        }

        // 3. Everything else = internal failure. Log + 500.
        h.Error(r.Context(), "create order", err)
        h.InternalError(w, r)
        return
    }

    h.JSON(w, http.StatusCreated, NewCreatedResponse("Order created", order))
}
```

Resolution order matters: check concrete types first (`*InsufficientStockDetails`), then the marker interface (`DomainError`), then fall through to 500. `errors.As` walks the chain — wrapped internal errors resolve to the innermost cause.

Errors use the same `[Type.Method]` prefix convention — see [Log Message Convention](#log-message-convention) for the canonical definition.

## Error Flow Diagram

```
Service                         Handler                      Client
───────                         ───────                      ──────
return ErrNotFound           →  errors.As → DomainError   → 400 {"message":"order not found"}
                                h.BadRequest(w, r, err.Error())

return &InsufficientStockDetails → errors.As → concrete    → 400 {"message":"insufficient stock for X"}
                                   h.BadRequest(w, r, stockErr.Error())

return fmt.Errorf(            →  errors.As → no match      → 500 {"message":"Internal Server Error"}
  "[orderSvc.PlaceOrder]:      →  fallthrough                h.Error(ctx, msg, err)
  %w", sqlErr)                                                h.InternalError(w, r)
```

## Design Rules

| Rule | Why |
|------|-----|
| Domain errors implement `shared.DomainError` | Handler knows what's safe to expose |
| Plain `errors.New()` is NOT a domain error | No implicit safety. Type system enforces the contract. |
| Custom error type only when handler needs structured fields | Otherwise `orderError("msg")` is enough |
| Internal errors always wrapped with `[Type.Method]` | Greppable. Stack-free. Method-level trace. |
| `errors.As` order: concrete type → DomainError → fallthrough | Concrete types are narrower; check first |
| `err.Error()` for domain errors is the client message | No mapping. Service writes the message. |
| `h.InternalError(w, r)` has no message param | Hardcoded "Internal Server Error". Never leaks internals. |
| `h.Error(ctx, desc, err)` for internal failures only | Domain errors don't need logging — they're expected |
| One `errors.go` per module | All domain errors in one place. Exported for handler and tests. |

---

# Dependency Rules

## The One Rule

**Modules may only depend on:**
1. The `shared` module (shared kernel)
2. Other modules' `ports/` (public contracts)
3. Standard library + third-party packages

**Modules may NOT:**
1. Import anything except `ports/` from another module
2. Import from a module that represents a sibling/peer business capability unless declared as an upstream dependency
3. Create circular imports (compile-time enforced in Go, but the rule prevents logical cycles even via interfaces)

## Dependency Direction

```
┌─────────────────────────────────────┐
│              shared                  │
│   (shared kernel — no dependencies)  │
└─────────────────────────────────────┘
         ▲           ▲           ▲
         │           │           │
   ┌─────┴───┐ ┌─────┴───┐ ┌─────┴───┐
   │  user   │ │  order  │ │ payment │
   └─────────┘ └────┬────┘ └─────────┘
                    │
                    ▼
              ┌─────────────┐
              │ notification │
              └─────────────┘
```

- `shared` depends on nothing internal — only stdlib and third-party libs.
- Business modules depend on `shared` + other modules' `ports/`.
- A module that depends on another module must declare it (see Dependency Declaration).

## Dependency Declaration

Each module documents which modules it depends on in `module.go`:

```go
// modules/order/module.go
// Register wires the order module.
// Dependencies: shared, user.ports, payment.ports
// Provides: order.ports
package order
```

This makes the dependency graph visible and auditable.

---

# Shared Kernel (`modules/shared/`)

## What Belongs in Shared

Only truly shared concerns that every module needs. The bar is **very high**.

| Concern | In shared? | Rationale |
|---------|-----------|-----------|
| Custom error types (app-level) | Yes | Every module uses them |
| Logging interface | Yes | Every module logs |
| Context helpers (user ID from ctx) | Yes | Cross-cutting |
| Base entity (ID, timestamps) | Yes | Every entity has these |
| Pagination types | Yes | Universal |
| Money/decimal type | Yes | Multiple modules deal with money |
| **User model** | **No** | Belongs to `user` module |
| **Order model** | **No** | Belongs to `order` module |
| **Payment logic** | **No** | Belongs to `payment` module |
| HTTP router setup | **No** | Belongs in `cmd/` or a `server` module |
| Config structs | **No** (maybe) | Module-specific config stays in module; app-level config in `cmd/` |

## Shared Kernel Rules

- **No business logic** in shared. Pure data types, interfaces, and helpers only.
- **Never** reference any business module from shared.
- **Minimize** — every addition to shared must be justified. When in doubt, keep it in the module.
- Shared does **not** have an `ports/` directory — its entire package is public (it's the kernel).

## Example `modules/shared/`

```
modules/shared/
  httputil/
    response.go        # ApiResponse, PaginationMetadata, all NewXxx constructors
    handler.go         # BaseHandler with typed response helpers + GetPaginationParams()
  valueobject/         # Shared value objects — immutable, self-validating
    money.go           # Money{Amount, Currency}
    email.go           # Email{Address}
    phone.go           # PhoneNumber{CountryCode, Number}
    rating.go          # Rating{Value, Max}
  platform/
    outbox/
      entity.go        # MessageEntity, Handler interface
      repository.go    # Repository interface + impl
      worker.go        # Worker — poll + dispatch + retry
    logs/
      entity.go        # AlertEntity
      worker.go        # LogsWorker — channel consume + persist
      repository.go    # Repository impl
    config/
      config.go        # Config struct, Load(), Validate()
    db/
      transactor.go    # UnitOfWork[T] interface + sqlx impl
  container.go         # DI Container, Set/Get/MustGet
  errors.go            # DomainError marker interface
  context.go           # GetUserID(ctx), GetRequestID(ctx)
  entity.go           # BaseEntity — CreatedAt + UpdatedAt + SetCreate()/SetUpdate()
  eventbus.go          # EventBus — typed subscribe/publish
```

### Value Objects — `shared/valueobject/`

Value objects are immutable, self-validating types with no identity. They model primitive business concepts that span multiple modules. If a type has no ID field and is compared by all its fields, it's a value object.

**Candidates:** `Money`, `Email`, `PhoneNumber`, `Address`, `Quantity`, `Rating`, `Percentage`, `Currency`, `DateRange`.

Each value object gets its own file: `money.go`, `email.go`, `phone.go`. One type per file. Package is `valueobject`, not `value_objects` or `vo`.

```go
// modules/shared/valueobject/money.go
package valueobject

import "fmt"

// Money is an immutable monetary value with currency. Compare by all fields.
type Money struct {
    amount   int64  // cents — unexported means caller goes through constructor
    currency string // "USD", "EUR", "JPY"
}

func NewMoney(amount int64, currency string) (Money, error) {
    if currency == "" {
        return Money{}, fmt.Errorf("currency is required")
    }
    return Money{amount: amount, currency: currency}, nil
}

func MustMoney(amount int64, currency string) Money {
    m, err := NewMoney(amount, currency)
    if err != nil {
        panic(err)
    }
    return m
}

func (m Money) AmountInCents() int64    { return m.amount }
func (m Money) Currency() string         { return m.currency }
func (m Money) String() string          { return fmt.Sprintf("%.2f %s", float64(m.amount)/100, m.currency) }
func (m Money) Equal(other Money) bool  { return m == other }
```

```go
// modules/shared/valueobject/email.go
package valueobject

import "strings"

// Email is an immutable email address. Guaranteed valid after construction.
type Email struct {
    address string
}

func NewEmail(address string) (Email, error) {
    address = strings.TrimSpace(address)
    if address == "" || !strings.Contains(address, "@") {
        return Email{}, fmt.Errorf("invalid email: %q", address)
    }
    return Email{address: strings.ToLower(address)}, nil
}

func (e Email) String() string  { return e.address }
func (e Email) Domain() string  { return e.address[strings.LastIndex(e.address, "@")+1:] }
func (e Email) Equal(other Email) bool { return e == other }
```

**Properties shared by all value objects:**
- Unexported fields — construction only via `NewXxx()` or `MustXxx()`. No zero-value misuse.
- Validated at construction — invalid state is unrepresentable. Caller never checks `IsValid()`.
- `Equal(other T) bool` — compare by all fields (or just `==` for structs with all comparable fields).
- Immutable — no setters. Only accessor methods.
- No DB tags, no JSON tags, no ORM annotations — pure domain. Only `String()` for display.

**Modules use value objects in their domain types:**

```go
// modules/order/domain/dto.go
type orderDTO struct {
    id    uuid.UUID
    items []orderItemDTO
    total valueobject.Money       // not int64, not float64
    email valueobject.Email       // not string
}
```

A `Money` field says "this is validated money" to every reader. An `int64` field says "maybe it's cents, maybe it's dollars, maybe it's a pointer to something else." Value objects make the domain code self-documenting.

### When NOT to put a type in `valueobject/`

| Scenario | Correct Home |
|----------|-------------|
| Type has an `ID` field | It's an entity, not a value object. Belongs in a module. |
| Type is specific to one module | Keep it in that module's `domain/`. Example: `OrderStatus` stays in `modules/order/domain/`. |
| Type carries DB/JSON tags | It's a persistence or transport concern, not a pure value object. Add mapping in the adapter layer. |
| Type is just a type alias (`type UserID string`) | Keep it in the owning module. A bare string alias isn't a value object. |

---

# Logging

## Principle

One `Logger` interface for all code. No direct dependency on `log/slog`, `zap`, or `zerolog` — always depend on the abstraction. Backed by `slog` (stdlib) via a thin adapter. Fan-out via `TeeHandler`: stdout for dev visibility, database for persisted error/warn alerts.

## Logger Interface — `shared/log.go`

```go
// modules/shared/log.go
package shared

// Logger is the central logging abstraction. All code depends on this interface,
// never on a concrete logging library.
//
// Every method accepts a message followed by alternating key/value pairs
// (slog convention):
//
//   log.Info("user logged in", "user_id", 42, "ip", "10.0.0.1")
type Logger interface {
    Debug(msg string, args ...any)
    Info(msg string, args ...any)
    Warn(msg string, args ...any)
    Error(msg string, args ...any)

    // With returns a child logger that prepends the given key/value pairs
    // to every subsequent log entry. Use for request-scoped fields.
    With(args ...any) Logger
}
```

**Why an interface and not `*slog.Logger` directly:**
- Testable — inject `NoopLogger` in unit tests
- Swappable — replace slog with zap without touching business code
- Enforces the convention — only four levels, no `Log(level, ...)` escape hatch

## Slog Adapter

```go
// modules/shared/log_slog.go
package shared

import "log/slog"

type slogAdapter struct {
    inner *slog.Logger
}

func NewSlogLogger(sl *slog.Logger) Logger {
    if sl == nil {
        sl = slog.Default()
    }
    return &slogAdapter{inner: sl}
}

func (l *slogAdapter) Debug(msg string, args ...any) { l.inner.Debug(msg, args...) }
func (l *slogAdapter) Info(msg string, args ...any)  { l.inner.Info(msg, args...) }
func (l *slogAdapter) Warn(msg string, args ...any)  { l.inner.Warn(msg, args...) }
func (l *slogAdapter) Error(msg string, args ...any) { l.inner.Error(msg, args...) }

func (l *slogAdapter) With(args ...any) Logger {
    return &slogAdapter{inner: l.inner.With(args...)}
}
```

## TeeHandler — Dual Output (Console + Database)

One `*slog.Logger` fans out to multiple `slog.Handler`:

```go
// modules/shared/log_tee.go
// TeeHandler fans out slog records to multiple handlers (stdout + DB).
type TeeHandler struct {
    handlers []slog.Handler
}

func NewTeeHandler(handlers ...slog.Handler) *TeeHandler {
    return &TeeHandler{handlers: handlers}
}

func (h *TeeHandler) Handle(ctx context.Context, r slog.Record) error {
    for _, handler := range h.handlers {
        if handler.Enabled(ctx, r.Level) {
            _ = handler.Handle(ctx, r)
        }
    }
    return nil
}

func (h *TeeHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
    newHandlers := make([]slog.Handler, len(h.handlers))
    for i, h2 := range h.handlers {
        newHandlers[i] = h2.WithAttrs(attrs)
    }
    return &TeeHandler{handlers: newHandlers}
}
```

## DBHandler — Capture WARN+ERROR to Channel

A `slog.Handler` that captures only `WARN` and `ERROR` records and pushes them onto a buffered channel. Non-blocking — drops if channel is full (logging must never stall the application).

```go
// modules/shared/log_db_handler.go
type LogRecord struct {
    ID           uuid.UUID
    Level        string   // "WARN" or "ERROR"
    Message      string
    Context      json.RawMessage   // key=value pairs as JSON object
    Trace        string            // stack trace (ERROR only)
    SourceMethod string            // extracted [Type.Method] prefix
    CreatedAt    time.Time
}

type dbHandler struct {
    out   chan<- LogRecord
    attrs []slog.Attr
}

func NewDBHandler(out chan<- LogRecord) slog.Handler {
    return &dbHandler{out: out}
}

func (h *dbHandler) Enabled(_ context.Context, level slog.Level) bool {
    return level >= slog.LevelWarn   // only WARN+ERROR go to DB
}

func (h *dbHandler) Handle(ctx context.Context, r slog.Record) error {
    // Extract [Type.Method] from message prefix
    sourceMethod := ""
    cleanMsg := r.Message
    if idx := strings.Index(r.Message, "]"); idx > 0 && strings.HasPrefix(r.Message, "[") {
        sourceMethod = r.Message[1:idx]
        cleanMsg = strings.TrimSpace(r.Message[idx+1:])
    }

    // Collect key=value pairs
    fields := map[string]any{}
    for _, a := range h.attrs {
        fields[a.Key] = a.Value.Any()
    }
    r.Attrs(func(a slog.Attr) bool {
        fields[a.Key] = a.Value.Any()
        return true
    })
    ctxJSON, _ := json.Marshal(fields)

    // Capture stack trace on ERROR
    var trace string
    if r.Level >= slog.LevelError {
        stack := make([]byte, 2048)
        n := runtime.Stack(stack, false)
        trace = string(stack[:n])
    }

    id, _ := uuid.NewV7()
    record := LogRecord{
        ID:           id,
        Level:        r.Level.String(),
        Message:      cleanMsg,
        Context:      ctxJSON,
        Trace:        trace,
        SourceMethod: sourceMethod,
        CreatedAt:    r.Time,
    }

    // Non-blocking send — drop if channel full
    select {
    case h.out <- record:
    default:
    }
    return nil
}
```

Key behaviors:
- **`Enabled` filter** — only `WARN` and `ERROR` hit the DB. DEBUG/INFO are console-only.
- **Source method extraction** — parses `[orderService.PlaceOrder]` from message prefix. No regex parsing at query time.
- **Stack trace on ERROR only** — WARN gets empty trace. Saves storage.
- **Non-blocking channel send** — `select { case ch <- rec: default: }`. Logging never blocks business flow.

## Logs Worker — Persist Channel to DB

A platform-level background worker consumes `LogRecord` from the channel and INSERTs into `system_alerts`:

```go
// modules/shared/platform/logs/worker.go
type Worker interface {
    Start(ctx context.Context)
}

type workerImpl struct {
    repo   repository
    logIn  <-chan LogRecord
    logger Logger
}

func NewWorker(repo repository, logIn <-chan LogRecord, log Logger) Worker {
    return &workerImpl{repo: repo, logIn: logIn, logger: log}
}

func (w *workerImpl) Start(ctx context.Context) {
    ticker := time.NewTicker(12 * time.Hour)  // reaper
    defer ticker.Stop()
    w.runReaper(ctx)

    for {
        select {
        case <-ctx.Done():
            return
        case record := <-w.logIn:
            alert := &AlertEntity{
                ID:           record.ID,
                Level:        record.Level,
                Message:      record.Message,
                Context:      record.Context,
                Trace:        record.Trace,
                SourceMethod: record.SourceMethod,
                CreatedAt:    record.CreatedAt,
            }
            if err := w.repo.Save(ctx, alert); err != nil {
                w.logger.Error("[logsWorker.Start] failed to save alert", "error", err)
            }
        case <-ticker.C:
            w.runReaper(ctx)
        }
    }
}
```

DB entity:

```go
// modules/shared/platform/logs/entity.go
type AlertEntity struct {
    ID           uuid.UUID       `db:"id"`
    Level        string          `db:"level"`          // WARN | ERROR
    Message      string          `db:"message"`
    Context      json.RawMessage `db:"context"`        // key=value JSON
    Trace        string          `db:"trace"`           // stack (ERROR only)
    SourceMethod string          `db:"source_method"`   // [Type.Method]
    CreatedAt    time.Time       `db:"created_at"`
}
```

## System Alerts Table Schema

```sql
-- MySQL
CREATE TABLE system_alerts (
    id CHAR(36) NOT NULL PRIMARY KEY,
    level VARCHAR(10) NOT NULL,
    message TEXT NOT NULL,
    context JSON,
    trace TEXT,
    source_method VARCHAR(255),
    created_at TIMESTAMP NOT NULL,
    INDEX idx_level (level),
    INDEX idx_source_method (source_method),
    INDEX idx_created_at (created_at)
);
```

```sql
-- PostgreSQL
CREATE TABLE system_alerts (
    id UUID NOT NULL PRIMARY KEY,
    level VARCHAR(10) NOT NULL,
    message TEXT NOT NULL,
    context JSONB,
    trace TEXT,
    source_method VARCHAR(255),
    created_at TIMESTAMPTZ NOT NULL,
);
CREATE INDEX idx_system_alerts_level ON system_alerts (level);
CREATE INDEX idx_system_alerts_source_method ON system_alerts (source_method);
CREATE INDEX idx_system_alerts_created_at ON system_alerts (created_at);
```

```sql
-- MSSQL
CREATE TABLE system_alerts (
    id UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
    level NVARCHAR(10) NOT NULL,
    message NVARCHAR(MAX) NOT NULL,
    context NVARCHAR(MAX),           -- JSON stored as NVARCHAR(MAX) (MSSQL JSON columns)
    trace NVARCHAR(MAX),
    source_method NVARCHAR(255),
    created_at DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
CREATE INDEX idx_system_alerts_level ON system_alerts (level);
CREATE INDEX idx_system_alerts_source_method ON system_alerts (source_method);
CREATE INDEX idx_system_alerts_created_at ON system_alerts (created_at);
```

| Column | Type | Purpose |
|--------|------|---------|
| `id` | CHAR(36) / UUID | UUID V7. Time-sorted. |
| `level` | VARCHAR(10) | `"WARN"` or `"ERROR"`. DBHandler only captures these two levels. |
| `message` | TEXT | Clean message. `[Type.Method]` prefix already extracted to `source_method`. |
| `context` | JSON | Structured key=value pairs from the log call. Queryable: `context->>'$.user_id'`. |
| `trace` | TEXT | Full stack trace. Populated only for ERROR level. NULL for WARN. |
| `source_method` | VARCHAR(255) | `orderService.PlaceOrder` — extracted from `[Type.Method]` prefix. |
| `created_at` | TIMESTAMP | When the log record was created. Matches the slog record time. |

**Index rationale:**
- `idx_level` — filter: `WHERE level = 'ERROR'`. Dashboard: "show me only errors."
- `idx_source_method` — filter by service: `WHERE source_method LIKE 'orderService.%'`.
- `idx_created_at` — time-range queries and reaper cleanup.

## Reaper Retention

```sql
-- MySQL
DELETE FROM system_alerts
WHERE created_at < NOW() - INTERVAL 30 DAY;

-- PostgreSQL
DELETE FROM system_alerts
WHERE created_at < NOW() - INTERVAL '30 days';

-- MSSQL
DELETE FROM system_alerts
WHERE created_at < DATEADD(DAY, -30, SYSUTCDATETIME());
```

Default 30 days. Configurable via `cfg.LogRetentionDays`. Longer than outbox (7 days) — alerts are diagnostic and worth keeping for trend analysis.

---

## Wiring at Startup

**Production**: stdout handler + DB handler via Tee. Warn/Error go to DB.

**Development**: stdout handler only. No DB handler. No channel. No logs worker.

```go
// internal/app/builder.go
func (b *builder) setupLogger(env string) {
    stdoutHandler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelDebug})

    if env == "production" {
        // Production: dual output — console + database
        b.logChan = make(chan LogRecord, 1000)
        dbLogHandler := NewDBHandler(b.logChan)
        teeHandler := NewTeeHandler(stdoutHandler, dbLogHandler)
        b.logger = NewSlogLogger(slog.New(teeHandler)).With("app", "my-api")
    } else {
        // Development: console only. No DB overhead. No channel.
        b.logger = NewSlogLogger(slog.New(stdoutHandler)).With("app", "my-api")
    }
}

func (b *builder) setupWorkers(env string) {
    if env != "production" {
        return // no logs worker in dev
    }
    logsRepo := logs.NewRepository(b.dbPool)
    b.logsWorker = logs.NewWorker(logsRepo, b.logChan, b.logger)
    go b.logsWorker.Start(ctx)
}
```

## NoopLogger — For Tests

```go
// modules/shared/log_noop.go
type NoopLogger struct{}

func (NoopLogger) Debug(string, ...any) {}
func (NoopLogger) Info(string, ...any)  {}
func (NoopLogger) Warn(string, ...any)  {}
func (NoopLogger) Error(string, ...any) {}
func (n NoopLogger) With(...any) Logger { return n }
```

```go
// In tests
func TestPlaceOrder(t *testing.T) {
    svc := newService(repo, NoopLogger{})
    // ...
}
```

## Log Message Convention

```go
// Service — [Type.Method] prefix on every log
s.log.Error("[orderService.PlaceOrder] get user email", "user_id", input.userID, "error", err)
s.log.Warn("[orderService.PlaceOrder] low stock", "product_id", pid, "available", stock)

// Handler — method name from BaseHandler
h.Error(r.Context(), "create order", err)
// logs: ERROR [create order] error="dial tcp 10.0.1.5:5432: timeout"

// Worker — same [Type.Method] prefix
w.log.Info("[OrderReaper.run] Deleted stale orders", "count", deleted)
```

Service logs use `[receiverName.Method]`. Handler logs delegate to `BaseHandler.Error()` which adds `[methodName]` internally. Worker logs same pattern.

## Rules

| Rule | Why |
|------|-----|
| Depend on `Logger` interface, never `*slog.Logger` | Testable, swappable. No library lock-in. |
| Four levels only: Debug, Info, Warn, Error | No `Log(level)` escape. Enforces convention. |
| Key=value pairs, not format strings | Structured. Queryable. Machine-parseable. |
| `log.Error()` message is constant prefix, detail in args | Grep for `"get user email"` finds the exact call site. |
| `[Type.Method]` prefix in every log message | Source method visible in stdout AND persisted in DB column. |
| Console: JSON, all levels. DB: WARN+ERROR only | Console for dev. DB for persisted alerts without noise. |
| Non-blocking channel in DBHandler | Logging must never block business flow. Dropping a log is better than stalling. |
| Stack trace on ERROR only | WARN doesn't need it. Saves storage. |
| Inject `Logger` via constructor | Never use global logger variable. Each component receives its own. |
| `NoopLogger` in unit tests | Silence logs during test runs. |
| Reaper cleans old alerts on slow timer | Prevents unbounded `system_alerts` table growth. |

---

# Middleware

## Principle

Middleware is cross-cutting request processing. Applied at the router level, never inside a handler. chi middleware stack runs top-to-bottom before the handler, bottom-to-top after.

Place all middleware under `internal/middleware/`. One concern per file. Factory functions that take dependencies (logger, JWT manager, DB) and return `func(http.Handler) http.Handler`.

## Middleware Order

```go
// cmd/server/main.go — order is load-bearing
r := chi.NewRouter()
r.Use(middleware.RequestID)          // 1. Tag every request with a unique ID
r.Use(middleware.Recoverer(log))     // 2. Catch panics, log stack, return 500 (custom — see below)
r.Use(middleware.CloudflareRealIP)   // 3. Restore real client IP + extract geo-country
r.Use(ratelimit.Global())           // 4. Rate limit by real IP (safety net)
r.Use(middleware.RequestLogger(log)) // 5. Log request with real IP, country, duration
```

Recoverer is first after RequestID — it must wrap all downstream middleware and handlers. A panic in CloudflareRealIP, rate limiter, or any handler must be caught.
`CloudflareRealIP` MUST run before the rate limiter. If rate limiter runs first, it sees Cloudflare's edge IP and rate-limits the entire user base as a single IP.

## Recoverer

Custom panic recovery. chi's built-in `middleware.Recoverer` prints a stack trace to stdout — no structured logging, no request context. Replace with your own: log the panic via the project logger, return a sanitized 500, and guard against already-committed responses.

```go
// internal/middleware/recoverer.go
package middleware

import (
    "encoding/json"
    "net/http"
    "runtime/debug"

    "project/modules/shared"
)

// responseWriter wraps http.ResponseWriter to track whether WriteHeader was called.
// If a handler writes headers then panics, recovery must not try to write again.
type responseWriter struct {
    http.ResponseWriter
    wroteHeader bool
}

func (rw *responseWriter) WriteHeader(code int) {
    if rw.wroteHeader {
        return
    }
    rw.wroteHeader = true
    rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
    if !rw.wroteHeader {
        rw.WriteHeader(http.StatusOK)
    }
    return rw.ResponseWriter.Write(b)
}

// Recoverer catches panics, logs the stack with request context, and returns
// a sanitized 500. The client never sees stack traces or internal details.
func Recoverer(log shared.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            rw := &responseWriter{ResponseWriter: w}

            defer func() {
                if rec := recover(); rec != nil {
                    stack := debug.Stack()

                    log.Error("panic recovered",
                        "request_id", GetReqID(r.Context()),
                        "method", r.Method,
                        "path", r.URL.Path,
                        "panic", rec,
                        "stack", string(stack),
                    )

                    // Only write 500 if headers haven't been sent yet.
                    // If the handler already flushed a partial response, we can't
                    // change the status code — just log and let the connection close.
                    if !rw.wroteHeader {
                        w.Header().Set("Content-Type", "application/json")
                        w.WriteHeader(http.StatusInternalServerError)
                        _ = json.NewEncoder(w).Encode(map[string]any{
                            "success":  false,
                            "code":     http.StatusInternalServerError,
                            "message":  "Internal Server Error",
                            "trace_id": GetReqID(r.Context()),
                        })
                    }
                }
            }()
            next.ServeHTTP(rw, r)
        })
    }
}
```

No import on `modules/shared/` response types. The inline `map[string]any` keeps middleware independent. If the response envelope changes, middleware doesn't break.

### Recoverer Rules

| Rule | Why |
|------|-----|
| Custom, not chi's built-in `Recoverer` | chi's version logs to stderr with no structured fields. Custom version uses the project's `Logger` interface. |
| `shared.Logger` (interface), not `*slog.Logger` | Consistent with every other component. Injectable, testable. |
| Wrap `ResponseWriter` to track `wroteHeader` | Handler may flush headers before panicking (SSE, streaming). Recovery must not double-write. |
| Return `500 Internal Server Error` | Generic message. Never expose the panic value to the client. |
| Include `trace_id` in the response | Client can report the ID. Operator greps logs for it. |
| Inline error struct (`map[string]any`) | Middleware must not import `modules/shared/` response types. Middleware is infrastructure, not application. |
| Log full stack via `debug.Stack()` | Panics are bugs. You need the stack trace to fix them. |

## Rate Limiter

Three named tiers with fixed limits. Every route group gets one tier. A global safety-net limiter catches ungrouped routes.

```go
// internal/middleware/ratelimit/ratelimit.go
package ratelimit

import (
    "encoding/json"
    "net/http"
    "time"

    "github.com/go-chi/chi/v5/middleware"
    "github.com/go-chi/httprate"
)

// Tier limits (requests per minute per IP).
const (
    strict   = 5   // sensitive ops: login, register, password reset
    standard = 30  // authenticated reads, admin, OAuth, lookups
    relaxed  = 60  // public GET endpoints
    global   = 120 // catch-all safety net
)

// limitHandler returns the standard ApiResponse envelope on rate limit hit.
func limitHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("Retry-After", "60")
    w.WriteHeader(http.StatusTooManyRequests)
    _ = json.NewEncoder(w).Encode(ApiResponse{
        Success: false,
        Code:    http.StatusTooManyRequests,
        Message: "Rate limit exceeded. Please try again later.",
        TraceID: middleware.GetReqID(r.Context()),
    })
}

func Strict() func(http.Handler) http.Handler {
    return httprate.Limit(strict, 1*time.Minute,
        httprate.WithKeyByIP(), httprate.WithLimitHandler(limitHandler))
}
func Standard() func(http.Handler) http.Handler {
    return httprate.Limit(standard, 1*time.Minute,
        httprate.WithKeyByIP(), httprate.WithLimitHandler(limitHandler))
}
func Relaxed() func(http.Handler) http.Handler {
    return httprate.Limit(relaxed, 1*time.Minute,
        httprate.WithKeyByIP(), httprate.WithLimitHandler(limitHandler))
}

// Global is the safety-net. Applied once at router root.
// Route groups override this with their own tier.
func Global() func(http.Handler) http.Handler {
    return httprate.Limit(global, 1*time.Minute,
        httprate.WithKeyByIP(), httprate.WithLimitHandler(limitHandler))
}
```

`httprate.WithKeyByIP()` keys on `r.RemoteAddr`. Without a real-IP middleware before it, every request behind a proxy shares the same bucket. Order matters.

### Tier Assignment

```go
// cmd/server/main.go
r := chi.NewRouter()
r.Use(ratelimit.Global())  // safety net: 120 req/min, catches every route

// modules/auth/module.go — strict tier overrides global
func (m *Module) RegisterRoutes(r chi.Router) {
    r.Route("/v1/auth", func(r chi.Router) {
        r.With(ratelimit.Strict()).Post("/login", m.handler.Login)
        r.With(ratelimit.Strict()).Post("/register", m.handler.Register)
        r.With(ratelimit.Standard()).Post("/google", m.handler.GoogleLogin)
    })
}

// modules/pvp/module.go — relaxed for public read-heavy endpoints
func (m *Module) RegisterRoutes(r chi.Router) {
    r.Route("/v1/pvp", func(r chi.Router) {
        r.With(ratelimit.Relaxed()).Get("/events", m.handler.GetEvents)
        r.With(ratelimit.Relaxed()).Get("/hall-of-fame", m.handler.GetHallOfFame)
    })
}
```

chi applies the innermost (route-level) limiter first. The global limiter acts only when no route-level limiter was set. Every route is covered.

### Rate Limiter Rules

| Rule | Why |
|------|-----|
| Three named tiers: Strict, Standard, Relaxed | No per-endpoint magic numbers. Consistent across the app. |
| Global limiter at router root | Safety net. No unprotected route. |
| Route groups override with `.With(tier)` | Specific beats general. Auth gets Strict even though Global is loose. |
| Rate limit by IP, not user ID | Works for unauthenticated endpoints. No side-channel from failed auth. |
| `Retry-After: 60` header on 429 | Client can back off automatically. |
| Standard `ApiResponse` envelope on 429 | Same shape as every other error. Client code doesn't branch. |

## Real IP — Cloudflare + Reverse Proxy

When behind Cloudflare, `RemoteAddr` is always Cloudflare's edge IP — rate limiting becomes useless. `CloudflareRealIP` restores the true client IP from Cloudflare's un-spoofable header and extracts geo-country for location-aware logic. Falls back gracefully when Cloudflare is absent (dev, direct connection).

```go
// internal/middleware/realip.go
package middleware

func CloudflareRealIP(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()

        // 1. Extract geo-country from Cloudflare header (ISO 3166-1 alpha-2).
        //    XX = unknown, T1 = Tor — skip both.
        if country := r.Header.Get("CF-IPCountry"); country != "" {
            upper := strings.ToUpper(country)
            if upper != "XX" && upper != "T1" {
                ctx = context.WithValue(ctx, GeoCountryKey, upper)
            }
        }

        // 2. Restore real client IP. Priority: CF → X-Real-IP → X-Forwarded-For → RemoteAddr.
        if cfIP := r.Header.Get("CF-Connecting-IP"); cfIP != "" {
            r.RemoteAddr = cfIP  // un-spoofable: Cloudflare strips client-set copies
        } else if realIP := r.Header.Get("X-Real-IP"); realIP != "" {
            r.RemoteAddr = realIP
        } else if xff := r.Header.Get("X-Forwarded-For"); xff != "" {
            if i := strings.Index(xff, ","); i > 0 {
                r.RemoteAddr = strings.TrimSpace(xff[:i])
            } else {
                r.RemoteAddr = strings.TrimSpace(xff)
            }
        }
        // else: keep original RemoteAddr (direct connection, local dev)

        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func GetGeoCountry(ctx context.Context) (string, bool) {
    val, ok := ctx.Value(GeoCountryKey).(string)
    return val, ok
}
```

### Header Priority

| Header | Source | Spoofable? | Purpose |
|--------|--------|-----------|---------|
| `CF-Connecting-IP` | Cloudflare edge | No — Cloudflare strips client-set copies | Real client IP |
| `CF-IPCountry` | Cloudflare edge | No | ISO 3166-1 country code |
| `X-Real-IP` | Reverse proxy (nginx, Traefik) | Weak — trusts proxy | Fallback when no Cloudflare |
| `X-Forwarded-For` | Reverse proxy chain | Weak — client can inject fake leftmost IP | Last-resort fallback |

### Using Geo-Country in a Handler

```go
func (h *OrderHandler) Create(w http.ResponseWriter, r *http.Request) {
    country, ok := middleware.GetGeoCountry(r.Context())
    if ok && country == "IR" {
        h.Forbidden(w, r, "Service unavailable in your region")
        return
    }
}
```

No external geo-IP database. No API call latency. Cloudflare does the lookup at the edge for free. The middleware just reads the header.

### Real IP Rules

| Rule | Why |
|------|-----|
| `CloudflareRealIP` before rate limiter | Rate limiter must see real client IP, not Cloudflare's edge IP. |
| `CF-Connecting-IP` as primary IP source | Cloudflare strips client-set copies. Un-spoofable. |
| `CF-IPCountry` through context, not header read | Handler calls `GetGeoCountry(ctx)`, not `r.Header.Get(...)`. Middleware is the single parser. |
| Graceful fallback when Cloudflare absent | Same code works in dev (direct), staging (reverse proxy), and production (Cloudflare). |

---

# Configuration

## Principle

One place loads every env var. One struct holds it. Modules pull what they need from the config — they never call `os.Getenv` directly.

## `shared/config/config.go` — Single Config Source

```go
// modules/shared/config/config.go
package config

import (
    "os"
    "strconv"
    "strings"
)

// Config holds all application configuration. One struct, one Load().
// Every env var the app reads is listed here. Nothing else calls os.Getenv.
type Config struct {
    AppPort    string
    LogLevel   string
    DatabaseURL string
    DatabasePool PoolConfig
    JWTSecret   string
    // ... add other env vars here as the app grows
}

type PoolConfig struct {
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetimeSec int
}

// Load reads all env vars once. Fallbacks are safe defaults for local dev.
// Secrets (JWT_SECRET, DATABASE_URL) have no fallback — must be set in prod.
func Load() *Config {
    return &Config{
        AppPort:     getEnv("APP_PORT", "8080"),
        LogLevel:    getEnv("LOG_LEVEL", "debug"),
        DatabaseURL: os.Getenv("DATABASE_URL"),
        DatabasePool: PoolConfig{
            MaxOpenConns:        getEnvInt("DB_MAX_OPEN_CONNS", 25),
            MaxIdleConns:        getEnvInt("DB_MAX_IDLE_CONNS", 10),
            ConnMaxLifetimeSec:  getEnvInt("DB_CONN_MAX_LIFETIME_SEC", 300),
        },
        JWTSecret: os.Getenv("JWT_SECRET"),
    }
}

// Validate panics at startup if required secrets are missing.
// Fail-fast is better than nil-pointer at runtime.
func (c *Config) Validate() {
    if c.DatabaseURL == "" {
        panic("config: DATABASE_URL is required")
    }
    if c.JWTSecret == "" {
        panic("config: JWT_SECRET is required")
    }
}

// Helpers. Unexported — only config.go uses them.

func getEnv(key, fallback string) string {
    if val := os.Getenv(key); val != "" {
        return val
    }
    return fallback
}

func getEnvInt(key string, fallback int) int {
    val := os.Getenv(key)
    if val == "" {
        return fallback
    }
    n, err := strconv.Atoi(val)
    if err != nil {
        return fallback
    }
    return n
}
```

## No Env Scattering

- **`os.Getenv` appears exactly once** — inside `config.Load()`. Nowhere else.
- **Modules don't read env vars** — they receive values from `Config`.
- **Validate at startup** — `cfg.Validate()` panics before the server starts. No runtime surprise.

## Registration

`Config` is created in `cmd/` and stored in the container. Modules pull what they need from it — they never call `config.Load()` themselves.

```go
// cmd/server/main.go
func main() {
    cfg := config.Load()
    cfg.Validate()

    db := connectDB(cfg.DatabaseURL, cfg.DatabasePool)
    c := shared.NewContainer(db, cfg)

    user.Register(c)
    order.Register(c)
    // ...
}
```

```go
// modules/order/module.go
func Register(c *shared.Container) {
    // Pull only what this module needs from config
    uow := db.NewUnitOfWork(c.DB)
    repo := newRepository(c.DB)
    svc := newService(repo, uow, c.Config.JWTSecret, c.Config.LogLevel)
    c.Set(ports.Key, svc)
}
```

## Container Update

```go
// modules/shared/container.go
type Container struct {
    DB       *sqlx.DB
    Config   *config.Config
    EventBus *EventBus
    Logger   Logger
    // ...
}
```

---

# Unit of Work (Transaction Management)

## Principle

One `UnitOfWork` per module. Module never opens a transaction that reaches into another module's tables. Cross-module consistency uses events/sagas, not shared DB transactions.

## Shared UoW Interface

Lives in `shared/db/`. Generic. Thin. No ORM abstraction.

```go
// modules/shared/db/transactor.go
package db

import "context"

type UnitOfWork[T any] interface {
    Execute(ctx context.Context, fn func(tx T) error) error
}
```

## Implementation

```go
// modules/shared/db/sqlx.go
package db

import (
    "context"
    "fmt"

    "github.com/jmoiron/sqlx"
)

type sqlxUnitOfWork struct {
    db *sqlx.DB
}

func NewUnitOfWork(db *sqlx.DB) UnitOfWork[*sqlx.Tx] {
    return &sqlxUnitOfWork{db: db}
}

func (u *sqlxUnitOfWork) Execute(ctx context.Context, fn func(tx *sqlx.Tx) error) error {
    tx, err := u.db.BeginTxx(ctx, nil)
    if err != nil {
        return fmt.Errorf("db: begin tx: %w", err)
    }

    defer func() {
        if p := recover(); p != nil {
            _ = tx.Rollback()
            panic(p)
        }
    }()

    if err := fn(tx); err != nil {
        _ = tx.Rollback()
        return err
    }

    if err := tx.Commit(); err != nil {
        return fmt.Errorf("db: commit tx: %w", err)
    }

    return nil
}
```

## Per-Module UoW Registration

Each module creates its own `UnitOfWork` in `module.go`:

```go
// modules/order/module.go
func Register(c *shared.Container) {
    uow := db.NewUnitOfWork(c.DB)
    repo := newRepository(c.DB) // repo just holds *sqlx.DB, see below
    svc := newService(repo, uow, userAPI, paymentAPI)
    c.Set(ports.Key, svc)
}
```

## Repository — Just SQL

Repository is a dumb bag of SQL queries. No DTO conversion. No abstraction. No "findByX / save / update" template. Just the queries the module actually needs, named for what they do.

### Named Parameters — Always

Every query uses named parameters (`:field_name`), never positional (`$1, $2`). Positional params break when columns are added or reordered. Named params are self-documenting and survive schema changes.

```go
// Wrong — positional. Fragile. What is $3?
tx.ExecContext(ctx,
    "INSERT INTO orders (id, user_id, status) VALUES ($1, $2, $3)",
    entity.ID, entity.UserID, entity.Status)

// Correct — named. Survives column reordering.
tx.NamedExecContext(ctx,
    "INSERT INTO orders (id, user_id, status) VALUES (:id, :user_id, :status)", entity)
```

### Params from Entity — Yes, Always

Named params map directly from the entity struct. Every column in the query binds to a field of the entity struct passed as the last argument. This is the whole point of `sqlx.NamedExecContext` — the struct becomes the source of truth for parameter values.

**Single-entity writes** — pass the entity directly:

```go
func (r *repository) insertOrder(ctx context.Context, tx *sqlx.Tx, e domain.orderEntity) error {
    _, err := tx.NamedExecContext(ctx,
        "INSERT INTO orders (id, user_id, status, total, created_at, updated_at)
         VALUES (:id, :user_id, :status, :total, :created_at, :updated_at)", e)
    return err
}
```

**Multi-entity batch writes** — loop, each entity struct passed:

```go
func (r *repository) insertItems(ctx context.Context, tx *sqlx.Tx, items []domain.orderItemEntity) error {
    for i := range items {
        if _, err := tx.NamedExecContext(ctx,
            "INSERT INTO order_items (id, order_id, product_id, quantity, unit_price)
             VALUES (:id, :order_id, :product_id, :quantity, :unit_price)", items[i]); err != nil {
            return err
        }
    }
    return nil
}
```

**Writes with a mix of entity fields + external values** — use a small anonymous struct at the call site:

```go
func (r *repository) insertOrderWithRequestID(ctx context.Context, tx *sqlx.Tx, e domain.orderEntity, requestID string) error {
    _, err := tx.NamedExecContext(ctx,
        "INSERT INTO orders (id, user_id, status, request_id)
         VALUES (:id, :user_id, :status, :request_id)",
        struct {
            ID        string
            UserID    string `db:"user_id"`
            Status    string
            RequestID string `db:"request_id"`
        }{e.ID, e.UserID, e.Status, requestID})
    return err
}
```

**Reads** — scan directly into entity struct, named params for WHERE:

```go
func (r *repository) getByID(ctx context.Context, tx *sqlx.Tx, id string) (domain.orderEntity, error) {
    var e domain.orderEntity
    err := tx.GetContext(ctx, &e, "SELECT * FROM orders WHERE id = :id",
        map[string]any{"id": id})
    return e, err
}

func (r *repository) getByUserID(ctx context.Context, tx *sqlx.Tx, userID string) ([]domain.orderEntity, error) {
    var rows []domain.orderEntity
    err := tx.SelectContext(ctx, &rows,
        "SELECT * FROM orders WHERE user_id = :user_id ORDER BY created_at DESC",
        map[string]any{"user_id": userID})
    return rows, err
}
```

For single-param queries, `?` is acceptable — it's unambiguous. For multi-param queries, named params are mandatory.

```go
// Single param — ? is fine
tx.GetContext(ctx, &e, "SELECT * FROM orders WHERE id = ?", id)

// Multi param — named always
tx.SelectContext(ctx, &rows,
    "SELECT * FROM orders WHERE user_id = :user_id AND status = :status ORDER BY created_at DESC",
    map[string]any{"user_id": userID, "status": status})
```

### Rules

| Rule | Why |
|------|-----|
| Named params (`:field`) for all queries | Survives column reordering. Self-documenting. |
| Entity struct as param source for writes | `sqlx` maps `:column_name` → struct field by `db` tag. Zero manual binding. |
| Anonymous struct for mixed params | When entity + external values needed. Explicit, one place. |
| `map[string]any` for reads with multiple params | Cleaner than anon struct for read-only params. |
| `?` OK for single-param reads | Unambiguous. Don't over-engineer. |
| **No interface** — `repository` is a concrete struct | Only the module's own service uses it. Mock at service level, not repo. |
| **No DTO conversion** — repo returns/accepts `entity` types directly | Service converts entity ↔ DTO. Repo stays dumb. |
| **Tx always explicit** — every method takes `*sqlx.Tx` | Caller controls transaction boundary. |
| **Named by operation** — `insertOrder`, `getByID`, `updateStatus` | Not `save`, `find`, `update`. |
| **No generic CRUD** — if the module doesn't need `deleteOrder`, don't write it | Dead code rots. |

## Service — Where UoW Meets Business Logic

Service owns the transaction boundary. It calls repo methods passing the tx.

```go
// modules/order/service.go
func (s *orderService) PlaceOrder(ctx context.Context, input orderDTO) (orderDTO, error) {
    // Validate outside tx
    if err := input.validate(); err != nil {
        return orderDTO{}, err
    }

    var result orderDTO

    err := s.uow.Execute(ctx, func(tx *sqlx.Tx) error {
        // Build entities
        entity := input.toEntity() // request/entity mapping lives in entity.go or service

        // Insert order header
        if err := s.repo.insertOrder(ctx, tx, entity); err != nil {
            return fmt.Errorf("insert order: %w", err)
        }

        // Insert line items
        if err := s.repo.insertItems(ctx, tx, entity.items); err != nil {
            return fmt.Errorf("insert items: %w", err)
        }

        result = entity.toDTO()
        return nil
    })
    if err != nil {
        return orderDTO{}, err
    }

    // Event after commit
    s.eventBus.Emit(OrderPlaced{OrderID: result.id, ...})
    return result, nil
}
```

## When NOT to Use UoW

| Scenario | Why Skip |
|----------|----------|
| Single read query | Just call `r.db.GetContext(ctx, ...)` — no tx needed |
| Cross-module atomicity | Never share a tx between modules. Use events/sagas. |
| No DB writes in flow | UoW is for writes. Reads don't need it. |
| Idempotent operation | If safe to retry, skip UoW overhead. |

## Service Rules

- **One UoW per service method** — don't nest UoW calls. If you need nesting, the boundary is wrong.
- **Validate before tx** — no DB work in validation. Fail fast.
- **Side effects after tx** — events, emails, notifications fire after commit succeeds.
- **External calls inside tx are OK** — if the external call fails, tx rolls back. If tx commit fails after external call succeeds, compensation runs via events/sagas.

---

# Cross-Module Communication

## Allowed Mechanisms

1. **Direct function call** via `ports/` — for synchronous, same-process operations.
2. **Domain events** — for asynchronous, fire-and-forget operations.
3. **API/SDK contracts** — a module's `ports/` IS its SDK.

## Forbidden Mechanisms

- **Shared database tables** — each module owns its tables. No joins across modules.
- **Importing internal files** — only `ports/` is public.
- **Global state** — no module-level `init()` that mutates global vars.

## When to Use Events vs Direct Calls

| Scenario | Mechanism |
|----------|-----------|
| Order needs user email to render confirmation | Direct call: `userAPI.GetEmail(ctx, userID)` |
| Order placed → send notification | Event: `events.Emit(OrderPlaced{...})`, notification module subscribes |
| Payment completed → update order status | Direct call OR saga orchestration |
| User deleted → cleanup orders, payments, notifications | Event (fan-out) |

## Event Contract

Events are types defined in the publishing module's `domain/`:

```go
// modules/order/domain/events.go
package domain

type OrderPlaced struct {
    OrderID   string
    UserID    string
    Total     shared.Money
    Timestamp time.Time
}
```

Subscribing modules import the event type from the publisher's `domain/`. The event bus/router lives in `shared`. The event bus/router lives in `shared`.

---

# Per-Module Dependency Injection

## Core Principle

**Each module owns its wiring.** No module's dependencies are assembled in `cmd/` or some central `app.go`. The composition root only calls `module.Register(...)` — it never knows how a module is built.

This means:
- A module's `module.go` declares what it **provides** and what it **needs**.
- The composition root provides infrastructure (DB, config, event bus). Modules wire themselves from there.
- Adding a module never requires editing a central wiring file beyond one `Register` call.

## `module.go` — The Self-Contained DI Unit

Every module has a `module.go` with a single exported function: `Register`. This function:

1. **Declares** its dependency interfaces (defined by other modules in their `ports/`).
2. **Resolves** them from the DI container.
3. **Constructs** its own service, repository, handlers.
4. **Subscribes** to events it cares about.
5. **Returns** nothing — it registers itself into the app.

```go
// modules/order/module.go
package order

import (
    "project/modules/shared"
    userPorts "project/modules/user/ports"
    paymentPorts "project/modules/payment/ports"
)

// Register wires the order module into the application.
// Called once at startup from the composition root.
func Register(c *shared.Container) {
    // Resolve what we need from other modules — by exported Key, no magic strings
    userAPI := c.MustGet[userPorts.Service](c, userPorts.Key)
    paymentAPI := c.MustGet[paymentPorts.Service](c, paymentPorts.Key)

    // Build our internals
    repo := newRepository(c.DB)
    svc := newService(repo, userAPI, paymentAPI)

    // Register what we provide for other modules
    c.Set(ports.Key, svc)

    // Subscribe to events we care about
    c.EventBus.Subscribe(payment.PaymentCompleted{}, svc.handlePaymentCompleted)
}
```

## Dependency Interfaces

Dependencies on other modules are declared as **interfaces in the dependent module**, satisfied by the provider module's exported `ports.Service`:

```go
// modules/order/ports/repository.go — written by order module
package ports

type UserService interface {
    GetEmail(ctx context.Context, userID string) (string, error)
}

type PaymentService interface {
    Charge(ctx context.Context, input ChargeInput) (*ChargeResult, error)
}
```

The consumer **owns** the interfaces it needs. The provider module's `ports.Service` satisfies them implicitly. This is DIP at module scale — the consumer never imports the provider's concrete implementation. The interface is a subset of what the provider can do — the consumer declares only what it actually calls.

## `shared/container.go` — The DI Container

A minimal, type-safe container lives in `shared/`:

```go
// modules/shared/container.go
package shared

import (
    "database/sql"
    "fmt"
    "sync"
)

// Container holds all registered module services and infrastructure.
// Keys follow the pattern "<module>.ports": "user.ports", "order.ports".
type Container struct {
    DB       *sql.DB
    Config   *AppConfig
    EventBus *EventBus
    Logger   Logger

    mu     sync.RWMutex
    items  map[string]any
}

func NewContainer(db *sql.DB, cfg *AppConfig) *Container {
    return &Container{
        DB:       db,
        Config:   cfg,
        EventBus: NewEventBus(),
        Logger:   NewLogger(cfg.LogLevel),
        items:    make(map[string]any),
    }
}

// Set registers a service by key.
func (c *Container) Set(key string, value any) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = value
}

// Get retrieves a service by key. Returns nil if not found.
func (c *Container) Get(key string) any {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.items[key]
}

// MustGet retrieves a service by key. Panics if not found — only for startup.
func MustGet[T any](c *Container, key string) T {
    val := c.Get(key)
    if val == nil {
        panic(fmt.Sprintf("container: key %q not registered — check module registration order", key))
    }
    typed, ok := val.(T)
    if !ok {
        panic(fmt.Sprintf("container: key %q is %T, not %T", key, val, typed))
    }
    return typed
}
```

Key design decisions:
- **No magic strings** — each module exports its key as a constant in `ports/`: `const Key = "order.ports"`. Other modules use `orderPorts.Key`. Compile-time safe.
- **`MustGet` panics at startup** — missing dependency caught immediately, not nil-pointer at runtime.
- **No reflection-based auto-wiring** — explicit, grep-friendly, no magic.
- **Type-safe generic getter** — `MustGet[orderPorts.Service](c, orderPorts.Key)` fails at compile time if types don't match.

## Registration Order

Modules must be registered in dependency order. `MustGet` panics if a dependency isn't registered yet:

```go
// cmd/server/main.go
func main() {
    db := connectDB()
    cfg := shared.LoadConfig()

    c := shared.NewContainer(db, cfg)
    r := chi.NewRouter()

    // Leaf modules first (no module dependencies)
    user.Register(c, r)
    payment.Register(c, r)

    // Dependent modules next
    order.Register(c, r) // depends on user, payment

    // Most dependent last
    notification.Register(c, r) // depends on order, payment

    // Start HTTP server
    http.ListenAndServe(":"+cfg.AppPort, r)
}
```

The registration order IS the dependency graph. No hidden cycles. If `order.Register` calls `MustGet[userPorts.Service](c, userPorts.Key)` before `user.Register` ran → panic at startup with a clear message.

## Event Subscriptions

Event subscriptions live in `module.go`, not in `cmd/`:

```go
// modules/notification/module.go
func Register(c *shared.Container) {
    orderAPI := c.MustGet[orderPorts.Service](c, orderPorts.Key)
    repo := newRepository(c.DB)
    svc := newService(repo, orderAPI)

    c.Set(ports.Key, svc)

    // I subscribe to events — I own my subscriptions
    c.EventBus.Subscribe(order.OrderPlaced{}, svc.handleOrderPlaced)
    c.EventBus.Subscribe(payment.PaymentCompleted{}, svc.handlePaymentCompleted)
}
```

The publishing module defines the event type. The subscribing module registers its handler. `cmd/` never touches event wiring.

## Full Module Example

```
modules/order/
  ports/
    service.go          # Service interface — inbound port
    repository.go       # Repository interface — outbound port
  adapters/
    http/
      handler.go        # OrderHandler
      request.go        # CreateOrderRequest, CancelOrderRequest
      response.go       # OrderResponse, OrderItemResponse
      routes.go         # RegisterRoutes(r, h)
    postgres/
      repository.go     # SQL implementation of ports.Repository
    worker/             # Background workers (secondary adapter)
      reaper.go         # struct orderReaper → periodic cleanup
  domain/
    entity.go           # orderEntity, orderItemEntity (gorm/sqlx tags, unexported)
    dto.go              # orderDTO, orderItemDTO (no tags, unexported)
    errors.go           # Domain error types (exported)
    events.go           # OrderPlaced, OrderCancelled (exported)
  module.go             # Register(c, r) — wires ports → adapters
```

```go
// modules/order/ports/service.go
package ports

import (
    "context"
    "project/modules/order/domain"
)

// Service is the module's inbound port. adapters/http handlers call it.
// Other modules call this via the container.
type Service interface {
    PlaceOrder(ctx context.Context, input domain.CreateOrderDTO) (domain.OrderDTO, error)
    GetOrder(ctx context.Context, orderID string) (domain.OrderDTO, error)
    CancelOrder(ctx context.Context, orderID string) (domain.OrderDTO, error)
}
```

```go
// modules/order/ports/service.go — container key constant
package ports

// Key is the container key for this module's service.
// Other modules use this constant — no magic strings.
const Key = "order.ports"
```

```go
// modules/order/domain/entity.go
package domain

import "time"

// orderEntity maps to the "orders" table. Unexported — never leaves this module.
type orderEntity struct {
    ID        string    `db:"id"`
    UserID    string    `db:"user_id"`
    Status    string    `db:"status"`
    Total     int64     `db:"total"` // cents
    CreatedAt time.Time `db:"created_at"`
    UpdatedAt time.Time `db:"updated_at"`
}

type orderItemEntity struct {
    ID        string `db:"id"`
    OrderID   string `db:"order_id"`
    ProductID string `db:"product_id"`
    Quantity  int    `db:"quantity"`
    UnitPrice int64  `db:"unit_price"` // cents
}

func (e *orderEntity) toDTO() orderDTO { ... }
func (e *orderItemEntity) toDTO() orderItemDTO { ... }
```

```go
// modules/order/adapters/http/request.go
package http

// CreateOrderRequest — API input shape. Exported.
type CreateOrderRequest struct {
    UserID string            `json:"user_id" validate:"required,uuid"`
    Items  []OrderItemRequest `json:"items" validate:"required,min=1,dive"`
}

type OrderItemRequest struct {
    ProductID string `json:"product_id" validate:"required"`
    Quantity  int    `json:"quantity" validate:"required,min=1"`
}

type CancelOrderRequest struct {
    OrderID string `json:"order_id" validate:"required,uuid"`
    Reason  string `json:"reason" validate:"required,min=1,max=500"`
}

func (r CreateOrderRequest) toDTO() domain.CreateOrderDTO { ... }
```

```go
// modules/order/adapters/http/response.go
package http

// OrderResponse — API output shape. Exported.
type OrderResponse struct {
    ID        string              `json:"id"`
    UserID    string              `json:"user_id"`
    Status    string              `json:"status"`
    Items     []OrderItemResponse `json:"items"`
    Total     string              `json:"total"` // "12.50 USD"
    CreatedAt string              `json:"created_at"` // ISO 8601
}

type OrderItemResponse struct {
    ProductID string `json:"product_id"`
    Quantity  int    `json:"quantity"`
    UnitPrice string `json:"unit_price"` // "3.99 USD"
}

func fromDTO(d domain.OrderDTO) OrderResponse { ... }
```

```go
// modules/order/domain/dto.go
package domain

// DTO types are internal transfer objects. No JSON tags. No DB tags. Unexported.
type orderDTO struct {
    id     string
    userID string
    status orderStatus
    items  []orderItemDTO
    total  shared.Money
}

type orderItemDTO struct {
    productID string
    quantity  int
    unitPrice shared.Money
}
```

```go
// modules/order/module.go
package order

import (
    "project/modules/order/adapters/http"
    "project/modules/order/adapters/postgres"
    "project/modules/order/ports"
    "project/modules/shared"

    userPorts "project/modules/user/ports"
    paymentPorts "project/modules/payment/ports"
)

// Register wires the order module.
// Dependencies: shared, user.ports, payment.ports
// Provides: order.ports
func Register(c *shared.Container, r chi.Router) {
    // Resolve dependencies by exported key constants. No magic strings.
    userAPI := shared.MustGet[userPorts.Service](c, userPorts.Key)
    paymentAPI := shared.MustGet[paymentPorts.Service](c, paymentPorts.Key)

    repo := postgres.NewRepository(c.DB)
    svc := newService(repo, userAPI, paymentAPI, c.EventBus)

    c.Set(ports.Key, svc)
    c.EventBus.Subscribe(payment.PaymentCompleted{}, svc.handlePaymentCompleted)

    // Mount routes
    handler := httpAdapter.NewHandler(svc, c.Logger)
    httpAdapter.RegisterRoutes(r, handler)
}
```

```go
// cmd/server/main.go
func main() {
    db := connectDB()
    cfg := shared.LoadConfig()
    c := shared.NewContainer(db, cfg)
    r := chi.NewRouter()

    shared.Register(c)
    user.Register(c, r)
    payment.Register(c, r)
    order.Register(c, r)
    notification.Register(c, r)

    http.ListenAndServe(":"+cfg.AppPort, r)
}
```

The composition root is flat, declarative, and trivially auditable. Every module's wiring lives in its own `module.go`.

## File Cheat Sheet

| File | Package | Exported? | Tags? | Contains |
|------|---------|----------|-------|----------|
| `ports/service.go` | `ports` | Interface | No | Service interface + container Key constant |
| `ports/repository.go` | `ports` | Interface | No | Repository interface |
| `domain/entity.go` | `domain` | No | gorm/sqlx | DB table structs, `toDTO()` methods |
| `domain/dto.go` | `domain` | No | None | Internal transfer objects |
| `domain/errors.go` | `domain` | Yes | No | Error types implementing DomainError |
| `domain/events.go` | `domain` | Yes | json | Domain event structs |
| `adapters/http/handler.go` | `http` | Unexported struct | No | Handler — calls ports.Service |
| `adapters/http/request.go` | `http` | Yes | json + validate | API input types |
| `adapters/http/response.go` | `http` | Yes | json | API output types, `fromDTO()` |
| `adapters/http/routes.go` | `http` | RegisterRoutes | No | Route registration |
| `adapters/postgres/repository.go` | `postgres` | Unexported struct | No | SQL queries implementing ports.Repository |
| `adapters/worker/*.go` | `worker` | Unexported struct | No | Background workers (secondary adapter) |
| `module.go` | `order` | Only `Register` | No | DI wiring, event subscriptions, route mounting |

---

# Background Workers

## Principle

Workers are long-running goroutines that react to triggers — timers, channels, queues, file watchers. They belong to the module that owns the business logic they execute. Never a central `workers/` package disconnected from domain code.

## Worker vs Scheduler vs Reaper — Naming

| Suffix | Trigger | Example |
|--------|---------|---------|
| `Worker` | External input (queue poll, channel consume, file tail) | `OutboxWorker`, `LogsWorker`, `ScrapeWorker` |
| `Scheduler` | Timer tick — periodic state transitions at fixed interval | `PvpScheduler`, `BillingScheduler` |
| `Reaper` | Timer tick — periodic cleanup/deletion | `TokenReaper`, `SessionReaper` |

Choose the suffix by what the thing reacts to, not what it does. A `TokenReaper` deletes expired tokens — nobody named it `TokenCleanupScheduler`. The name tells you the trigger + role at a glance.

## Where Workers Live

Workers are secondary adapters — driven by the module, not driving it. Same tier as `adapters/postgres/`. They live in `adapters/worker/`.

### One File Per Worker — Named After the Struct

Each worker gets its own file. File name is the **lowercase struct name**. No `worker_` prefix. No mega-file `worker.go` containing unrelated structs.

```
modules/order/
  adapters/
    http/                  # Primary adapter
    postgres/              # Secondary adapter — DB
    worker/                # Secondary adapter — background workers
      outbox_worker.go     # struct outboxWorker → polls outbox, dispatches
      reaper.go            # struct orderReaper → periodic cleanup
      event_worker.go      # struct eventWorker → consumes channel events
```

`ports/worker.go` is NOT per-module. The `Worker` interface (`Start(ctx context.Context)`) is shared across all modules — it lives once in `modules/shared/`. Modules that need a producer/consumer contract define that specific interface in their own `ports/`.

**Why the file is named after the struct, not the role**: When there are 3+ workers in a module, opening a flat directory of `outbox_worker.go`, `reaper.go`, `event_worker.go` tells you every worker at a glance. If they were all stuffed into `worker.go`, you'd scan a 400-line file hunting for struct definitions. One file per worker means one file per concern.

### Platform-Level Workers

Infrastructure workers that serve all modules (outbox dispatcher, log ingester) live in `shared/platform/`. They get their own directory — one concern, one package.

## Worker Interface

```go
// ports/worker.go
package ports

import "context"

// Worker is a long-running background process.
// Start blocks until ctx is cancelled, then drains and returns.
type Worker interface {
    Start(ctx context.Context)
}
```

`Start(ctx)` — not `Run() error`. Worker runs forever. When to stop? Context cancellation. Return value? None — the only failure mode is "log and keep running." If a worker must signal fatal failure, it panics and the recovery middleware restarts it.

## Registration — `StartWorkers(ctx)` on Module

Each module exposes one method that starts all its workers. `cmd/` calls it once at startup.

```go
// modules/order/module.go
type Module struct {
    handler  *http.OrderHandler
    worker   *worker.OutboxWorker
    reaper   *worker.OrderReaper
}

func Register(c *shared.Container, r chi.Router) *Module {
    // ... wire service, repo, handler ...
    outboxWorker := worker.NewOutboxWorker(repo, c.EventBus, c.Logger)
    orderReaper := worker.NewOrderReaper(repo, c.Logger)

    return &Module{
        handler: handler,
        worker:  outboxWorker,
        reaper:  orderReaper,
    }
}

func (m *Module) StartWorkers(ctx context.Context) {
    go m.worker.Start(ctx)
    go m.reaper.Start(ctx)
}
```

## Wiring in `cmd/`

```go
// cmd/server/main.go
func main() {
    // ... setup ...
    orderMod := order.Register(c, r)
    paymentMod := payment.Register(c, r)

    // Platform workers — not owned by any business module
    go outboxWorker.Start(ctx)
    go logsWorker.Start(ctx)

    // Module workers — each module starts its own
    orderMod.StartWorkers(ctx)
    paymentMod.StartWorkers(ctx)

    // Block until shutdown
    http.ListenAndServe(":"+cfg.AppPort, r)
}
```

## Graceful Shutdown

Every worker honors context cancellation. No `time.Sleep` without a `select` on `ctx.Done()`. No infinite loops without a done channel.

```go
// adapters/worker/reaper.go
func (w *OrderReaper) Start(ctx context.Context) {
    w.log.Info("[OrderReaper.Start] Starting stale order reaper")

    ticker := time.NewTicker(15 * time.Minute)
    defer ticker.Stop()

    // Initial run on startup
    w.run(ctx)

    for {
        select {
        case <-ctx.Done():
            w.log.Info("[OrderReaper.Start] Shutting down")
            return
        case <-ticker.C:
            w.run(ctx)
        }
    }
}

func (w *OrderReaper) run(ctx context.Context) {
    deleted, err := w.repo.DeleteStaleOrders(ctx, 7*24*time.Hour)
    if err != nil {
        w.log.Error("[OrderReaper.run] Failed to delete stale orders", "error", err)
        return
    }
    if deleted > 0 {
        w.log.Info("[OrderReaper.run] Reaped stale orders", "count", deleted)
    }
}
```

## Channel-Based Workers

Workers that consume from a channel rather than polling a timer:

```go
// adapters/worker/event_worker.go
func (w *EventWorker) Start(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        case event := <-w.eventCh:
            w.process(ctx, event)
        }
    }
}
```

No ticker. The channel is the trigger. `eventCh` is populated by whatever produces events — a queue consumer, a file watcher, or another goroutine.

## Worker with Registered Handlers

When a worker dispatches to multiple handler types (like outbox pattern):

```go
// The outbox worker doesn't know what handlers exist.
// Handlers register themselves by message type.
type Handler interface {
    Type() string
    Execute(ctx context.Context, msg MessageEntity) error
}

func (w *OutboxWorker) RegisterHandler(h Handler) {
    w.handlers[h.Type()] = h.Execute
}

// In module.go — each module registers its own handlers
func (m *Module) RegisterOutboxHandlers(worker *outbox.Worker) {
    worker.RegisterHandler(m.welcomeEmailHandler)
}
```

Outbox type keys use constants exported from the handler's `application/` package — zero magic strings in registration. See [Outbox Type Keys](#outbox-type-keys).

## Rules

| Rule | Why |
|------|-----|
| `Start(ctx context.Context)` signature | Context is the universal shutdown signal. No custom stop channels. |
| No `Stop()` method | Context cancellation IS stop. Don't duplicate. |
| Worker lives in owning module | If the worker processes orders, it belongs to the order module. |
| Platform workers in `shared/platform/` | Outbox, log ingest — infrastructure that isn't business logic. |
| `StartWorkers(ctx)` on Module struct | Single entry point per module. `cmd/` calls it once. |
| Log prefix `[TypeName.Method]` | Same greppable convention as services and handlers. |
| Recover from panics inside worker loop | A panic in one iteration must not kill the entire worker. |
| Flush on shutdown | When ctx is cancelled, drain buffers before returning. |
| No `init()` for worker registration | Workers are explicitly started in `StartWorkers`. |
| Options pattern for config | `WithPollInterval`, `WithBatchSize` — not 7 positional args. |

---

# Outbox Pattern

## Principle

Reliable side-effect delivery without distributed transactions. When a module's service method must trigger an external side effect (email, push notification, external API call), it does NOT perform the side effect inline. Instead, it writes a message to an outbox table **within the same database transaction** as the business write. A platform-level worker polls the outbox and performs the side effect asynchronously with retries.

```
Service method (inside tx)          Outbox Worker (poll loop)
─────────────────────────          ─────────────────────────
INSERT INTO orders (...)           SELECT * FROM outbox WHERE status=pending
INSERT INTO outbox (...)              ↓
COMMIT                            Execute handler
                                      ↓ (success) → MARK sent
                                      ↓ (failure) → MARK retry or failed
```

The write and the message are atomically committed. If the business write fails, the message is never visible. If the side effect fails, the worker retries.

## Why Not Events

The event bus (`shared/EventBus`) is in-process and fire-and-forget. If the process crashes between `COMMIT` and the event handler, the side effect is lost. Events are for **in-process pub/sub** — not durable delivery guarantees. Use outbox when the side effect must happen.

| Mechanism | Durability | Latency | Use Case |
|-----------|-----------|---------|----------|
| Event Bus | Lost on crash | Immediate | Cross-module notification, cache invalidation |
| Outbox | Survives crash | Poll interval (seconds) | Email, SMS, external API call, push notification |

## Where It Lives

Outbox is **platform infrastructure** — not a business module. Lives in `modules/shared/platform/outbox/`.

```
modules/shared/
  platform/
    outbox/
      entity.go          # MessageEntity, Handler interface, NewMessage()
      repository.go      # Insert, FetchPending, MarkSent, MarkFailed, IncrementRetry, DeleteExpired
      worker.go          # Worker — polls, dispatches to registered handlers, retries, reaps
```

Business modules never create their own outbox tables or poll loops. They use the shared outbox through a single `Repository` interface and register handlers.

## Message Entity

```go
// modules/shared/platform/outbox/entity.go
package outbox

const (
    StatusPending    = 0
    StatusProcessing = 1
    StatusSent       = 2
    StatusFailed     = 3
)

// MessageEntity maps to the shared "outbox" table.
type MessageEntity struct {
    ID          uuid.UUID       `db:"id"`
    Type        string          `db:"type"`
    Payload     json.RawMessage `db:"payload"`
    Status      int             `db:"status"`
    RetryCount  int             `db:"retry_count"`
    MaxRetries  int             `db:"max_retries"`
    CreatedAt   time.Time       `db:"created_at"`
    ProcessedAt *time.Time      `db:"processed_at"`
    LastError   *string         `db:"last_error"`
}

// Handler defines the interface for processing outbox messages.
// Each module implements this for its specific message types.
type Handler interface {
    Type() string                                   // unique type key returned from exported constant
    Execute(ctx context.Context, msg MessageEntity) error
}

// NewMessage creates a pending outbox message with UUIDv7 and current timestamp.
func NewMessage(msgType string, payload any) (MessageEntity, error) {
    data, _ := json.Marshal(payload)
    id, _ := uuid.NewV7()
    return MessageEntity{
        ID:        id,
        Type:      msgType,
        Payload:   data,
        Status:    StatusPending,
        CreatedAt: time.Now(),
    }, nil
}
```

One shared `outbox` table for all modules. The `type` column discriminates which handler processes the message. No per-module outbox tables — that's premature partitioning.

## Outbox Table Schema

```sql
-- MySQL
CREATE TABLE outbox (
    id CHAR(36) NOT NULL PRIMARY KEY,
    type VARCHAR(100) NOT NULL,
    payload JSON NOT NULL,
    status TINYINT NOT NULL DEFAULT 0,
    retry_count INT NOT NULL DEFAULT 0,
    max_retries INT NOT NULL DEFAULT 3,
    last_error TEXT,
    created_at TIMESTAMP NOT NULL,
    processed_at TIMESTAMP NULL,
    INDEX idx_status (status),
    INDEX idx_type (type),
    INDEX idx_created_at (created_at)
);
```

```sql
-- PostgreSQL
CREATE TABLE outbox (
    id UUID NOT NULL PRIMARY KEY,
    type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    status SMALLINT NOT NULL DEFAULT 0,
    retry_count INT NOT NULL DEFAULT 0,
    max_retries INT NOT NULL DEFAULT 3,
    last_error TEXT,
    created_at TIMESTAMPTZ NOT NULL,
    processed_at TIMESTAMPTZ,
    INDEX idx_status (status)
);
CREATE INDEX idx_outbox_type ON outbox (type);
CREATE INDEX idx_outbox_created_at ON outbox (created_at);
```

```sql
-- MSSQL
CREATE TABLE outbox (
    id UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
    type NVARCHAR(100) NOT NULL,
    payload NVARCHAR(MAX) NOT NULL,      -- JSON stored as NVARCHAR(MAX)
    status TINYINT NOT NULL DEFAULT 0,
    retry_count INT NOT NULL DEFAULT 0,
    max_retries INT NOT NULL DEFAULT 3,
    last_error NVARCHAR(MAX),
    created_at DATETIME2 NOT NULL,
    processed_at DATETIME2 NULL
);
CREATE INDEX idx_outbox_status ON outbox (status);
CREATE INDEX idx_outbox_type ON outbox (type);
CREATE INDEX idx_outbox_created_at ON outbox (created_at);
```

| Column | Type | Purpose |
|--------|------|---------|
| `id` | CHAR(36) / UUID / UNIQUEIDENTIFIER | Primary key. UUID V7 — time-sorted for natural poll order. |
| `type` | VARCHAR(100) | Handler discriminator. `"email"`, `"gmtools.chat"`, `"roles.sync_base_table"`. Worker routes to handler by this value. |
| `payload` | JSON / JSONB / NVARCHAR(MAX) | Serialized message body. Handler unmarshals to its expected struct. |
| `status` | TINYINT | `0=pending`, `1=processing`, `2=sent`, `3=failed`. Worker advances this. |
| `retry_count` | INT | How many times the worker has attempted. Incremented on retry. |
| `max_retries` | INT | Default 3. When `retry_count >= max_retries`, marked failed. Can be set per-message for critical side effects. |
| `last_error` | TEXT | Error message from last failed attempt. NULL on first try. |
| `created_at` | TIMESTAMP | When the message was inserted (== business write commit time). |
| `processed_at` | TIMESTAMP | When the message reached terminal state (sent or failed). NULL while pending/processing. |

**Index rationale:**
- `idx_status` — primary query filter. `WHERE status = 0` for `FetchPending`.
- `idx_type` — diagnostic queries: "how many mails failed today?"
- `idx_created_at` — reaper: `DELETE WHERE created_at < NOW() - INTERVAL 7 DAY`.

No index on `retry_count`, `max_retries` — not queried in isolation.

## Reaper Retention

```sql
-- MySQL
DELETE FROM outbox
WHERE status IN (2, 3)           -- sent or failed
  AND created_at < NOW() - INTERVAL 7 DAY;

-- PostgreSQL
DELETE FROM outbox
WHERE status IN (2, 3)
  AND created_at < NOW() - INTERVAL '7 days';

-- MSSQL
DELETE FROM outbox
WHERE status IN (2, 3)
  AND created_at < DATEADD(DAY, -7, SYSUTCDATETIME());
```

Default 7 days. Configurable via `WithRetentionDays(30)`.

## Repository

```go
// modules/shared/platform/outbox/repository.go
package outbox

type Repository interface {
    Insert(ctx context.Context, tx *sqlx.Tx, msg MessageEntity) error
    FetchPending(ctx context.Context, limit int) ([]MessageEntity, error)
    MarkProcessing(ctx context.Context, id uuid.UUID) error
    MarkSent(ctx context.Context, id uuid.UUID) error
    MarkFailed(ctx context.Context, id uuid.UUID, lastError string) error
    IncrementRetry(ctx context.Context, id uuid.UUID, lastError string) error
    DeleteExpired(ctx context.Context, retentionDays int) (int64, error)
}
```

**Key behaviors of a correct implementation:**

- `Insert` with `tx` — message is written in the same transaction as the business write. If invoked with `nil` tx, falls back to direct DB. Defaults `MaxRetries` to 3 if zero.
- `FetchPending` uses `FOR UPDATE SKIP LOCKED` — concurrent workers never grab the same message. Returns empty slice (never null) when idle.
- `IncrementRetry` resets status back to `StatusPending` — the worker re-picks it on the next poll cycle.
- `MarkFailed` is terminal — status `StatusFailed`, retries exhausted. Only reaper cleans these up.
- `DeleteExpired` cleans sent + failed messages older than N days. Called by the reaper on a slow timer.

## Worker

```go
// modules/shared/platform/outbox/worker.go
type Worker struct {
    repo           Repository
    log            logger.Logger
    handlers       map[string]func(context.Context, MessageEntity) error
    pollInterval   time.Duration
    batchSize      int
    maxConcurrency int
    retentionDays  int
}

func NewWorker(repo Repository, log logger.Logger, opts ...WorkerOption) *Worker

// RegisterHandler binds a handler to a message type key.
func (w *Worker) RegisterHandler(h Handler)

// Start begins the poll loop. Blocks until ctx is cancelled.
func (w *Worker) Start(ctx context.Context)
```

Worker is generic. It knows nothing about email, chat, or any business domain. It polls, dispatches, retries, reaps. Handlers are the plugin points.

**Worker lifecycle per tick:**
1. `FetchPending(batchSize)` with `FOR UPDATE SKIP LOCKED`
2. For each message, acquire semaphore (max concurrency gate), launch goroutine
3. Goroutine: `MarkProcessing` → `handler.Execute` → on success `MarkSent`, on failure check `RetryCount < MaxRetries` → `IncrementRetry` (back to pending) else `MarkFailed`
4. On reaper tick: `DeleteExpired(retentionDays)`

## Module Usage — Write Side

Service writes the message inside the UoW transaction:

```go
// modules/order/application/service.go
func (s *serviceImpl) PlaceOrder(ctx context.Context, input PlaceOrderInput) error {
    return s.uow.Execute(ctx, func(tx *sqlx.Tx) error {
        // Business write
        if err := s.repo.InsertOrder(ctx, tx, orderEntity); err != nil {
            return err
        }

        // Outbox message — same tx. Atomically committed.
        msg, err := outbox.NewMessage(application.OutboxTypeOrderPlaced, OrderPlacedPayload{
            OrderID: orderEntity.ID,
            UserID:  input.UserID,
            Total:   orderEntity.Total,
        })
        if err != nil {
            return fmt.Errorf("[serviceImpl.PlaceOrder] create outbox msg: %w", err)
        }
        if err := s.outboxRepo.Insert(ctx, tx, msg); err != nil {
            return fmt.Errorf("[serviceImpl.PlaceOrder] insert outbox msg: %w", err)
        }

        return nil
    })
}
```

The outbox message survives only if the business write succeeds. No dangling side effects.

## Module Usage — Handler Side

Each module creates a handler struct for its message types. Handler lives alongside other secondary adapters in `adapters/outbox/`:

```go
// modules/order/adapters/outbox/order_handler.go
package outbox

type OrderHandler struct {
    notifier notification.Client
}

func NewOrderHandler(notifier notification.Client) outboxAdapter.Handler {
    return &OrderHandler{notifier: notifier}
}

func (h *OrderHandler) Type() string { return application.OutboxTypeOrderPlaced }

func (h *OrderHandler) Execute(ctx context.Context, msg outboxAdapter.MessageEntity) error {
    var payload OrderPlacedPayload
    if err := json.Unmarshal(msg.Payload, &payload); err != nil {
        return fmt.Errorf("[OrderHandler.Execute] unmarshal: %w", err)
    }
    return h.notifier.SendOrderConfirmation(ctx, payload)
}
```

Handler is a small adapter. Unmarshal → call external system → return. No retry logic. No status management. The worker owns that.

## Handler Registration

Module wires its handlers and exposes a `RegisterOutboxHandlers` method:

```go
// modules/order/module.go
type Module struct {
    orderHandler outboxAdapter.Handler
}

func NewModule(..., notifier notification.Client) *Module {
    return &Module{
        orderHandler: outbox2.NewOrderHandler(notifier),
    }
}

func (m *Module) RegisterOutboxHandlers(worker *outboxAdapter.Worker) {
    worker.RegisterHandler(m.orderHandler)
}
```

## Wiring in cmd/

```go
// cmd/server/main.go
func main() {
    db := connectDB()
    cfg := config.Load()
    cfg.Validate()
    c := shared.NewContainer(db, cfg)

    // Infrastructure
    outboxRepo := outbox.NewRepository(db, log)
    outboxWorker := outbox.NewWorker(outboxRepo, log,
        outbox.WithPollInterval(5*time.Second),
        outbox.WithBatchSize(20),
        outbox.WithMaxConcurrency(10),
        outbox.WithRetentionDays(7),
    )

    // Modules — order depends on outboxRepo to write, registers handlers to outboxWorker
    order.Register(c, r, outboxRepo, outboxWorker)
    notification.Register(c, r, outboxWorker)

    go outboxWorker.Start(ctx)
    http.ListenAndServe(":"+cfg.AppPort, r)
}
```

One outbox table. One worker. Many handlers — each owned by its module.

## Outbox Type Keys

Type strings follow `<module>.<action>` or `<domain>.<thing>`:

```
order.placed          # module order, order placed event
notification.mail     # module notification, send email
notification.push     # module notification, push notification
auth.welcome_email    # module auth, welcome email on registration
```

Modules export their type constants:

```go
// modules/order/application/constants.go
const (
    OutboxTypeOrderPlaced = "order.placed"
)
```

No magic strings. The handler's `Type()` returns the constant.

## Rules

| Rule | Why |
|------|-----|
| One shared `outbox` table | Single poll source. No per-module table scatter. `type` column discriminates. |
| `Insert` inside the same tx as business write | Atomicity. Either both committed or neither. |
| `Repository` interface in platform package | One implementation. Modules depend on the interface only. |
| `Handler` interface with `Type()` + `Execute()` | Worker knows nothing about business. Handlers are plugins. |
| Handler is pure adapter — unmarshal → call → return | No retry, no status, no DB. Worker handles all that. |
| `RegisterOutboxHandlers(worker)` on Module | Module owns its handlers. Builder only calls registration. |
| Type constants exported from `application/` package | No magic strings. Compile-time safe. |
| `FOR UPDATE SKIP LOCKED` in FetchPending | Concurrent workers don't fight over same messages. |
| Retry: back to `StatusPending` with incremented counter | Worker re-picks it next cycle. No custom retry scheduling. |
| MaxRetries defaults to 3 if zero | Sensible default. Module can override via `NewMessage` payload. |
| Reaper cleans sent + failed older than N days | Prevents unbounded table growth. Separate slow timer. |
| `Start(ctx)` blocks until ctx cancelled | Graceful shutdown. Worker drains current batch on cancel. |

---

# Package Dependencies

Third-party packages used across the conventions documented above. Each has a single clear purpose.

## Third-Party Packages

| Package | Import Path | Purpose |
|---------|------------|---------|
| **chi** | `github.com/go-chi/chi/v5` | HTTP router — lightweight, stdlib-compatible, middleware chain |
| **chi middleware** | `github.com/go-chi/chi/v5/middleware` | RequestID, Logger, Recoverer, Timeout |
| **httprate** | `github.com/go-chi/httprate` | Rate limiting middleware (token bucket, per-IP) |
| **sqlx** | `github.com/jmoiron/sqlx` | SQL extension — `db:` struct tags, `NamedExec`, `Select`, `Get` |
| **uuid** | `github.com/google/uuid` | UUID V4 / V7 generation |
| **validator** | `github.com/go-playground/validator/v10` | Struct tag validation — `validate:"required,min=1"` |
| **swaggo** | `github.com/swaggo/swag` | OpenAPI/Swagger doc generator — `swag init` from godoc annotations |
| **slog** | `log/slog` (stdlib, Go 1.21+) | Structured logging — leveled, key=value pairs |
| **slogchi** | `github.com/go-chi/httplog/v2` | slog integration with chi request logging |

## Standard Library

| Package | Purpose |
|---------|---------|
| `context` | Request-scoped values, cancellation, deadlines |
| `database/sql` | SQL driver abstraction — `*sql.DB`, `*sql.Tx` |
| `encoding/json` | JSON marshal/unmarshal |
| `net/http` | HTTP server, handler interface, response writer |
| `time` | Timestamps, durations, tickers |
| `fmt` | Error wrapping (`fmt.Errorf("%w: ...", err)`) |
| `strings` | String builders, trimming, splitting |
| `sync` | `sync.Mutex`, `sync.Once`, `sync.WaitGroup` |
| `os` | Environment variables, file I/O, process signals |
| `os/signal` | Graceful shutdown — `signal.NotifyContext` |
| `errors` | `errors.Is`, `errors.As`, `errors.Join` |
| `runtime/debug` | Stack traces on panic recovery |

---

# What to NEVER Do

| Violation | Why It's Broken | Fix |
|-----------|----------------|-----|
| Import `modules/order/service.go` from `modules/user` | Bypasses public API | Use `order.Service` via `ports/` |
| Query `orders` table from `user` module | Shared data ownership | Go through order module's API |
| Put `User` model in `shared/` | Bleeds business domain into kernel | Keep in `user` module, export what others need |
| Circular import between modules | Compiler error + broken boundaries | Extract shared concern or merge modules |
| Skip `ports/` — export from random files | No contract, boundaries rot | One public file per module |
| Skip `module.go` — wire from `cmd/` | Scatters DI knowledge across codebase | Each module has `module.go` with `Register(c)` |
| `modules/handlers/`, `modules/models/` | Technical layering, not business capability | Reorganize by business domain |
| Global `var` for module state | Hidden coupling | Constructor + dependency injection |
| Pass `*sql.DB` between modules | Leaks infrastructure concern | Repository pattern inside each module |
| `c.Get` + type assertion in `cmd/` | Composition root knows module internals | `MustGet` calls live inside `module.go` |
| Module A's `module.go` imports Module B's `service.go` | Bypasses DI contract | Only use `ports/` types, resolve via container |

---

# Adding a New Module Checklist

1. [ ] Create `modules/<name>/` directory (singular, lowercase, business capability).
2. [ ] Define `ports/` — public types, dependency interfaces, public service struct + methods.
3. [ ] Define `module.go` — `Register(c *shared.Container)` that resolves deps, constructs internals, calls `c.Set`, subscribes to events.
4. [ ] Add dependency declaration comment at top of `module.go` (which modules it needs / provides key).
5. [ ] Implement business logic in private files (`service.go`, `repository.go`, `models.go`).
6. [ ] Register in `cmd/server/main.go` — one line: `<module>.Register(c)` — in correct dependency order.
7. [ ] Verify: `module.go` is the only file importing from other modules' `ports/`. Internal files only import `shared` + stdlib.
8. [ ] Verify: `MustGet` keys match `c.Set` keys from dependency modules.
9. [ ] Verify: grep for `c.Set(` across the project — each key must be unique and documented.

---

# Extracting a Module to a Service

The test of good boundaries: can you extract this module into its own service with minimal changes?

1. Copy the module directory to a new repo.
2. Replace the `ports/` interface definitions with real HTTP/gRPC clients.
3. In the monolith, replace the direct call with the HTTP/gRPC client implementing the same interface.
4. **Nothing else should change.** If other files need modification, the boundary was leaking.

---

# Enforcement

When writing or reviewing Go code in this project:

1. **Every Go file belongs to a module** — no loose `.go` files at project root (except `main.go` in `cmd/`).
2. **Every module has `ports/`** — the single public contract.
3. **Every module has `module.go`** — self-contained DI registration (`Register(c *shared.Container)`).
4. **No cross-module internal imports** — grep for `modules/X` imports from `modules/Y`. All must resolve to `ports/` exports.
5. **No wiring in `cmd/`** — `cmd/server/main.go` is a flat list of `module.Register(c)` calls. Zero knowledge of how any module is built.
6. **DI keys follow convention** — `<module>.ports` for services: `"user.ports"`, `"order.ports"`. Keys documented as constants in `ports/`.
7. **`MustGet` only in `module.go`** — never in `cmd/`, never in `ports/`, never in `service.go`.
8. **No shared database tables** — each module copies data it needs (eventual consistency) or calls the owning module's API.
9. **Dependency direction flows inward** — `cmd` → modules → `shared`. Never `shared` → module. Never circular between modules.
10. When in doubt, ask: "If I extracted this module into its own service tomorrow, what would break?"
