---
name: nextjs-modular-architecture
description: Enforce modular architecture in Next.js 16 — client-first, feature modules, per-module tsyringe DI, thin app/ routing shell, Zod validation schemas, API clients as infrastructure.
version: 2.2.0
author: User
---

# Next.js 16 Modular Architecture (Client-First)

## Purpose

Enforce modular architecture in Next.js 16 for client-first applications. No database. No Server Components data fetching. Every piece of business code lives inside a feature module. Modules own their schemas, types, UI, API clients, and DI wiring. The backend is an external API — Next.js is the frontend shell. Goal: replaceable modules, clear boundaries, no cross-module spaghetti, no fat pages.

---

# Core Principle

> A module is a self-contained business capability. It owns its Zod validation schemas, its API response types, its UI components, its API client, and its DI container. No other module may reach into its internals.

Every module must be extractable into its own package **without rewriting business logic**. If you can't extract it cleanly, the boundary is broken.

---

# Architecture Assumption

**Backend is external.** Next.js never talks to a database. All data comes from an external API (REST, GraphQL, gRPC — doesn't matter). The infrastructure layer is purely API clients.

```
Browser
  └── Next.js (this app — client-first)
        └── API Client → External Backend (any stack)
```

This means:
- No Server Components for data fetching. Pages are client-rendered shells.
- No `"use cache"`, no `cacheTag`, no `revalidateTag` — caching is the backend's job.
- No `cookies()`, `headers()`, `proxy.ts` rewrites to hide backend URLs — the API base URL is config.
- tsyringe DI is the universal pattern — every component resolves its dependencies.
- `infrastructure/` = API clients. No ORM, no DB driver, no `revalidateTag`.
- No "domain entities" or "value objects" — frontend only needs Zod request schemas and API response types.
- Validation happens at the form boundary with Zod. The backend owns the domain model.

---

# Project Structure

```
project-root/
├── proxy.ts                         # next-intl locale routing (Next.js 16)
├── next.config.ts                   # Security headers, reactCompiler, next-intl plugin
├── postcss.config.mjs               # Tailwind v4 PostCSS plugin
├── tsconfig.json                    # experimentalDecorators, emitDecoratorMetadata
├── .env.example                     # Template — commit this. Copy to .env.local
├── messages/                        # i18n translation JSON files
│   ├── en/                          # English — source of truth
│   │   ├── common.json              # Buttons, errors, nav, dates
│   │   ├── auth.json                # Auth module strings
│   │   ├── dashboard.json
│   │   └── orders.json
│   └── fr/                          # French — translators fill
│       ├── common.json
│       ├── auth.json
│       ├── dashboard.json
│       └── orders.json
│
└── src/
    ├── env.ts                       # T3 Env — createEnv({ server, client, runtimeEnv })
    │
    ├── i18n/
    │   ├── routing.ts               # defineRouting({ locales, defaultLocale, localePrefix })
    │   └── request.ts               # getRequestConfig — loads messages per locale
    │
    ├── app/                         # Thin routing shell — no business logic
    │   ├── robots.ts                # Crawl rules + sitemap reference
    │   ├── sitemap.ts               # All public URLs for search engines
    │   ├── manifest.json            # PWA manifest
    │   ├── layout.tsx               # Root: <html lang>, metadata, metadataBase, viewport
    │   │
    │   ├── _lib/                    # Shared app infrastructure (private folder)
    │   │   ├── di/
    │   │   │   ├── container.ts     # Root tsyringe container — import 'reflect-metadata'
    │   │   │   ├── use-resolve.ts   # useResolve<T>(token) hook — useMemo wrapper
    │   │   │   └── provider.tsx     # DIProvider — calls initializeAllModules() on mount
    │   │   └── query/
    │   │       └── provider.tsx     # QueryClientProvider — TanStack Query defaults
    │   │
    │   └── [locale]/                # Locale route group (from next-intl routing)
    │       ├── (auth)/
    │       │   ├── login/page.tsx   # export metadata + export default LoginPage
    │       │   ├── register/page.tsx
    │       │   └── layout.tsx       # Auth layout — centered card, no nav
    │       ├── (dashboard)/
    │       │   ├── page.tsx         # export default DashboardPage
    │       │   ├── projects/[id]/page.tsx  # export generateMetadata + page
    │       │   ├── projects/[id]/json-ld.tsx
    │       │   ├── settings/page.tsx
    │       │   └── layout.tsx       # Dashboard layout — sidebar + header, AuthGuard
    │       ├── loading.tsx
    │       ├── error.tsx
    │       └── not-found.tsx
    │
    └── modules/                     # Feature modules — extractable business capabilities
        ├── registry.ts              # initializeAllModules() — dependency-ordered init calls
        │
        ├── shared/                  # Cross-cutting kernel
        │   ├── di/
        │   │   ├── tokens.ts        # SHARED_TOKENS.ApiClient, Logger
        │   │   └── container.ts     # Logger reg, Axios + interceptors, initializeSharedModule()
        │   ├── types/
        │   │   ├── logger.ts        # ILogger interface
        │   │   ├── api-error.ts     # ApiError { code, message, field? }
        │   │   └── pagination.ts    # PaginationParams, PaginationMetadata, PaginatedResponse<T>
        │   ├── infrastructure/
        │   │   └── pino-logger.ts   # PinoLogger implements ILogger
        │   ├── stores/              # Shared Zustand stores
        │   │   ├── notification.store.ts
        │   │   └── theme.store.ts
        │   ├── ui/
        │   │   ├── form-field.tsx   # FormField — TanStack Form + shadcn styled input
        │   │   └── field-error.tsx  # FieldError — renders ALL errors
        │   ├── utils/
        │   │   ├── cn.ts           # clsx wrapper
        │   │   └── format-date.ts
        │   └── index.ts            # Barrel
        │
        ├── auth/                    # Auth module
        │   ├── schemas/
        │   │   ├── login.schema.ts
        │   │   ├── register.schema.ts
        │   │   └── change-password.schema.ts
        │   ├── types/
        │   │   ├── responses.ts     # LoginResponse, RegisterResponse, UserDto, TokenPair
        │   │   └── errors.ts
        │   ├── stores/
        │   │   └── auth.store.ts    # createAuthStore() — factory receives AuthService
        │   ├── application/
        │   │   └── auth.service.ts  # AuthService interface + AuthServiceImpl
        │   ├── repositories/
        │   │   └── auth.repository.ts
        │   ├── ui/
        │   │   ├── login-page.tsx
        │   │   ├── login-form.tsx   # TanStack Form + FormField + useTranslations
        │   │   ├── register-form.tsx
        │   │   ├── user-avatar.tsx
        │   │   ├── auth-guard.tsx
        │   │   └── logout-button.tsx
        │   ├── di/
        │   │   ├── tokens.ts        # AUTH_TOKENS: AuthService, AuthRepository, AuthStore
        │   │   └── container.ts     # createChildContainer + register
        │   └── index.ts             # Public barrel
        │
        ├── dashboard/               # Dashboard module
        │   ├── types/
        │   │   └── responses.ts
        │   ├── stores/
        │   │   └── dashboard.store.ts
        │   ├── application/
        │   │   └── dashboard.service.ts
        │   ├── repositories/
        │   │   └── dashboard.repository.ts
        │   ├── ui/
        │   │   └── dashboard-page.tsx
        │   ├── di/
        │   │   ├── tokens.ts
        │   │   └── container.ts
        │   └── index.ts
        │
        ├── orders/                  # Orders module — TanStack Query example
        │   ├── schemas/
        │   │   └── create-order.schema.ts
        │   ├── types/
        │   │   ├── responses.ts     # OrderDto
        │   │   ├── filters.ts       # OrderFilters
        │   │   └── query-keys.ts    # orderKeys — const key factory
        │   ├── stores/
        │   │   └── filter.store.ts  # Zustand — active filters (client state)
        │   ├── hooks/
        │   │   ├── use-order-list.ts    # useInfiniteQuery — paginated
        │   │   ├── use-order-detail.ts  # useQuery — single order
        │   │   └── use-create-order.ts  # useMutation — invalidates list
        │   ├── application/
        │   │   └── order.service.ts
        │   ├── repositories/
        │   │   └── order.repository.ts
        │   ├── ui/
        │   │   ├── order-list-page.tsx
        │   │   └── order-card.tsx
        │   ├── di/
        │   │   ├── tokens.ts
        │   │   └── container.ts
        │   └── index.ts
        │
        └── billing/                 # Billing module (example)
            ├── schemas/
            ├── types/
            ├── application/
            ├── repositories/
            ├── ui/
            ├── di/
            └── index.ts
```

---

# Next.js 16 — Relevant Bits Only

## React Compiler (Stable, Opt-In)

```ts
// next.config.ts
const nextConfig = {
  reactCompiler: true,
};
```

Auto-memoizes components. DI hooks with `useMemo` are still correct and self-documenting — the compiler cannot know `container.resolve` is side-effect-free.

## Turbopack Default

No config needed. Webpack via `--webpack` opt-out.

## Async Request APIs

`params`, `searchParams` are async in page/layout. Since pages are thin shells that just instantiate module components, this only affects the route layer.

## Environment Variables — `@t3-oss/env-nextjs`

### Why T3 Env

`process.env` has no type safety. `NEXT_PUBLIC_*` prefix is a convention, not enforced. T3 Env validates **at build time and runtime** — missing `API_BASE_URL` crashes the build, not the user. Server-only vars never leak to the client. Client vars get `NEXT_PUBLIC_*` prefix enforced.

### Setup

```bash
bun add @t3-oss/env-nextjs zod
```

Zod is already a project dependency (form schemas) — no extra cost.

### `src/env.ts` — Single Env Source

```ts
// src/env.ts
import { createEnv } from '@t3-oss/env-nextjs';
import { z } from 'zod';

export const env = createEnv({
  /**
   * Server-only vars. Never sent to the browser.
   * Accessible in: Server Components, Route Handlers, next.config.ts, DI container init.
   */
  server: {
    NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
    API_BASE_URL: z.string().url(),
    API_KEY: z.string().min(1),
  },

  /**
   * Client vars. Must be prefixed with NEXT_PUBLIC_.
   * Bundled into the client JS — no secrets here.
   */
  client: {
    NEXT_PUBLIC_API_BASE_URL: z.string().url(),
    NEXT_PUBLIC_APP_NAME: z.string().min(1),
  },

  runtimeEnv: {
    NODE_ENV: process.env.NODE_ENV,
    API_BASE_URL: process.env.API_BASE_URL,
    API_KEY: process.env.API_KEY,
    NEXT_PUBLIC_API_BASE_URL: process.env.NEXT_PUBLIC_API_BASE_URL,
    NEXT_PUBLIC_APP_NAME: process.env.NEXT_PUBLIC_APP_NAME,
  },

  skipValidation: process.env.SKIP_ENV_VALIDATION === 'true',
  emptyStringAsUndefined: true,
});
```

### Usage — Replace All `process.env`

```ts
// Before
baseURL: process.env.NEXT_PUBLIC_API_BASE_URL ?? '/api',

// After
import { env } from '@/env';
baseURL: env.NEXT_PUBLIC_API_BASE_URL,
```

**Rules**:
- Import `env` from `@/env` — never read `process.env` directly.
- Server vars in server code only. T3 Env throws if client code imports a server var.
- Client vars always `NEXT_PUBLIC_*` — T3 Env enforces the prefix.
- `skipValidation` + `SKIP_ENV_VALIDATION=true` for Docker/CI builds.
- Standalone output: add `transpilePackages: ["@t3-oss/env-nextjs", "@t3-oss/env-core"]` in next.config.ts.

### `.env.example` — Template

```bash
# .env.example — commit this. Copy to .env.local for development.

# Server
API_BASE_URL=http://localhost:8080/api/v1
API_KEY=dev-api-key

# Client (NEXT_PUBLIC_ prefix required)
NEXT_PUBLIC_API_BASE_URL=http://localhost:8080/api/v1
NEXT_PUBLIC_APP_NAME=MyApp
```

## next.config.ts — Security Headers

### Principle

Security headers are set once in `next.config.ts`, not in middleware or per-route. The `headers()` async function applies headers by path pattern. All security headers go on `/:path*` (global). CORS goes on `/api/:path*`.

### TypeScript Config

```ts
// next.config.ts
import type { NextConfig } from 'next';
import { env } from '@/env';
import createNextIntlPlugin from 'next-intl/plugin';

const nextConfig: NextConfig = {
  reactCompiler: true,
  transpilePackages: ['@t3-oss/env-nextjs', '@t3-oss/env-core'],

  async headers() {
    const isProd = env.NODE_ENV === 'production';

    return [
      // Global security headers — all routes
      {
        source: '/:path*',
        headers: [
          { key: 'X-DNS-Prefetch-Control', value: 'on' },
          ...(isProd
            ? [{ key: 'Strict-Transport-Security' as const, value: 'max-age=63072000; includeSubDomains; preload' }]
            : []),
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
        ],
      },
      // CORS — API routes only
      {
        source: '/api/:path*',
        headers: [
          { key: 'Access-Control-Allow-Origin', value: '*' },
          { key: 'Access-Control-Allow-Methods', value: 'GET, POST, PUT, DELETE, OPTIONS' },
          { key: 'Access-Control-Allow-Headers', value: 'Content-Type, Authorization' },
        ],
      },
    ];
  },
};

export default createNextIntlPlugin()(nextConfig);
```

### Header Summary

| Header | Value | When | Why |
|--------|-------|------|-----|
| `X-DNS-Prefetch-Control` | `on` | always | DNS prefetch for external links |
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains; preload` | production only | HTTPS-only — breaks localhost/dev |
| `X-Content-Type-Options` | `nosniff` | always | Block MIME-type sniffing |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | always | Strip path on cross-origin |
| `X-Frame-Options` | `DENY` | always | Block clickjacking |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | always | Disable browser API access |

### CSP (Content-Security-Policy)

CSP is **not** set globally via `headers()` for a client-first app. Why:
- Inline scripts/styles from Next.js's dev bundler conflict with strict `script-src` / `style-src`.
- CSP is easier to iterate on via a `Content-Security-Policy-Report-Only` header in development.
- Use `proxy.ts` for CSP so you can compute nonces dynamically if needed.

If the project needs CSP: add `proxy.ts` with `nonce` generation, set `Content-Security-Policy` per-request. Not covered here — start with the six headers above.

### Standalone Output (Docker)

```ts
// next.config.ts
const nextConfig: NextConfig = {
  output: 'standalone',
  transpilePackages: ['@t3-oss/env-nextjs', '@t3-oss/env-core'],
};
```

---

# Module Definition

## What Makes a Module

A module is a directory under `src/modules/` that represents a **business capability**, not a technical layer.

```
src/
  app/                          # Thin routing shell — pages compose from modules
    (auth)/login/page.tsx       # → modules/auth ui
    (dashboard)/page.tsx        # → modules/dashboard ui
    layout.tsx
    _lib/                       # Shared DI infra (useResolve, container)
  modules/
    auth/                       # Business capability
    dashboard/                  # Business capability
    settings/                   # Business capability
    shared/                     # Cross-cutting — axios, logger, types, utils
```

**Module = business capability.** Not a layer. Not a utility drawer.

```
src/
  components/                   # ❌ technical dump, not a module
  hooks/                        # ❌ technical dump, not a module
  services/                     # ❌ technical dump, not a module
  utils/                        # ❌ technical dump, not a module
```

## Module Name Rules

- **Singular** noun describing the business capability: `auth`, `dashboard`, `settings`, `billing`.
- **camelCase** directory name: `orderManagement`, not `OrderManagement` or `order_management`.
- Module directory name === module identity. Used in DI tokens, barrel exports, route groups.

---

# Module Internal Structure

## Ports & Adapters (Hexagonal Architecture)

Every module follows ports & adapters. Strict dependency direction.

```
src/modules/auth/
  schemas/                      # Zod validation schemas — request shapes
  types/                        # API response types, error types (plain interfaces)
  stores/                       # Zustand stores — client state, factory receives services
  hooks/                        # TanStack Query hooks — useQuery/useMutation wrappers
  application/                  # Services — thin orchestration, calls repositories
  repositories/                 # Data fetching — axios calls, returns response types
  ui/                           # React components — pages, forms, widgets
  di/                           # tsyringe container config
    container.ts                # Child container + registration
    tokens.ts                   # Symbol tokens
  index.ts                      # Public API barrel
```

**Why no `domain/`**: In a client-frontend that only calls an external API, there are no "entities" or "value objects" to model. The data shapes are:
- **Request shapes** — what you send to the API → Zod schemas with inferred types
- **Response shapes** — what the API sends back → plain TypeScript interfaces
- **Error shapes** — what the API returns on failure → plain TypeScript interfaces

The backend owns the domain model. The frontend only needs to validate inputs and type-check responses.

## Dependency Direction

```
ui → stores/hooks → application → repositories

- `schemas/` depends on **nothing** — pure Zod, no React, no Next.js, no HTTP.
- `types/` depends on **nothing** — pure TypeScript interfaces, no React, no Next.js, no HTTP.
- `stores/` depends on `application/`, `types/` — receives services via factory, holds client state.
- `hooks/` depends on `application/`, `types/` — wraps TanStack Query for components.
- `application/` depends on `types/`, `schemas/`, `repositories/` — uses response types, inferred request types, calls repos.
- `repositories/` depends on `types/`, `schemas/` — data fetching, axios calls, returns response types.
- `ui/` depends on `stores/`, `hooks/`, `schemas/` — uses store actions/selectors, query hooks, Zod schemas. Never calls services directly.
- `di/` depends on everything — wires the module.
- `index.ts` re-exports only what other modules may use.
  ```

## Package-by-Package

### `schemas/` — Zod Validation Schemas. Zero Dependencies.

Request validation schemas using Zod. Each schema file exports the schema + its inferred type. No React imports. No Next.js imports. No HTTP imports.

```
src/modules/auth/schemas/
  login.schema.ts                     # LoginRequest schema (form == request)
  register.schema.ts                  # RegisterRequest + registerFormSchema (form != request)
  change-password.schema.ts           # ChangePasswordRequest schema
  ```

```ts
// src/modules/auth/schemas/login.schema.ts
import { z } from 'zod';

export const loginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

export type LoginRequest = z.infer<typeof loginSchema>;
```

### Form Schema vs Request Type

Form fields don't always match the API request shape. Common case: `confirmPassword` exists in the form for UX but is never sent to the backend. The **form schema** validates what the user sees; the **request type** is what the API receives. Use `.transform()` to strip extra fields at the boundary.

```ts
// src/modules/auth/schemas/register.schema.ts
import { z } from 'zod';

// Form schema — validates both password fields match
export const registerFormSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(128),
  confirmPassword: z.string(),
  name: z.string().min(1).max(100),
}).refine(
  (data) => data.password === data.confirmPassword,
  { message: 'Passwords do not match', path: ['confirmPassword'] },
);

// Form type — what TanStack Form manages
export type RegisterFormValues = z.infer<typeof registerFormSchema>;

// Request type — what gets sent to the API (no confirmPassword)
export const registerSchema = registerFormSchema.transform(({ confirmPassword, ...rest }) => rest);
export type RegisterRequest = z.infer<typeof registerSchema>;
```

TanStack Form uses `registerFormSchema` for `validators.onSubmit`. The `onSubmit` callback receives the full form values, then calls the service with `RegisterRequest`:

```tsx
const form = useForm({
  defaultValues: { email: '', password: '', confirmPassword: '', name: '' },
  validatorAdapter: zodValidator(),
  validators: {
    onSubmit: registerFormSchema,
  },
  onSubmit: async ({ value }) => {
    // value is RegisterFormValues — includes confirmPassword
    const request = registerFormSchema.parse(value); // transform strips confirmPassword
    await authService.register(request); // request is RegisterRequest
  },
});
```

**Rules**:
- One schema file per operation — `login.schema.ts`, `register.schema.ts`.
- Suffix: `FormSchema` for the form validation schema, `Schema` for the API request schema (post-transform).
- Suffix: `FormValues` for form type, `Request` for request type.
- `.transform()` strips form-only fields (`confirmPassword`, terms checkboxes, etc.) — the API never sees them.
- `.refine()` for cross-field validations (password match) — complex rules sit on the form schema, not the request.
- Always export both the form schema and the request type. Form component imports `FormSchema`, service imports `Request`.
- Schemas are passed to TanStack Form's `validators.onSubmit` — form calls `schema.safeParse` internally via `zodValidator()`.
- Services receive already-validated data from the form. No manual `safeParse` in components.
- **Zod validates ALL rules at once** — every constraint on a field runs, all errors collected. `.email()` + `.min(1)` + `.max(255)` all fire simultaneously. Zod's default is eager collection (no `.abortEarly`). User sees every validation failure on first submit, not one-at-a-time whack-a-mole.
- **Generic error messages, not rule-specific** — prefer `'Invalid email address'` not `'Email must contain @'`. Users understand the field label + "invalid" better than raw constraint descriptions.

### `types/` — API Response Types. Plain Interfaces. Zero Dependencies.

What the backend returns. Plain TypeScript — no Zod, no React, no HTTP.

```
src/modules/auth/types/
  responses.ts             # LoginResponse, RegisterResponse, RefreshResponse
  errors.ts                # API error shapes
  ```

```ts
// src/modules/auth/types/responses.ts
export interface LoginResponse {
  user: UserDto;
  tokenPair: TokenPair;
}

export interface RegisterResponse {
  user: UserDto;
}

export interface UserDto {
  id: string;
  email: string;
  name: string;
  role: UserRole;
}

export type UserRole = 'admin' | 'member' | 'viewer';

export interface TokenPair {
  accessToken: string;
  refreshToken: string;
}
```

```ts
// src/modules/auth/types/errors.ts
export interface ApiError {
  code: string;
  message: string;
  field?: string;
}
```

**Rules**:
- Suffix: `Dto` for data-transfer objects from the API (`UserDto`, not `User`).
- Suffix: `Response` for endpoint response wrappers (`LoginResponse`).
- Plain interfaces — no classes, no Zod, no methods.
- Each module defines its own response types. Duplication across modules is fine — each module only needs the fields it consumes.
- `ApiError` shape is the standard error envelope from the backend. Shared across modules via `shared/types/`.

### `application/` — Services

Thin orchestration. Services receive repositories via constructor injection. Services call repository methods, unpack response DTOs, return what the UI needs.

```
src/modules/auth/application/
  auth.service.ts          # AuthService — login, register, logout, refresh
  ```

```ts
// src/modules/auth/application/auth.service.ts
import { inject, injectable } from 'tsyringe';
import { SHARED_TOKENS } from '@/modules/shared';
import type { ILogger } from '@/modules/shared/types/logger';
import type { UserDto } from '../types/responses';
import type { AuthRepository } from '../repositories/auth.repository';
import type { LoginRequest } from '../schemas/login.schema';
import type { RegisterRequest } from '../schemas/register.schema';

export interface AuthService {
  login(input: LoginRequest): Promise<UserDto>;
  register(input: RegisterRequest): Promise<UserDto>;
  logout(): Promise<void>;
  refreshToken(): Promise<TokenPair>;
}

@injectable()
export class AuthServiceImpl implements AuthService {
  private readonly log: ILogger;

  constructor(
    @inject(SHARED_TOKENS.Logger) logger: ILogger,
    private readonly authRepo: AuthRepository,
  ) {
    this.log = logger.child({ module: 'auth', layer: 'application' });
  }

  async login(input: LoginRequest): Promise<UserDto> {
    this.log.info({ email: input.email }, 'login attempt');
    const { user } = await this.authRepo.login(input);
    this.log.info({ userId: user.id }, 'login success');
    return user;
  }

  async register(input: RegisterRequest): Promise<UserDto> {
    this.log.info({ email: input.email }, 'register attempt');
    const { user } = await this.authRepo.register(input);
    this.log.info({ userId: user.id }, 'register success');
    return user;
  }

  async logout(): Promise<void> {
    await this.authRepo.logout();
  }

  async refreshToken(): Promise<TokenPair> {
    return this.authRepo.refreshToken();
  }
}
```

**Service rules**:
- Services depend on `repositories/` types directly — the repository IS the data-fetching contract.
- Thin orchestration — unpack response DTO, return what the UI needs.
- No HTTP concerns. No `axios`, no `Result<T>`, no error normalization. Repositories handle that.
- Inject `ILogger` via `SHARED_TOKENS.Logger`, create child with `{ module, layer }`.

### `repositories/` — Data Fetching

Repositories handle all data fetching. One repository per backend resource. Injects the shared `AxiosInstance` from DI. Service layer never touches axios.

```
src/modules/auth/repositories/
  auth.repository.ts       # AuthRepository — login, register, logout, refresh
  ```

```ts
// src/modules/auth/repositories/auth.repository.ts
import { inject, injectable } from 'tsyringe';
import type { AxiosInstance } from 'axios';
import { SHARED_TOKENS } from '@/modules/shared';
import type { ILogger } from '@/modules/shared/types/logger';
import type { LoginResponse, RegisterResponse, TokenPair } from '../types/responses';
import type { LoginRequest } from '../schemas/login.schema';
import type { RegisterRequest } from '../schemas/register.schema';

@injectable()
export class AuthRepository {
  private readonly log: ILogger;

  constructor(
    @inject(SHARED_TOKENS.Logger) logger: ILogger,
    @inject(SHARED_TOKENS.ApiClient) private readonly apiClient: AxiosInstance,
  ) {
    this.log = logger.child({ module: 'auth', layer: 'repository' });
  }

  async login(req: LoginRequest): Promise<LoginResponse> {
    this.log.debug({ method: 'POST', path: '/auth/login' }, 'api request');
    try {
      const { data } = await this.apiClient.post<LoginResponse>('/auth/login', req);
      return data;
    } catch (err) {
      this.log.error({ method: 'POST', path: '/auth/login', error: err }, 'api error');
      throw err;
    }
  }

  async register(input: RegisterRequest): Promise<RegisterResponse> {
    const { data } = await this.apiClient.post<RegisterResponse>('/auth/register', input);
    return data;
  }

  async logout(): Promise<void> {
    await this.apiClient.post('/auth/logout');
  }

  async refreshToken(): Promise<TokenPair> {
    const { data } = await this.apiClient.post<TokenPair>('/auth/refresh');
    return data;
  }
}
```

**Repository rules**:
- One repository per resource. No interface abstraction — the class IS the contract.
- Inject `SHARED_TOKENS.ApiClient` (`AxiosInstance`) — the singleton axios instance.
- Inject `ILogger` via `SHARED_TOKENS.Logger`, create child with `{ module, layer }`.
- Return `response.data` — strip axios envelope at this layer.
- Throw on non-2xx — axios does this by default. `AxiosError` carries the backend error body.
- No `Result<T>`, no `HttpClient`. Direct. Simple.

### `ui/` — React Components

All `'use client'`. Pages, forms, widgets. Depends on `stores/` and `hooks/` — never calls services directly. Forms use TanStack Form with Zod adapter + shared `FormField` component. Never calls `axios`/`fetch` directly. Never imports from `repositories/`.

```
src/modules/auth/ui/
  login-page.tsx           # LoginPage — full page component
  login-form.tsx           # LoginForm — form with Zod validation
  register-form.tsx        # RegisterForm
  user-avatar.tsx          # UserAvatar — shared widget
  auth-guard.tsx           # AuthGuard — route protection wrapper
  logout-button.tsx        # LogoutButton
```

```tsx
// src/modules/auth/ui/login-form.tsx
'use client';

import { useForm } from '@tanstack/react-form';
import { zodValidator } from '@tanstack/zod-form-adapter';
import { useResolve } from '@/app/_lib/di/use-resolve';
import { AUTH_TOKENS } from '../di/tokens';
import type { AuthService } from '../application/auth.service';
import { loginSchema } from '../schemas/login.schema';
import { FormField } from '@/modules/shared/ui/form-field';
import { useTranslations } from 'next-intl';
import { useState } from 'react';

export function LoginForm() {
  const authService = useResolve<AuthService>(AUTH_TOKENS.AuthService);
  const te = useTranslations('common.errors');
  const t = useTranslations('auth.LoginPage.LoginForm');
  const [error, setError] = useState<string | null>(null);

  const form = useForm({
    defaultValues: { email: '', password: '' },
    validatorAdapter: zodValidator(),
    validators: { onSubmit: loginSchema },
    onSubmit: async ({ value }) => {
      setError(null);
      try {
        await authService.login(value);
      } catch (err) {
        setError(err instanceof Error ? err.message : te('generic'));
      }
    },
  });

  return (
    <form
      onSubmit={(e) => { e.preventDefault(); e.stopPropagation(); form.handleSubmit(); }}
      className="flex flex-col gap-4 w-full max-w-md mx-auto"
      noValidate
    >
      {error && (
        <p role="alert" className="bg-destructive/10 text-destructive text-sm rounded-md px-4 py-2">
          {error}
        </p>
      )}

      <form.Field name="email">
        {(field) => (
          <FormField
            field={field}
            label={t('emailLabel')}
            type="email"
            autoComplete="email"
            placeholder={t('emailPlaceholder')}
          />
        )}
      </form.Field>

      <form.Field name="password">
        {(field) => (
          <FormField
            field={field}
            label={t('passwordLabel')}
            type="password"
            autoComplete="current-password"
            placeholder={t('passwordPlaceholder')}
            rightLabel={
              <a href="/forgot-password" className="text-xs text-primary hover:underline">
                {t('forgotPassword')}
              </a>
            }
          />
        )}
      </form.Field>

      <form.Subscribe selector={(state) => [state.canSubmit, state.isSubmitting]}>
        {([canSubmit, isSubmitting]) => (
          <button
            type="submit"
            disabled={!canSubmit}
            className="w-full h-10 bg-primary text-primary-foreground rounded-md font-medium text-sm hover:bg-primary/90 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
          >
            {isSubmitting ? t('loadingButton') : t('submitButton')}
          </button>
        )}
      </form.Subscribe>
    </form>
  );
}
```

```tsx
// src/modules/auth/ui/register-form.tsx
'use client';

import { useForm } from '@tanstack/react-form';
import { zodValidator } from '@tanstack/zod-form-adapter';
import { useResolve } from '@/app/_lib/di/use-resolve';
import { AUTH_TOKENS } from '../di/tokens';
import type { AuthService } from '../application/auth.service';
import { registerSchema } from '../schemas/register.schema';
import { FormField } from '@/modules/shared/ui/form-field';
import { useTranslations } from 'next-intl';
import { useState } from 'react';

export function RegisterForm() {
  const authService = useResolve<AuthService>(AUTH_TOKENS.AuthService);
  const te = useTranslations('common.errors');
  const t = useTranslations('auth.RegisterPage.RegisterForm');
  const [error, setError] = useState<string | null>(null);

  const form = useForm({
    defaultValues: { email: '', password: '', name: '' },
    validatorAdapter: zodValidator(),
    validators: { onSubmit: registerSchema },
    onSubmit: async ({ value }) => {
      setError(null);
      try {
        await authService.register(value);
      } catch (err) {
        setError(err instanceof Error ? err.message : te('generic'));
      }
    },
  });

  return (
    <form
      onSubmit={(e) => { e.preventDefault(); e.stopPropagation(); form.handleSubmit(); }}
      className="flex flex-col gap-4 w-full max-w-md mx-auto"
      noValidate
    >
      {error && (
        <p role="alert" className="bg-destructive/10 text-destructive text-sm rounded-md px-4 py-2">
          {error}
        </p>
      )}

      <form.Field name="name">
        {(field) => <FormField field={field} label={t('nameLabel')} type="text" autoComplete="name" placeholder={t('namePlaceholder')} />}
      </form.Field>

      <form.Field name="email">
        {(field) => <FormField field={field} label={t('emailLabel')} type="email" autoComplete="email" placeholder={t('emailPlaceholder')} />}
      </form.Field>

      <form.Field name="password">
        {(field) => <FormField field={field} label={t('passwordLabel')} type="password" autoComplete="new-password" placeholder={t('passwordPlaceholder')} />}
      </form.Field>

      <form.Subscribe selector={(state) => [state.canSubmit, state.isSubmitting]}>
        {([canSubmit, isSubmitting]) => (
          <button type="submit" disabled={!canSubmit} className="w-full h-10 bg-primary text-primary-foreground rounded-md font-medium text-sm hover:bg-primary/90 disabled:opacity-50 disabled:cursor-not-allowed transition-colors">
            {isSubmitting ? t('loadingButton') : t('submitButton')}
          </button>
        )}
      </form.Subscribe>
    </form>
  );
}
```

```tsx
// src/modules/auth/ui/login-page.tsx
'use client';

import { LoginForm } from './login-form';

export function LoginPage() {
  return (
    <main>
      <h1>Sign In</h1>
      <LoginForm />
    </main>
  );
}
```

### `di/` — Dependency Injection Wiring

Child container, tokens, registrations. The module's composition root. The only place that knows concrete implementations.

```
src/modules/auth/di/
  container.ts             # Child container — createChildContainer()
  tokens.ts                # Symbol tokens for all registrations
```

```ts
// src/modules/auth/di/tokens.ts
import type { InjectionToken } from 'tsyringe';
import type { AuthService } from '../application/auth.service';
import type { AuthRepository } from '../repositories/auth.repository';
import type { AuthStore } from '../stores/auth.store';

export const AUTH_TOKENS = {
  AuthService: Symbol.for('auth.AuthService') as unknown as InjectionToken<AuthService>,
  AuthRepository: Symbol.for('auth.AuthRepository') as unknown as InjectionToken<AuthRepository>,
  AuthStore: Symbol.for('auth.AuthStore') as unknown as InjectionToken<AuthStore>,
} as const;
```

```ts
// src/modules/auth/di/container.ts
import { container as rootContainer } from '@/app/_lib/di/container';
import { AUTH_TOKENS } from './tokens';
import { AuthServiceImpl } from '../application/auth.service';
import { AuthRepository } from '../repositories/auth.repository';
import { createAuthStore } from '../stores/auth.store';

export function initializeAuthModule(): void {
  const childContainer = rootContainer.createChildContainer();

  // Repositories — data fetching
  childContainer.register(AUTH_TOKENS.AuthRepository, {
    useClass: AuthRepository,
  });

  // Application — services
  childContainer.register(AUTH_TOKENS.AuthService, {
    useClass: AuthServiceImpl,
  });

  // Stores — created once via factory, reused as singleton
  childContainer.register(AUTH_TOKENS.AuthStore, {
    useFactory: (c) => {
      const authService = c.resolve<AuthService>(AUTH_TOKENS.AuthService);
      return createAuthStore(authService);
    },
  });
}
```

### `index.ts` — Public API Barrel

Re-exports only what other modules may use. No implementation.

```ts
// src/modules/auth/index.ts
export { AUTH_TOKENS } from './di/tokens';
export { initializeAuthModule } from './di/container';
export type { AuthService } from './application/auth.service';
export type { AuthRepository } from './repositories/auth.repository';
export type { UserDto, LoginResponse } from './types/responses';

// Schemas — exported so other modules can validate cross-cutting inputs
export { loginSchema } from './schemas/login.schema';
export type { LoginRequest } from './schemas/login.schema';

// UI — exported so app/ routes can compose
export { LoginPage } from './ui/login-page';
export { RegisterForm } from './ui/register-form';
export { AuthGuard } from './ui/auth-guard';
export { LogoutButton } from './ui/logout-button';
```

## Full Module Layout

```
src/modules/auth/
  schemas/
    login.schema.ts
    register.schema.ts
    change-password.schema.ts
  types/
    responses.ts
    errors.ts
  stores/
    auth.store.ts
  application/
    auth.service.ts
  repositories/
    auth.repository.ts
  ui/
    login-page.tsx
    login-form.tsx
    register-form.tsx
    user-avatar.tsx
    auth-guard.tsx
    logout-button.tsx
  di/
    tokens.ts
    container.ts
  index.ts
```

## Module Layer Order

```
schemas/     →  depends on nothing (pure Zod)
types/       →  depends on nothing (pure TS interfaces)
stores/      →  depends on application/, types/
hooks/       →  depends on application/, types/
application/ →  depends on types/, schemas/, repositories/
repositories/ → depends on types/, schemas/, shared (axios, logger)
ui/          →  depends on stores/, hooks/, schemas/
di/          →  depends on everything — wires the module
```

`stores/` and `hooks/` sit between `application/` and `ui/`. They resolve services from DI, hold client state, and expose query/mutation hooks. UI components consume them — never call services directly (use store actions) and never call `useQuery`/`useMutation` directly (use hooks).

## Type Visibility Summary

| Layer | Visibility | Imported By |
|-------|-----------|-------------|
| `schemas/` Zod schemas | `export` | `ui/` (form validation), `application/` (service inputs), other modules' `di/` |
| `schemas/` inferred types | `export` | `repositories/` (request types), `ui/`, `application/` |
| `types/` response interfaces | `export` | `application/`, `repositories/`, `ui/` |
| `types/` error shapes | `export` | `application/`, `ui/` |
| `stores/` | `export` via barrel | `ui/`, other modules' `di/` |
| `hooks/` | `export` via barrel | `ui/` |
| `application/` service interface | `export` | `stores/`, `hooks/`, `ui/`, other modules' `di/` |
| `application/` service impl | `export` but only `di/` imports it | `di/container.ts` |
| `repositories/` classes | `export` | `di/` (wires), `application/` (imports class type), other modules' `di/` |
| `ui/` components | `export` | `app/` routes, other modules' layouts |
| `di/tokens.ts` | `export` | `stores/`, `hooks/`, `ui/` (useResolve), other modules' `di/` |
| `di/container.ts` init fn | `export` | root `DIProvider` |

---

# The `app/` Directory — Thin Routing Shell

## Principle

The `app/` directory **only** contains routing files (`page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`) and shared infrastructure (`_lib/`). No business logic. No API calls. No component implementations.

Every page is a thin wrapper:

```tsx
// app/(auth)/login/page.tsx
import type { Metadata } from 'next';
import { LoginPage } from '@/modules/auth';

export const metadata: Metadata = {
  title: 'Sign In',
  description: 'Sign in to your account to access your dashboard.',
};

export default LoginPage;
```

That's it. One import, one export. Metadata resolved server-side, component renders client-side. Pages are the thinnest possible shell.

## Route Groups

Use route groups `(group)` to organize without affecting URLs:

```
app/
  (auth)/
    login/page.tsx
    register/page.tsx
    layout.tsx                  # Auth layout — centered card, no nav
  (dashboard)/
    page.tsx
    settings/page.tsx
    layout.tsx                  # Dashboard layout — sidebar + header
  layout.tsx                    # Root layout — html, body, DIProvider
```

## Private Folders (`_` prefix)

Excluded from routing. Use for shared infrastructure inside `app/`:

```
app/
  _lib/
    di/
      container.ts              # Root tsyringe container
      use-resolve.ts            # useResolve hook
      provider.tsx              # DI initialization provider
    components/                 # Shared UI (header, footer, sidebar)
```

---

# Dependency Injection — tsyringe

## Setup

```bash
bun add tsyringe reflect-metadata
bun add @tanstack/react-form @tanstack/zod-form-adapter
bun add @tanstack/react-query
bun add tailwindcss @tailwindcss/postcss postcss clsx
bun add @t3-oss/env-nextjs zod
bun add axios
bun add pino
bun add -D @types/pino
bun add next-intl
bun add zustand
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

## QueryClientProvider

TanStack Query needs a `QueryClientProvider` wrapping the app. Configure default options — no retry on mutations, no auto-refetch on window focus for mutations.

```tsx
// app/_lib/query/provider.tsx
'use client';

import { QueryClient, QueryClientProvider, type QueryClientConfig } from '@tanstack/react-query';
import { useState, type ReactNode } from 'react';

function makeQueryClient(config?: QueryClientConfig): QueryClient {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 30_000,
        retry: 1,
        refetchOnWindowFocus: true,
      },
      mutations: {
        retry: false,
      },
    },
    ...config,
  });
}

let browserQueryClient: QueryClient | undefined;

function getQueryClient(): QueryClient {
  if (typeof window === 'undefined') {
    return makeQueryClient();
  }
  if (!browserQueryClient) {
    browserQueryClient = makeQueryClient();
  }
  return browserQueryClient;
}

interface QueryProviderProps {
  children: ReactNode;
}

export function QueryProvider({ children }: QueryProviderProps) {
  const queryClient = getQueryClient();

  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}
```

Import `reflect-metadata` once in the DI container file:

```ts
// app/_lib/di/container.ts
import 'reflect-metadata';
import { container } from 'tsyringe';

export { container };
```

## `useResolve` Hook

The single entry point for components to get services.

```ts
// app/_lib/di/use-resolve.ts
'use client';

import { useMemo } from 'react';
import { container, type InjectionToken } from 'tsyringe';

export function useResolve<T>(token: InjectionToken<T>): T {
  return useMemo(() => container.resolve(token), [token]);
}
```

**Why `useMemo`**: `container.resolve` creates a new instance on every call (unless `@singleton`). Without `useMemo`, every render = new resolve = lost state, wasted memory.

**React Compiler interaction**: With `reactCompiler: true`, the compiler auto-memoizes components. But it cannot know `container.resolve` is a stable function — explicit `useMemo` is still correct.

**Token stability**: Token must be stable across renders — Symbol, class reference. Never create tokens inline in render.

## `DIProvider` — Module Initialization

```tsx
// app/_lib/di/provider.tsx
'use client';

import { useEffect, type ReactNode } from 'react';
import { initializeAllModules } from '@/modules/registry';

interface DIProviderProps {
  children: ReactNode;
}

export function DIProvider({ children }: DIProviderProps) {
  useEffect(() => {
    initializeAllModules();
  }, []);

  return <>{children}</>;
}
```

## Module Registry

Single entry point that initializes all modules in dependency order.

```ts
// src/modules/registry.ts
import { initializeSharedModule } from '@/modules/shared';
import { initializeAuthModule } from '@/modules/auth';
import { initializeDashboardModule } from '@/modules/dashboard';
import { initializeBillingModule } from '@/modules/billing';

export function initializeAllModules(): void {
  initializeSharedModule();       // Logger + ApiClient first — everything depends on them
  initializeAuthModule();         // no deps
  initializeDashboardModule();    // depends on auth
  initializeBillingModule();      // depends on auth, dashboard
}
```

## Root Layout — Wire Providers

```tsx
// app/[locale]/layout.tsx
import { NextIntlClientProvider } from 'next-intl';
import { QueryProvider } from '@/app/_lib/query/provider';
import { DIProvider } from '@/app/_lib/di/provider';
import type { LayoutProps } from '@/app/[locale]/layout';

export default async function RootLayout({
  children,
  params,
}: LayoutProps<'/[locale]'>) {
  const { locale } = await params;

  return (
    <html lang={locale}>
      <body>
        <NextIntlClientProvider>
          <QueryProvider>
            <DIProvider>
              {children}
            </DIProvider>
          </QueryProvider>
        </NextIntlClientProvider>
      </body>
    </html>
  );
}
```

## Per-Module Child Containers

Each module creates a child container from the root. Isolates registrations, enforces module boundaries.

```
rootContainer
  ├── authContainer        (AuthService, AuthRepository)
  ├── dashboardContainer   (DashboardService, DashboardApiClient)
  └── billingContainer     (BillingService, BillingApiClient)
```

Child container can resolve from parent. But `authContainer` cannot resolve `DashboardService` — module isolation preserved.

## Token Naming Convention

```
<module>.<Type>
```

```ts
export const AUTH_TOKENS = {
  AuthService: Symbol.for('auth.AuthService'),
  AuthRepository: Symbol.for('auth.AuthRepository'),
} as const;
```

`Symbol.for()` — globally keyed symbols. Consistent across child containers and HMR cycles.

---

# Shared Kernel — `src/modules/shared/`

## Logger — `ILogger` Interface + `PinoLogger` Implementation

### Why Interface-Abstracted

Modules depend on `ILogger`, never `pino.Logger`. Swap pino for winston/bunyan/console without touching module code. Same pattern as `axios` — the concrete tool is an infrastructure detail.

### `ILogger` — Contract

```ts
// src/modules/shared/types/logger.ts
export interface ILogger {
  debug(obj: Record<string, unknown>, msg?: string): void;
  info(obj: Record<string, unknown>, msg?: string): void;
  warn(obj: Record<string, unknown>, msg?: string): void;
  error(obj: Record<string, unknown>, msg?: string): void;
  child(bindings: Record<string, unknown>): ILogger;
}
```

No `trace`/`fatal` — four levels enough for client-frontend. Every method takes object as first argument — structured logging, not string interpolation. **No string-first overload**: `logger.info('user logged in')` banned. Always pass context: `logger.info({ userId: 'abc' }, 'user logged in')`.

### Token

```ts
// src/modules/shared/di/tokens.ts
import type { InjectionToken } from 'tsyringe';
import type { AxiosInstance } from 'axios';
import type { ILogger } from '../types/logger';

export const SHARED_TOKENS = {
  ApiClient: Symbol.for('shared.ApiClient') as unknown as InjectionToken<AxiosInstance>,
  Logger: Symbol.for('shared.Logger') as unknown as InjectionToken<ILogger>,
} as const;
```

### `PinoLogger` — Implementation

```ts
// src/modules/shared/infrastructure/pino-logger.ts
import pino, { type Logger as PinoInstance } from 'pino';
import type { ILogger } from '../types/logger';

const isBrowser = typeof window !== 'undefined';
const isProd = process.env.NODE_ENV === 'production';

function createRootPino(): PinoInstance {
  if (isBrowser) {
    return pino({
      level: isProd ? 'info' : 'debug',
      transport: isProd
        ? undefined
        : {
            target: 'pino/browser',
            options: {
              transmit: {
                send: (_level: string, logEvent: { messages: unknown[] }) => {
                  const [obj, msg] = logEvent.messages;
                  const prefix = obj && typeof obj === 'object' && 'module' in obj
                    ? `[${(obj as Record<string, string>).module}]`
                    : '';
                  console.log(prefix, msg ?? obj);
                },
              },
            },
          },
      browser: { asObject: true },
    });
  }

  return pino({
    level: isProd ? 'info' : 'debug',
    ...(isProd
      ? {}
      : {
          transport: {
            target: 'pino-pretty',
            options: { colorize: true, translateTime: 'HH:MM:ss', ignore: 'pid,hostname' },
          },
        }),
  });
}

const rootPino = createRootPino();

export class PinoLogger implements ILogger {
  constructor(private readonly pino: PinoInstance) {}

  debug(obj: Record<string, unknown>, msg?: string): void { this.pino.debug(obj, msg); }
  info(obj: Record<string, unknown>, msg?: string): void { this.pino.info(obj, msg); }
  warn(obj: Record<string, unknown>, msg?: string): void { this.pino.warn(obj, msg); }
  error(obj: Record<string, unknown>, msg?: string): void { this.pino.error(obj, msg); }
  child(bindings: Record<string, unknown>): ILogger {
    return new PinoLogger(this.pino.child(bindings));
  }
}

export const rootLogger: ILogger = new PinoLogger(rootPino);
```

### DI Registration — Logger First

```ts
// src/modules/shared/di/container.ts
import { container as rootContainer } from '@/app/_lib/di/container';
import { env } from '@/env';
import axios from 'axios';
import { SHARED_TOKENS } from './tokens';
import { rootLogger } from '../infrastructure/pino-logger';

export function initializeSharedModule(): void {
  // Logger — register first, everything depends on it
  rootContainer.register(SHARED_TOKENS.Logger, { useValue: rootLogger });

  // Axios — singleton apiClient
  const apiClient = axios.create({
    baseURL: env.NEXT_PUBLIC_API_BASE_URL,
    withCredentials: true,
    headers: { 'Content-Type': 'application/json' },
  });

  apiClient.interceptors.request.use((config) => {
    const token = getAuthToken();
    if (token) config.headers.Authorization = `Bearer ${token}`;
    return config;
  });

  apiClient.interceptors.response.use(
    (res) => res,
    async (error) => {
      if (error.response?.status === 401 && !error.config._retry) {
        error.config._retry = true;
      }
      return Promise.reject(error);
    },
  );

  rootContainer.register(SHARED_TOKENS.ApiClient, { useValue: apiClient });
}

function getAuthToken(): string | null {
  return null;
}
```

### Logger Rules

| Rule | Why |
|------|-----|
| Modules depend on `ILogger`, never `pino.Logger` | Swap implementation without touching module code |
| Always pass object as first arg: `logger.info({ userId }, msg)` | Structured logs. Greppable. No string-only logs. |
| `logger.child({ module })` in constructor | Every log auto-tags its module. No manual `module:` strings. |
| `logger.child({ layer })` for application vs repository | Debug filtering: `layer: 'repository'` shows HTTP logs only. |
| One `.child()` per class in constructor | Bind once. Use through the instance's `this.log`. |
| Browser dev: pretty console with `[module]` prefix | No raw JSON in devtools. `[auth] login success`. |
| Production: structured JSON via pino transport | Collect to log aggregator (Datadog, Grafana, etc.) |
| `SHARED_TOKENS.Logger` registered first | Everything depends on it. Register before axios, before modules. |

## Axios Instance

One pre-configured axios instance. Every API client imports it. Handles base URL, credentials, auth headers via interceptor.

**What the singleton `apiClient` provides**:
- Single axios instance — every API client injects the same instance via DI
- `baseURL` from `env.NEXT_PUBLIC_API_BASE_URL` — all paths relative, no URL construction
- `withCredentials` — cookies sent cross-origin
- Request interceptor — injects auth token, one place to change token source
- Response interceptor — 401 handling, token refresh, centralized
- API clients inject `SHARED_TOKENS.ApiClient` as `AxiosInstance` — no custom wrapper type

## Shared API Error Type

```ts
// src/modules/shared/types/api-error.ts
export interface ApiError {
  code: string;
  message: string;
  field?: string;
}
```

## Axios Interceptor — Error Normalization

Add a response interceptor to normalize backend errors into `ApiError`:

```ts
// In shared/api-client.ts response interceptor
apiClient.interceptors.response.use(
  (res) => res,
  (error: AxiosError<ApiError>) => {
    if (error.response) {
      const apiError: ApiError = {
        code: error.response.data?.code ?? 'http.error',
        message: error.response.data?.message ?? error.message,
        field: error.response.data?.field,
      };
      return Promise.reject(apiError);
    }
    return Promise.reject({
      code: 'network.error',
      message: error.message,
    } satisfies ApiError);
  },
);
```

Components catch `ApiError` — always the same shape, whether backend error or network failure.

## Shared Pagination Types

```ts
// src/modules/shared/types/pagination.ts

export interface PaginationParams {
  page: number;
  pageSize: number;
}

export function createPaginationParams(page?: number, pageSize?: number): PaginationParams {
  return {
    page: Math.max(1, Math.min(page ?? 1, 50)),
    pageSize: Math.max(1, Math.min(pageSize ?? 20, 100)),
  };
}

export interface PaginationMetadata {
  page: number;
  pageSize: number;
  totalItems: number;
  totalPages: number;
}

export interface PaginatedResponse<T> {
  data: T[];
  pagination: PaginationMetadata;
}
```

## FormField & FieldError — Shared UI Components

Form fields use reusable components, not raw input JSX. `FieldError` renders ALL errors for a field, not just the first one. Zod collects every failure — UX matches.

### `FieldError`

```tsx
// src/modules/shared/ui/field-error.tsx
'use client';

import { cn } from '@/modules/shared/utils/cn';

interface FieldErrorProps {
  errors: unknown[] | undefined;
  className?: string;
}

export function FieldError({ errors, className }: FieldErrorProps) {
  if (!errors?.length) return null;

  return (
    <ul className={cn('text-xs text-destructive flex flex-col gap-0.5 mt-1', className)}>
      {errors.map((err, i) => {
        const message =
          typeof err === 'string'
            ? err
            : typeof err === 'object' && err !== null && 'message' in err
              ? (err as { message: string }).message
              : null;
        if (!message) return null;
        return (
          <li key={i} className="flex items-start gap-1">
            <span aria-hidden="true" className="select-none">&bull;</span>
            <span>{message}</span>
          </li>
        );
      })}
    </ul>
  );
}
```

**Why `unknown[]`**: TanStack Form's `field.state.meta.errors` is `unknown[]`. Zod issues are objects with `.message`, string error maps return `string[]`. The component normalizes both. No cast at call site.

### `FormField`

```tsx
// src/modules/shared/ui/form-field.tsx
'use client';

import type { ReactNode } from 'react';
import { cn } from '@/modules/shared/utils/cn';
import { FieldError } from './field-error';

interface FormFieldProps {
  field: {
    name: string;
    state: {
      value: string;
      meta: { errors: unknown[] };
    };
    handleChange: (value: string) => void;
    handleBlur: () => void;
  };
  label: string;
  type?: 'text' | 'email' | 'password' | 'number' | 'tel' | 'url';
  autoComplete?: string;
  placeholder?: string;
  disabled?: boolean;
  rightLabel?: ReactNode;
  rightElement?: ReactNode;
  className?: string;
}

export function FormField({
  field,
  label,
  type = 'text',
  autoComplete,
  placeholder,
  disabled = false,
  rightLabel,
  rightElement,
  className,
}: FormFieldProps) {
  const hasErrors = field.state.meta.errors.length > 0;

  return (
    <div className="flex flex-col gap-1.5">
      <div className={rightLabel ? 'flex items-center justify-between' : ''}>
        <label htmlFor={field.name} className="text-sm font-medium text-foreground">
          {label}
        </label>
        {rightLabel}
      </div>

      <div className={rightElement ? 'relative' : ''}>
        <input
          id={field.name}
          name={field.name}
          type={type}
          required
          autoComplete={autoComplete}
          placeholder={placeholder ?? label}
          disabled={disabled}
          value={field.state.value}
          onChange={(e) => field.handleChange(e.target.value)}
          onBlur={field.handleBlur}
          className={cn(
            'w-full h-9 rounded-md border bg-background px-3 py-1 text-sm text-foreground',
            'placeholder:text-muted-foreground',
            'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2',
            'disabled:cursor-not-allowed disabled:opacity-50',
            hasErrors && 'border-destructive focus-visible:ring-destructive',
            rightElement && 'pr-10',
            className,
          )}
        />
        {rightElement}
      </div>

      <FieldError errors={field.state.meta.errors} />
    </div>
  );
}
```

**Key details**:
- `noValidate` on `<form>` — disables browser's native validation bubble. Zod owns validation.
- `field.handleBlur` passed to input — TanStack Form marks field as touched, triggers validation display.
- `hasErrors` on input border — destructive border + ring color. Field visually "breaks" alongside error list.
- `<li>&bull;</li>` bullet — accessible (`aria-hidden`), non-selectable texture. Each error on its own line.
- Server error (top `error` state) uses `destructive/10` bg — distinct from field errors. Server error = banner. Field error = inline list.
- `FormField` is **pure presentational**. No logic. No Zod. Just wiring TanStack Form's `field` object to shadcn-styled DOM.

## What Belongs in Shared

- `ILogger` + `PinoLogger` — interface-abstracted logger, DI-wired singleton
- `apiClient` — pre-configured axios instance
- `ApiError` type
- `PaginationParams`, `PaginationMetadata`, `PaginatedResponse<T>` types
- `useResolve` hook
- DI infrastructure (root container)
- `FormField` + `FieldError` — reusable form field components
- Utility functions (`cn`, `formatDate`, etc.)
- Reusable hooks (`useDebounce`, `useMediaQuery`)

## What Does NOT Belong in Shared

- Any business schema or response type
- Any module-specific service
- Any module-specific component
- Any module-specific API client
- Any business rule

## Barrel

```ts
// src/modules/shared/index.ts
export { SHARED_TOKENS } from './di/tokens';
export { initializeSharedModule } from './di/container';
export type { ILogger } from './types/logger';
export type { ApiError } from './types/api-error';
export { FormField } from './ui/form-field';
export { FieldError } from './ui/field-error';
```

---

# Cross-Module Access

## Allowed

Other modules import from `index.ts` (barrel) only. Never from `application/`, `repositories/`, `schemas/`, `types/`, or `ui/` directly.

```ts
// src/modules/dashboard/application/dashboard.service.ts
import type { AuthService } from '@/modules/auth';
import { AUTH_TOKENS } from '@/modules/auth';
import { injectable } from 'tsyringe';

@injectable()
export class DashboardServiceImpl {
  constructor(
    private readonly authService: AuthService,
    private readonly dashboardRepo: DashboardRepository,
  ) {}

  async getContext(): Promise<DashboardContext> {
    const user = this.authService.getCurrentUser(); // cross-module call
    const stats = await this.dashboardRepo.getStats();
    return { user, stats };
  }
}
```

Wired in `dashboard/di/container.ts`:

```ts
import { AUTH_TOKENS } from '@/modules/auth';

export function initializeDashboardModule(): void {
  const childContainer = rootContainer.createChildContainer();

  childContainer.register<AuthService>(AUTH_TOKENS.AuthService, {
    useFactory: (c) => c.resolve(AUTH_TOKENS.AuthService),
  });

  childContainer.register(DASHBOARD_TOKENS.DashboardService, {
    useClass: DashboardServiceImpl,
  });
}
```

## Forbidden

| Forbidden | Why |
|-----------|-----|
| Importing `repositories/` from another module | Bypasses contracts |
| Importing `application/` from another module | Bypasses barrel |
| Importing `ui/` from another module | Compose via page routes, not direct imports |
| Importing `schemas/` or `types/` from another module directly | Use barrel — each module curates its public types |
| Calling `axios`/`fetch` directly in `ui/` or `application/` | Bypasses API client contract |

---

# Page Pattern — Thin Shell

Every page follows the same pattern:

```tsx
// app/(dashboard)/page.tsx
import type { Metadata } from 'next';
import { DashboardPage } from '@/modules/dashboard';

export const metadata: Metadata = { title: 'Dashboard' };
export default DashboardPage;
```

With layout:

```tsx
// app/(dashboard)/layout.tsx
import { AuthGuard } from '@/modules/auth';

export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <AuthGuard>
      <DashboardShell>{children}</DashboardShell>
    </AuthGuard>
  );
}
```

**Rule**: Pages are one-liners with optional static metadata. Layouts compose module guards and shells. Loading states are module skeletons. Zero business logic at the route level.

---

# Data Flow

```
User action (click, submit)
  → ui/ form validates input with Zod schema
    → ui/ calls store action or hook
      → stores/ calls application/ service
        → application/ service calls repository
          → repositories/ calls apiClient (axios)
            → external API
          ← response data
        ← service unpacks DTO, returns what store needs
      ← store updates state, hook caches result
    ← component re-renders with new state
```

Every step is a typed contract. No layer crosses another's boundary without conversion.

```
ui/           knows: store hooks, query hooks, Zod schemas, FormField
stores/       knows: service interface, types
hooks/        knows: service interface, types, query keys
application/  knows: response types, repository type, schema-inferred request types, ILogger
repositories/ knows: axios apiClient, response types, request types, ILogger
schemas/      knows: nothing but Zod
types/        knows: nothing but its own interfaces
```

---

# Zustand — State Management

## When to Use Zustand

Zustand stores **client state** — the kind of state that lives beyond a single component's lifecycle and is shared across multiple components in a module.

| Zustand | React `useState` | tsyringe Service |
|---------|-----------------|-----------------|
| Cross-component state | Single-component state | Business logic, orchestration |
| Global or module-scoped | Local to one component | Stateless or API-backed |
| Subscriptions + selectors | Direct setter | Async calls, validation |
| Persisted to localStorage | Dies with unmount | No state of its own |
| Optimistic updates | Form inputs | API calls |

**Use Zustand when**: multiple components need the same state, state survives navigation, or you need fine-grained subscriptions (one component re-renders on `user.name` only, not the entire user object).

**Use `useState` when**: state is local to one component (form inputs, toggle open/closed, single-use loading flag).

**Use tsyringe service when**: the operation is "call the backend and return a result" — services own business logic, not UI state.

## Where Stores Live

Stores live in `stores/` at module root — same level as `application/`, `repositories/`, `ui/`. They depend on services, consumed by components: not a UI subfolder, not an application detail. Own layer.

```
src/modules/auth/
  stores/
    auth.store.ts          # Auth state — user, tokens, login status
```

For cross-module state (e.g., current user), export the store via the module's barrel:

```ts
// src/modules/auth/index.ts
export { useAuthStore } from './stores/auth.store';
```

Shared/generic stores live in `src/modules/shared/stores/`:

```
src/modules/shared/
  stores/
    notification.store.ts  # Toast notifications — used by any module
    theme.store.ts          # Dark/light mode
```

## Store Naming

- **Store file**: `${name}.store.ts` — `auth.store.ts`, `dashboard.store.ts`
- **Store hook**: `useXxxStore` — `useAuthStore`, `useDashboardStore`
- **State type**: `XxxState` — `AuthState`, `DashboardState`
- **Actions**: verb or `set` prefix — `login`, `logout`, `setUser`, `addItem`, `removeItem`

## Store Factory Pattern — Injecting Services

Stores that need services (API calls, business logic) receive them via a factory function. The store itself is created at DI initialization time, then registered as a singleton in the child container.

```ts
// src/modules/auth/stores/auth.store.ts
import { create } from 'zustand';
import type { UserDto } from '../types/responses';
import type { AuthService } from '../application/auth.service';

interface AuthState {
  user: UserDto | null;
  isLoading: boolean;
  error: string | null;

  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  setUser: (user: UserDto | null) => void;
  clearError: () => void;
}

export function createAuthStore(authService: AuthService) {
  return create<AuthState>((set) => ({
    user: null,
    isLoading: false,
    error: null,

    login: async (email, password) => {
      set({ isLoading: true, error: null });
      try {
        const user = await authService.login({ email, password });
        set({ user, isLoading: false });
      } catch (err) {
        set({
          error: err instanceof Error ? err.message : 'Login failed',
          isLoading: false,
        });
      }
    },

    logout: async () => {
      await authService.logout();
      set({ user: null });
    },

    setUser: (user) => set({ user }),
    clearError: () => set({ error: null }),
  }));
}

export type AuthStore = ReturnType<typeof createAuthStore>;
```

**Why factory and not `useAuthStore.getState()` inside the service**: The service doesn't know about stores. The store knows about the service. Dependency direction: `stores/ → application/`. Never the reverse.

## Register Store in DI Container

```ts
// src/modules/auth/di/container.ts
import { createAuthStore } from '../stores/auth.store';

export function initializeAuthModule(): void {
  const childContainer = rootContainer.createChildContainer();

  childContainer.register(AUTH_TOKENS.AuthRepository, { useClass: AuthRepository });
  childContainer.register(AUTH_TOKENS.AuthService, { useClass: AuthServiceImpl });

  // Stores — created once via factory, reused as singleton
  childContainer.register(AUTH_TOKENS.AuthStore, {
    useFactory: (c) => {
      const authService = c.resolve<AuthService>(AUTH_TOKENS.AuthService);
      return createAuthStore(authService);
    },
  });
}
```

## Using Stores in Components

Components resolve the store hook via `useResolve`, then use it like any Zustand store:

```tsx
'use client';
import { useResolve } from '@/app/_lib/di/use-resolve';
import { AUTH_TOKENS } from '../di/tokens';
import type { AuthStore } from '../stores/auth.store';

export function UserAvatar() {
  const useAuthStore = useResolve<AuthStore>(AUTH_TOKENS.AuthStore);
  const user = useAuthStore((s) => s.user);
  const logout = useAuthStore((s) => s.logout);

  if (!user) return null;
  return (
    <div>
      <span>{user.email}</span>
      <button onClick={logout}>Sign Out</button>
    </div>
  );
}
```

## Store Persistence — `zustand/middleware`

```ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import type { AuthService } from '../application/auth.service';

export function createAuthStore(authService: AuthService) {
  return create<AuthState>()(
    persist(
      (set) => ({ /* ... */ }),
      {
        name: 'auth-store',
        partialize: (state) => ({ user: state.user }),
      },
    ),
  );
}
```

**Rules for `persist`**:
- Only persist session identity (`user`, `token` existence flag) — never sensitive data
- `partialize` to pick only what's needed on reload
- Tokens live in axios interceptor (cookie or memory), not in persisted Zustand
- Clear persisted state on logout: `useAuthStore.persist.clearStorage()`

## Store Location Summary

| Store Type | Location | Registered In | Consumed By |
|-----------|----------|--------------|-------------|
| Module store | `modules/<name>/stores/<name>.store.ts` | `modules/<name>/di/container.ts` | Module's `ui/` components |
| Shared store | `modules/shared/stores/<name>.store.ts` | `modules/shared/di/container.ts` | Any module's `ui/` components |
| Cross-module access | Via barrel `index.ts` of owning module | Owning module's DI | Other modules' components + DI |

## Store Rules

| Rule | Why |
|------|-----|
| Store factory receives services as parameter | DI direction: store → service. Store doesn't import container. |
| Store factory is called in `di/container.ts` | Single creation. Registered as singleton. |
| Store type exported as `ReturnType<typeof createXxxStore>` | Components need the hook type for `useResolve`. |
| Selectors for fine-grained subscriptions | `useAuthStore((s) => s.user)` avoids re-render on unrelated state changes. |
| Actions are thin — call service, set state | Business logic lives in `application/`, not in stores. |
| Never call `useXxxStore.getState()` inside services | Dependency direction. Service doesn't know about UI state. |
| `persist.clearStorage()` on logout | Stale user data must not survive logout. |

---

# TanStack Query — Server State

## Why TanStack Query

TanStack Query owns **server state** — data fetched from the API. It handles caching, background refetch, stale-while-revalidate, pagination, and optimistic mutations. Zustand handles client state; TanStack Query handles everything that originates from the backend.

```
TanStack Query → server state (API data, pagination, cache)
Zustand        → client state (UI state, session identity, filters)
useState       → local state (form inputs toggle, one-shot flags)
```

## DI Setup

TanStack Query needs no per-module DI registration. `QueryClient` is created once in `QueryProvider`.

## Query Keys — Per-Module Key Factory

```ts
// src/modules/orders/types/query-keys.ts
export const orderKeys = {
  all: ['orders'] as const,
  lists: () => [...orderKeys.all, 'list'] as const,
  list: (filters: OrderFilters) => [...orderKeys.lists(), filters] as const,
  details: () => [...orderKeys.all, 'detail'] as const,
  detail: (id: string) => [...orderKeys.details(), id] as const,
};
```

## Query Hooks — In `hooks/`

Each module exposes `useXxx` hooks that wrap TanStack Query. Hooks live in `hooks/` at module root — same level as `stores/`, `application/`, `ui/`. Own adapter layer.

```
src/modules/orders/
  hooks/
    use-order-list.ts       # useOrderList — paginated list
    use-order-detail.ts     # useOrderDetail — single order
    use-create-order.ts     # useCreateOrder — mutation
```

### List Hook — `useInfiniteQuery`

```ts
// src/modules/orders/hooks/use-order-list.ts
import { useInfiniteQuery } from '@tanstack/react-query';
import { useResolve } from '@/app/_lib/di/use-resolve';
import { ORDER_TOKENS } from '../../di/tokens';
import type { OrderService } from '../../application/order.service';
import type { OrderFilters } from '../../types/filters';
import { createPaginationParams } from '@/modules/shared/types/pagination';

interface UseOrderListOptions {
  filters: OrderFilters;
  pageSize?: number;
}

export function useOrderList({ filters, pageSize = 20 }: UseOrderListOptions) {
  const orderService = useResolve<OrderService>(ORDER_TOKENS.OrderService);

  return useInfiniteQuery({
    queryKey: ['orders', 'list', filters],
    queryFn: async ({ pageParam }: { pageParam: number }) => {
      const params = createPaginationParams(pageParam, pageSize);
      return orderService.list(filters, params);
    },
    initialPageParam: 1,
    getNextPageParam: (lastPage) => {
      const { page, totalPages } = lastPage.pagination;
      return page < totalPages ? page + 1 : undefined;
    },
    staleTime: 30_000,
  });
}
```

### Mutation Hook

```ts
// src/modules/orders/hooks/use-create-order.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useResolve } from '@/app/_lib/di/use-resolve';
import { ORDER_TOKENS } from '../../di/tokens';
import type { OrderService } from '../../application/order.service';
import type { CreateOrderRequest } from '../../schemas/create-order.schema';

export function useCreateOrder() {
  const orderService = useResolve<OrderService>(ORDER_TOKENS.OrderService);
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (input: CreateOrderRequest) => orderService.create(input),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['orders', 'list'] });
    },
  });
}
```

## TanStack Query Rules

| Rule | Why |
|------|-----|
| Query hooks live in `hooks/` | Adapter layer between application and UI. Components consume them directly. |
| One query hook per query type | `useOrderList`, `useOrderDetail` — not one generic `useQuery` wrapper. |
| Query keys use `[module, entity, ...params]` | Cache invalidation by prefix. |
| `useInfiniteQuery` for paginated lists | `fetchNextPage` + `hasNextPage` built in. |
| `PaginationParams` / `PaginationMetadata` are transport-agnostic | Shared types. No HTTP details. |
| `staleTime` per query type | Lists: 30s. Detail: 60s. Invalidate on mutations. |
| Mutation `onSuccess` invalidates affected query keys | Create/update/delete invalidates the list. |
| Query hooks don't call `useResolve` inside `queryFn` | `useResolve` called once at component level, values captured in closure. |

---

# Tailwind CSS — Styling

## Principle

Tailwind is the project's styling layer. No CSS modules, no styled-components, no inline styles. Every component uses Tailwind utility classes.

## Setup

```bash
bun add tailwindcss @tailwindcss/postcss postcss
```

Next.js 16 uses Tailwind v4 with the PostCSS plugin. Tailwind v4 is CSS-first — no `tailwind.config.ts`, no `content` globs. Theme is configured in CSS with `@theme`. Classes are auto-detected from source files.

```ts
// postcss.config.mjs
const config = {
  plugins: {
    '@tailwindcss/postcss': {},
  },
};
export default config;
```

```css
/* app/globals.css */
@import 'tailwindcss';

@theme {
  --color-brand-50: #eef2ff;
  --color-brand-500: #6366f1;
  --color-brand-700: #4338ca;
  --font-sans: 'Inter', system-ui, sans-serif;
}
```

## Tailwind Rules

| Rule | Why |
|------|-----|
| Utility classes in JSX only | No `@apply`, no CSS modules, no `styles.ts`. One system. |
| `className` order: layout → sizing → spacing → visual → state | Consistent scanning. |
| Theme in CSS `@theme` block — no JS config | Tailwind v4 is CSS-first. |
| Design tokens in `@theme` are project-wide only | Module-specific colors use built-in palette or raw hex. |
| No arbitrary values in production | `w-[327px]` is prototyping only. |
| Responsive: mobile-first | `sm:`, `md:`, `lg:` prefixes. Default classes = mobile. |
| `cn` for conditional classes | `cn('base-class', isActive && 'active-class')`. |
| Focus states on every interactive element | `focus-visible:outline-none focus-visible:ring-2` — accessibility baseline. |
| `disabled:` state on every button/input | `disabled:opacity-50 disabled:cursor-not-allowed`. |
| Tailwind v4: `@import 'tailwindcss'` | Replaces v3 `@tailwind base/components/utilities` directives. |

## `cn` Helper

```ts
// src/modules/shared/utils/cn.ts
import { clsx, type ClassValue } from 'clsx';

export function cn(...inputs: ClassValue[]): string {
  return clsx(inputs);
}
```

```bash
bun add clsx
```

---

# SEO — Metadata, Sitemap, Robots

## Principle

SEO lives in the `app/` directory. Title template, metadataBase, Open Graph defaults, robots.txt, and sitemap are route-level concerns — not module concerns. Modules export page components; the app layer wraps them with search-optimized metadata.

## How It Fits Client-First Architecture

Module page components are `'use client'`. But `metadata` export and `generateMetadata` are **Server Component only**. Solution: `app/` pages remain Server Components that export metadata + render the module's client component.

```tsx
// app/(auth)/login/page.tsx
import type { Metadata } from 'next';
import { LoginPage } from '@/modules/auth';

export const metadata: Metadata = {
  title: 'Sign In',
  description: 'Sign in to your account to access your dashboard.',
};

export default LoginPage;
```

## Root Layout — Global SEO Defaults

Root layout sets the base URL, title template, and Open Graph defaults. `viewport` is a separate named export in Next.js 14+.

```tsx
// app/[locale]/layout.tsx
import type { Metadata, Viewport } from 'next';
import { env } from '@/env';

export const viewport: Viewport = {
  width: 'device-width',
  initialScale: 1,
  themeColor: '#ffffff',
};

export const metadata: Metadata = {
  metadataBase: new URL(env.NEXT_PUBLIC_API_BASE_URL),
  title: {
    template: '%s | Acme',
    default: 'Acme — Project Management',
  },
  description: 'Acme helps teams ship faster with modular project management.',
  keywords: ['project management', 'team collaboration', 'productivity'],
  authors: [{ name: 'Acme Inc.' }],
  creator: 'Acme Inc.',
  publisher: 'Acme Inc.',
  formatDetection: { email: false, address: false, telephone: false },
  openGraph: {
    type: 'website',
    locale: 'en_US',
    siteName: 'Acme',
    title: 'Acme — Project Management',
    description: 'Acme helps teams ship faster with modular project management.',
    url: '/',
  },
  twitter: {
    card: 'summary_large_image',
    title: 'Acme — Project Management',
    description: 'Acme helps teams ship faster with modular project management.',
  },
  robots: { index: true, follow: true },
  icons: { icon: '/favicon.ico', apple: '/apple-touch-icon.png' },
  manifest: '/manifest.json',
};
```

**Metadata merging**: Child routes export `metadata.title` → template applied (`'%s | Acme'`). Child routes that DON'T export `openGraph` inherit root. Child routes that DO export `openGraph` **replace** entirely.

## Static vs Dynamic Metadata

### Static — `metadata` object

Use when metadata is known at build time. Always prefer this.

```tsx
// app/about/page.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'About Us',
  description: 'Learn about the Acme team.',
  alternates: { canonical: '/about' },
};
```

### Dynamic — `generateMetadata`

Use when metadata depends on route params or fetched data. `params` is `Promise<>` in Next.js 16.

```tsx
// app/projects/[id]/page.tsx
import type { Metadata, ResolvingMetadata } from 'next';

type Props = { params: Promise<{ id: string }> };

export async function generateMetadata(
  { params }: Props,
  parent: ResolvingMetadata,
): Promise<Metadata> {
  const { id } = await params;
  const project = await fetch(`${env.API_BASE_URL}/projects/${id}`).then(r => r.json());
  const previousImages = (await parent).openGraph?.images ?? [];

  return {
    title: project.name,
    description: project.summary,
    alternates: { canonical: `/projects/${id}` },
    openGraph: {
      title: project.name,
      description: project.summary,
      images: [project.coverImage, ...previousImages],
    },
  };
}
```

## Canonical URLs

Every page exports `alternates.canonical`. Prevents duplicate content penalties.

## `robots.ts` — Programmatic Rules

```ts
// app/robots.ts
import type { MetadataRoute } from 'next';
import { env } from '@/env';

export default function robots(): MetadataRoute.Robots {
  return {
    rules: { userAgent: '*', allow: '/', disallow: ['/api/', '/_next/'] },
    sitemap: `${env.NEXT_PUBLIC_API_BASE_URL}/sitemap.xml`,
  };
}
```

## `sitemap.ts` — Programmatic Generation

```ts
// app/sitemap.ts
import type { MetadataRoute } from 'next';
import { env } from '@/env';

export default function sitemap(): MetadataRoute.Sitemap {
  const baseUrl = env.NEXT_PUBLIC_API_BASE_URL;
  return [
    { url: baseUrl, lastModified: new Date(), changeFrequency: 'yearly', priority: 1 },
    { url: `${baseUrl}/about`, lastModified: new Date(), changeFrequency: 'monthly', priority: 0.8 },
  ];
}
```

## JSON-LD — Structured Data

For rich snippets, add JSON-LD components in `app/` — not inside modules. Structured data is a route concern.

```tsx
// app/projects/[id]/json-ld.tsx
export function JsonLd() {
  const data = {
    '@context': 'https://schema.org',
    '@type': 'WebApplication',
    name: 'Acme',
    description: 'Project management for teams',
    applicationCategory: 'ProjectManagementApplication',
  };

  return (
    <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify(data) }} />
  );
}
```

## SEO Rules

| Rule | Why |
|------|-----|
| `metadataBase` in root layout | All relative paths resolve to full URLs automatically |
| Title template in root layout | Consistent branding — `Page Title \| Acme` everywhere |
| Always export `alternates.canonical` per page | Prevents duplicate content from query params / trailing slashes |
| `robots.ts` disallows `/api/`, `/_next/` | Internal paths pollute search results |
| `sitemap.ts` returns `MetadataRoute.Sitemap` | Type-safe, cached by default, auto-format XML |
| JSON-LD in `app/` not in modules | Structured data describes the route, not the component |
| `viewport` is separate export, not in metadata | Next.js 14+ requirement — deprecated inside metadata |
| Static `metadata` over `generateMetadata` when possible | Zero runtime cost |
| `params` is `Promise<>` in Next.js 16 | Must `await` inside `generateMetadata` |

---

# Internationalization — `next-intl`

## Why `next-intl`

Client-first app still needs content translation. `next-intl` handles locale routing, ICU message syntax, and per-module translation JSON. Translations load async server-side in layouts, passed to client components via `NextIntlClientProvider`.

```bash
bun add next-intl
```

## Architecture: Server-Driven Messages

1. Root `layout.tsx` (Server Component) resolves locale + loads messages
2. Wraps children in `NextIntlClientProvider` with loaded messages
3. Client components call `useTranslations()` — sync, reads from context

No per-page `getTranslations`. No prop drilling. One provider, all components consume.

## File Structure

```
messages/
  en/
    common.json             # Shared: buttons, nav, errors, dates
    auth.json               # Auth module translations
    dashboard.json          # Dashboard module translations
  fr/
    common.json
    auth.json
    dashboard.json
```

**Per-module JSON files**, folder per locale. Developer opens `messages/en/auth.json` to edit auth strings — no scrolling through unrelated keys.

## Setup

### `src/i18n/routing.ts`

```ts
// src/i18n/routing.ts
import { defineRouting } from 'next-intl/routing';

export const routing = defineRouting({
  locales: ['en', 'fr', 'es'],
  defaultLocale: 'en',
  localePrefix: 'as-needed', // '/' for default locale, '/fr/' for others
});
```

### `proxy.ts` (Next.js 16)

```ts
// proxy.ts
import createIntlProxy from 'next-intl/proxy';
import { routing } from '@/i18n/routing';

export default createIntlProxy(routing);

export const config = {
  matcher: ['/((?!api|_next|_vercel|.*\\..*).*)'],
};
```

### `src/i18n/request.ts`

```ts
// src/i18n/request.ts
import { getRequestConfig } from 'next-intl/server';
import { routing } from './routing';

export default getRequestConfig(async ({ requestLocale }) => {
  let locale = await requestLocale;
  if (!locale || !routing.locales.includes(locale as (typeof routing.locales)[number])) {
    locale = routing.defaultLocale;
  }

  return {
    locale,
    messages: {
      common: (await import(`../../messages/${locale}/common.json`)).default,
      auth: (await import(`../../messages/${locale}/auth.json`)).default,
      dashboard: (await import(`../../messages/${locale}/dashboard.json`)).default,
    },
  };
});
```

## Translation Messages — Key Naming

### Key Structure: Page → Component → Key

```json
// messages/en/auth.json
{
  "LoginPage": {
    "title": "Sign In",
    "description": "Sign in to access your account",
    "LoginForm": {
      "emailLabel": "Email",
      "emailPlaceholder": "Email address",
      "passwordLabel": "Password",
      "passwordPlaceholder": "Password",
      "submitButton": "Sign In",
      "loadingButton": "Signing in...",
      "forgotPassword": "Forgot password?",
      "errors": {
        "invalidCredentials": "Invalid email or password",
        "networkError": "Connection failed. Try again."
      }
    }
  }
}
```

**Rule**: Keys mirror the component tree. `auth.LoginPage.LoginForm.submitButton` maps to `src/modules/auth/ui/login-form.tsx`. Missing key → instant file + path to fix.

### Shared/Common Keys

```json
// messages/en/common.json
{
  "buttons": { "cancel": "Cancel", "save": "Save", "delete": "Delete", "confirm": "Confirm" },
  "errors": { "required": "This field is required", "generic": "Something went wrong." },
  "navigation": { "dashboard": "Dashboard", "settings": "Settings", "profile": "Profile", "signOut": "Sign Out" }
}
```

Only truly shared UI strings. Module-specific strings stay in module files.

## ICU Message Syntax

### Interpolation

```json
{ "greeting": "Hello, {name}!", "itemCount": "{count} items" }
```

### Pluralization

```json
{ "followers": "{count, plural, =0 {No followers} =1 {One follower} other {# followers}}" }
```

### Select (Gender/Conditional)

```json
{ "greeting": "{gender, select, male {Welcome, sir} female {Welcome, ma'am} other {Welcome}}" }
```

### Rich Text

```json
{ "tos": "I accept the <link>terms of service</link>" }
```

```tsx
t.rich('tos', {
  link: (chunks) => <a href="/terms" className="underline">{chunks}</a>,
});
```

## Usage in Client Components

```tsx
// src/modules/auth/ui/login-form.tsx
'use client';
import { useTranslations } from 'next-intl';

export function LoginForm() {
  const te = useTranslations('common.errors');
  const t = useTranslations('auth.LoginPage.LoginForm');
  // ...
  return <input placeholder={t('emailPlaceholder')} />;
}
```

## Default Language: JSON as Source of Truth

English (`en`) JSON files are the **source of truth** for keys. Other locales mirror the key structure. Missing key → fallback to English automatically.

```
messages/en/auth.json → add new key (dev writes)
messages/fr/auth.json → copy structure, translate values (translator writes)
```

**Dev never touches non-English files.** Translation is a separate concern.

## i18n Rules

| Rule | Why |
|------|-----|
| Per-module JSON files (`auth.json`, `dashboard.json`) | Dev opens one file per module — no scrolling |
| Keys mirror component tree | Missing key → immediate file + path to fix |
| `localePrefix: 'as-needed'` | Clean URLs. Default locale has no prefix. |
| `proxy.ts` handles locale detection | Path → cookie → `accept-language` header → fallback |
| `NextIntlClientProvider` in root layout | Client components read translations from context |
| `useTranslations('namespace.Page.Component')` | Scoped to component. No accidental reuse. |
| English JSON = source of truth | Dev writes English. Translators fill other locales. |
| Non-English files never edited by dev | Keys auto-fallback to English. |
| `next-intl/plugin` wraps `next.config.ts` | Required for build-time i18n config injection |
| `useTranslations` stays out of DI | Reads React context, not a service — DI adds indirection |

---

# Naming Conventions

| Thing | Convention | Example |
|-------|-----------|---------|
| Module directory | camelCase | `auth`, `userManagement` |
| Zod schema file | `${name}.schema.ts` | `login.schema.ts` |
| Form schema object | camelCase + `FormSchema` | `registerFormSchema` |
| Request schema object | camelCase + `Schema` | `loginSchema`, `registerSchema` |
| Form values type | PascalCase + `FormValues` | `RegisterFormValues` |
| Request type | PascalCase + `Request` | `LoginRequest`, `RegisterRequest` |
| Response type file | `responses.ts` | `responses.ts` |
| Response interface | PascalCase + `Response` or `Dto` | `LoginResponse`, `UserDto` |
| Error type file | `errors.ts` | `errors.ts` |
| Error interface | PascalCase + `Error` | `ApiError` |
| Service file | `${name}.service.ts` | `auth.service.ts` |
| Repository file | `${name}.repository.ts` | `auth.repository.ts` |
| Repository class | PascalCase + `Repository` | `AuthRepository` |
| DI tokens file | `tokens.ts` | `tokens.ts` |
| DI container file | `container.ts` | `container.ts` |
| Store file | `${name}.store.ts` | `auth.store.ts` |
| Store hook | `useXxxStore` | `useAuthStore` |
| Query hook file | `use-${name}.ts` | `use-order-list.ts` |
| Query hook function | `useXxx` | `useOrderList` |
| UI component file | PascalCase `.tsx` | `LoginForm.tsx` |
| Page component | PascalCase `-page.tsx` | `login-page.tsx` |
| Barrel file | `index.ts` | `index.ts` |
| DI token | `module.Type` | `auth.AuthService` |
| Token constant object | `UPPER_CASE_TOKENS` | `AUTH_TOKENS` |

---

# What Does NOT Belong in a Module

- **Route registration** — lives in `app/`. Modules export page components, app composes them.
- **Root layout** — lives in `app/layout.tsx`. Modules export UI pieces layouts can use.
- **Global styles** — lives in `app/globals.css` or shared design tokens.
- **Config** — lives in `next.config.ts`, `postcss.config.mjs`, `src/env.ts`. Env vars validated by T3 Env.
- **Raw `fetch` calls** — repositories only. Even then, use the shared `apiClient`. No bare network calls.
- **Domain entities or value objects** — frontend needs request validation (Zod) + response types (interfaces). Backend owns the domain model.

---

# Quick Rules Reference

| Rule | Why |
|------|-----|
| `app/` is a thin routing shell | No business logic, no API calls. Compose from modules. |
| Module = business capability under `modules/` | Not a technical layer. Extract by capability. |
| `schemas/` depends on nothing | Pure Zod. No React, no Next.js, no HTTP. |
| `types/` depends on nothing | Pure TypeScript interfaces. No React, no Next.js, no HTTP. |
| `stores/` depends on `application/`, `types/` | Factory receives services. Holds client state. |
| `hooks/` depends on `application/`, `types/` | Wraps TanStack Query for components. |
| `application/` depends on `types/`, `schemas/`, `repositories/` | Thin orchestration. Injects `ILogger`. |
| `repositories/` = data fetching | Axios calls to external backend. One repo per resource. Injects `ILogger`. |
| `ui/` depends on `stores/`, `hooks/`, `schemas/` | Uses store actions/selectors, query hooks, FormField. |
| `di/` wires everything | Only place that knows concrete classes. |
| `index.ts` = public API barrel | Re-exports only what other modules may use. |
| All components are `'use client'` | Client-first. No server data fetching. |
| `apiClient` (axios) is the only HTTP surface | Repositories wrap it. No raw `axios`/`fetch` anywhere else. |
| `ILogger` is the only logging surface | Modules depend on interface. PinoLogger is DI-wired. |
| Child containers per module | Isolate registrations. Prevent circular resolution. |
| Token naming: `module.Type` | Grep-friendly. No magic strings. |
| Pages are one-liners | `export default ModulePage` — nothing more. |
| Never import another module's `repositories/` | Use barrel + DI tokens. |
| Zod validates ALL rules at once | No `.abortEarly`. User sees every error on first submit. |
| FormField shows ALL errors per field | Bullet list. Destructive border on input. |
| Zod validates at the form boundary | `ui/` validates before `stores/` — backend owns business rules. |
| `proxy.ts` handles locale routing | next-intl `createIntlProxy`. Replaces `middleware.ts`. |
| Security headers in `next.config.ts` | Set once. HSTS production-only. |
| Env vars via `@t3-oss/env-nextjs` | Typed. Validated at build + runtime. No raw `process.env`. |

---

# Anti-Patterns

| Anti-pattern | Correct |
|-------------|---------|
| Business logic in `app/page.tsx` | Page re-exports module's page component |
| `axios`/`fetch` directly in a component | Store → Service → Repository → `apiClient` |
| `axios`/`fetch` directly in a service | Inject the repository, call its methods |
| Module importing sibling's `repositories/` | Use barrel `index.ts` + DI token |
| Single global tsyringe container | Per-module child containers |
| `container.resolve()` in render without `useMemo` | Always use `useResolve` hook |
| `export *` in barrel | Intentional barrel — re-export only the public surface |
| `src/components/`, `src/services/`, `src/hooks/` at root | Move to appropriate module or `shared/` |
| DB imports in `repositories/` | Client-first — repositories call external API, not DB |
| `domain/` with entities, VOs, domain errors | Frontend needs Zod schemas (requests) + interfaces (responses) |
| Untyped form submissions | Validate with TanStack Form + Zod adapter |
| `IUser` or `TUser` prefixes on types | Plain interfaces: `UserDto`, `LoginResponse` |
| Strings in JSX | Use `useTranslations('module.Page.Component')` |
| Raw `process.env` | Import `env` from `@/env` |
| `console.log` in services/repos | Inject `ILogger`, call `this.log.info({...}, msg)` |
| Stores in `ui/stores/` | `stores/` at module root — own layer |
| Hooks in `ui/hooks/` | `hooks/` at module root — own adapter layer |

---

# Enforcement

When building Next.js 16 with modular client-first architecture:

1. Every new feature starts as a module — `src/modules/<name>/`.
2. Every module has `schemas/`, `types/`, `stores/`, `application/`, `repositories/`, `ui/`, `di/` — skip only if a layer is genuinely empty.
3. Every component is `'use client'`. No server-side data fetching.
4. Every page in `app/` is one line + metadata: `export const metadata = {...}; export default ModulePage;`.
5. Every API call goes through a repository — never raw `axios`/`fetch`.
6. Every Client Component resolves services via `useResolve(token)` — never `new Service()` directly.
7. Every DI registration uses Symbol tokens: `Symbol.for('module.Type')`.
8. Every repository is a plain class injecting `AxiosInstance` + `ILogger`.
9. Every service injects `ILogger` via `SHARED_TOKENS.Logger`, creates child with `{ module, layer }`.
10. Every log call passes structured object as first argument: `log.info({ userId }, 'msg')`.
11. Every form validates with Zod before calling the service — schema passed to TanStack Form's `validators.onSubmit`.
12. Every form field uses `FormField` component — renders ALL errors, not just first.
13. Zod validates ALL rules at once — no `.abortEarly`. User sees every failure on first submit.
14. Every type from the API is a plain interface with `Dto` or `Response` suffix — no entities, no value objects.
15. Every user-facing string uses `useTranslations('module.Page.Component')` — keys mirror component tree.
16. Every public page exports canonical URL via `alternates.canonical`.
17. Every env variable is validated by T3 Env — no raw `process.env`.
18. Security headers are set once in `next.config.ts` — HSTS production-only.
19. When in doubt, apply the extraction test: "Can I move this module to a separate package without rewriting its business logic?"
