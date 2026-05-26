---
name: nextjs-modular-architecture
description: Enforce modular architecture in Next.js 16 — client-first, feature modules, per-module tsyringe DI, thin app/ routing shell, Zod validation schemas, API clients as infrastructure.
version: 2.1.0
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
    shared/                     # Cross-cutting — axios instance, types, utils
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

Every module follows ports & adapters. Three layers, strict dependency direction.

```
src/modules/auth/
  schemas/                      # Zod validation schemas — request shapes
  types/                        # API response types, error types (plain interfaces)
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
ui → application → repositories

- `schemas/` depends on **nothing** — pure Zod, no React, no Next.js, no HTTP.
- `types/` depends on **nothing** — pure TypeScript interfaces, no React, no Next.js, no HTTP.
- `application/` depends on `types/` and `schemas/` — uses response types, inferred request types.
- `repositories/` depends on `types/` — data fetching, axios calls, returns response types.
- `ui/` depends on `application/` and `schemas/` — validates form input with Zod, calls services via `useResolve`.
- `di/` depends on everything — wires the module.
- `index.ts` re-exports only what other modules may use.

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
import { injectable } from 'tsyringe';
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
  constructor(
    private readonly authRepo: AuthRepository,
  ) {}

  async login(input: LoginRequest): Promise<UserDto> {
    const { user } = await this.authRepo.login(input);
    return user;
  }

  async register(input: RegisterRequest): Promise<UserDto> {
    const { user } = await this.authRepo.register(input);
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
import type { LoginResponse, RegisterResponse, TokenPair } from '../types/responses';
import type { LoginRequest } from '../schemas/login.schema';
import type { RegisterRequest } from '../schemas/register.schema';

@injectable()
export class AuthRepository {
  constructor(
    @inject(SHARED_TOKENS.ApiClient) private readonly apiClient: AxiosInstance,
  ) {}

  async login(req: LoginRequest): Promise<LoginResponse> {
    const { data } = await this.apiClient.post<LoginResponse>('/auth/login', req);
    return data;
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
- Return `response.data` — strip axios envelope at this layer.
- Throw on non-2xx — axios does this by default. `AxiosError` carries the backend error body.
- No `Result<T>`, no `HttpClient`. Direct. Simple.

### `ui/` — React Components

All `'use client'`. Pages, forms, widgets. Depends on `application/` — resolves services via `useResolve`. Forms use TanStack Form with Zod adapter — schema validation, typed fields, form state managed by TanStack. Never calls `axios`/`fetch` directly. Never imports from `repositories/`.

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
import { useState } from 'react';

export function LoginForm() {
  const authService = useResolve<AuthService>(AUTH_TOKENS.AuthService);
  const [error, setError] = useState<string | null>(null);

  const form = useForm({
    defaultValues: { email: '', password: '' },
    validatorAdapter: zodValidator(),
    validators: {
      onSubmit: loginSchema,
    },
    onSubmit: async ({ value }) => {
      setError(null);
      try {
        await authService.login(value);
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Login failed');
      }
    },
  });

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        e.stopPropagation();
        form.handleSubmit();
      }}
    >
      {error && <p role="alert">{error}</p>}

      <form.Field
        name="email"
        children={(field) => (
          <>
            <input
              type="email"
              name={field.name}
              value={field.state.value}
              onBlur={field.handleBlur}
              onChange={(e) => field.handleChange(e.target.value)}
              placeholder="Email"
            />
            {field.state.meta.errors.length > 0 && (
              <em>{field.state.meta.errors[0]}</em>
            )}
          </>
        )}
      />

      <form.Field
        name="password"
        children={(field) => (
          <>
            <input
              type="password"
              name={field.name}
              value={field.state.value}
              onBlur={field.handleBlur}
              onChange={(e) => field.handleChange(e.target.value)}
              placeholder="Password"
            />
            {field.state.meta.errors.length > 0 && (
              <em>{field.state.meta.errors[0]}</em>
            )}
          </>
        )}
      />

      <form.Subscribe
        selector={(state) => [state.canSubmit, state.isSubmitting]}
        children={([canSubmit, isSubmitting]) => (
          <button type="submit" disabled={!canSubmit}>
            {isSubmitting ? 'Signing in...' : 'Sign In'}
          </button>
        )}
      />
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
import { useState } from 'react';

export function RegisterForm() {
  const authService = useResolve<AuthService>(AUTH_TOKENS.AuthService);
  const [error, setError] = useState<string | null>(null);

  const form = useForm({
    defaultValues: { email: '', password: '', name: '' },
    validatorAdapter: zodValidator(),
    validators: {
      onSubmit: registerSchema,
    },
    onSubmit: async ({ value }) => {
      setError(null);
      try {
        await authService.register(value);
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Registration failed');
      }
    },
  });

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        e.stopPropagation();
        form.handleSubmit();
      }}
    >
      {error && <p role="alert">{error}</p>}

      <form.Field name="name" children={...} />
      <form.Field name="email" children={...} />
      <form.Field name="password" children={...} />

      <form.Subscribe
        selector={(state) => [state.canSubmit, state.isSubmitting]}
        children={([canSubmit, isSubmitting]) => (
          <button type="submit" disabled={!canSubmit}>
            {isSubmitting ? 'Creating account...' : 'Create Account'}
          </button>
        )}
      />
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

export const AUTH_TOKENS = {
  AuthService: Symbol.for('auth.AuthService') as unknown as InjectionToken<AuthService>,
  AuthRepository: Symbol.for('auth.AuthRepository') as unknown as InjectionToken<AuthRepository>,
} as const;
```

```ts
// src/modules/auth/di/container.ts
import { container as rootContainer } from '@/app/_lib/di/container';
import { AUTH_TOKENS } from './tokens';
import { AuthServiceImpl } from '../application/auth.service';
import { AuthRepository } from '../repositories/auth.repository';

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

## Type Visibility Summary

| Layer | Visibility | Imported By |
|-------|-----------|-------------|
| `schemas/` Zod schemas | `export` | `ui/` (form validation), `application/` (service inputs), other modules' `di/` |
| `schemas/` inferred types | `export` | `repositories/` (request types), `ui/`, `application/` |
| `types/` response interfaces | `export` | `application/`, `repositories/`, `ui/` |
| `types/` error shapes | `export` | `application/`, `ui/` |
| `application/` service interface | `export` | `ui/`, other modules' `di/` |
| `application/` service impl | `export` but only `di/` imports it | `di/container.ts` |
| `repositories/` classes | `export` | `di/` (wires), `application/` (imports class type), other modules' `di/` |
| `ui/` components | `export` | `app/` routes, other modules' layouts |
| `di/tokens.ts` | `export` | `ui/` (useResolve), other modules' `di/` |
| `di/container.ts` init fn | `export` | root `DIProvider` |

---

# The `app/` Directory — Thin Routing Shell

## Principle

The `app/` directory **only** contains routing files (`page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`) and shared infrastructure (`_lib/`). No business logic. No API calls. No component implementations.

Every page is a thin wrapper:

```tsx
// app/(auth)/login/page.tsx
'use client';

import { LoginPage } from '@/modules/auth';
export default LoginPage;
```

That's it. One import, one export. Pages are the thinnest possible shell.

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
import { initializeAuthModule } from '@/modules/auth';
import { initializeDashboardModule } from '@/modules/dashboard';
import { initializeBillingModule } from '@/modules/billing';

export function initializeAllModules(): void {
  initializeAuthModule();         // no deps
  initializeDashboardModule();    // depends on auth
  initializeBillingModule();      // depends on auth, dashboard
}
```

## Root Layout — Wire Providers

```tsx
// app/layout.tsx
import { QueryProvider } from './_lib/query/provider';
import { DIProvider } from './_lib/di/provider';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <QueryProvider>
          <DIProvider>
            {children}
          </DIProvider>
        </QueryProvider>
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

## Axios Instance

One pre-configured axios instance. Every API client imports it. Handles base URL, credentials, auth headers via interceptor.

```bash
bun add axios
```

```ts
// src/modules/shared/di/tokens.ts
import type { InjectionToken } from 'tsyringe';
import type { AxiosInstance } from 'axios';

export const SHARED_TOKENS = {
  ApiClient: Symbol.for('shared.ApiClient') as unknown as InjectionToken<AxiosInstance>,
} as const;
```

```ts
// src/modules/shared/di/container.ts
import { container as rootContainer } from '@/app/_lib/di/container';
import axios from 'axios';
import { SHARED_TOKENS } from './tokens';

export function initializeSharedModule(): void {
  const apiClient = axios.create({
    baseURL: process.env.NEXT_PUBLIC_API_BASE_URL ?? '/api',
    withCredentials: true,
    headers: {
      'Content-Type': 'application/json',
    },
  });

  apiClient.interceptors.request.use((config) => {
    const token = getAuthToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  });

  apiClient.interceptors.response.use(
    (res) => res,
    async (error) => {
      if (error.response?.status === 401 && !error.config._retry) {
        error.config._retry = true;
        // Refresh token logic — call /auth/refresh, update stored token, retry
      }
      return Promise.reject(error);
    },
  );

  rootContainer.register(SHARED_TOKENS.ApiClient, {
    useValue: apiClient,
  });
}

function getAuthToken(): string | null {
  // Read from cookie or auth store
  return null;
}
```

```ts
// src/modules/shared/index.ts
export { SHARED_TOKENS } from './di/tokens';
export { initializeSharedModule } from './di/container';
```

**What the singleton `apiClient` provides**:
- Single axios instance — every API client injects the same instance via DI
- `baseURL` — all paths are relative, no URL construction in modules
- `withCredentials` — cookies sent cross-origin
- Request interceptor — injects auth token, one place to change token source
- Response interceptor — 401 handling, token refresh, centralized
- API clients inject `SHARED_TOKENS.ApiClient` as `AxiosInstance` — no custom wrapper type needed

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
      // Backend returned an error response — shape is ApiError
      const apiError: ApiError = {
        code: error.response.data?.code ?? 'http.error',
        message: error.response.data?.message ?? error.message,
        field: error.response.data?.field,
      };
      return Promise.reject(apiError);
    }
    // Network error — no response
    return Promise.reject({
      code: 'network.error',
      message: error.message,
    } satisfies ApiError);
  },
);
```

Components catch `ApiError` — always the same shape, whether backend error or network failure.

## What Belongs in Shared

- `apiClient` — pre-configured axios instance
- `ApiError` type
- `useResolve` hook
- DI infrastructure (root container)
- Utility functions (`cn`, `formatDate`, etc.)
- Reusable hooks (`useDebounce`, `useMediaQuery`)

## What Does NOT Belong in Shared

- Any business schema or response type
- Any module-specific service
- Any module-specific component
- Any module-specific API client
- Any business rule

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
'use client';

import { DashboardPage } from '@/modules/dashboard';
export default DashboardPage;
```

With layout:

```tsx
// app/(dashboard)/layout.tsx
'use client';

import { AuthGuard } from '@/modules/auth';

export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <AuthGuard>
      <DashboardShell>
        {children}
      </DashboardShell>
    </AuthGuard>
  );
}
```

With loading:

```tsx
// app/(dashboard)/loading.tsx
'use client';

export default function DashboardLoading() {
  return <DashboardSkeleton />;
}
```

**Rule**: Pages are one-liners. Layouts compose module guards and shells. Loading states are module skeletons. Zero business logic at the route level.

---

# Data Flow

```
User action (click, submit)
  → ui/ form validates input with Zod schema
    → ui/ calls service via useResolve(token) with typed request
      → application/ service calls repository
        → repositories/ calls apiClient (axios)
          → external API
        ← response data
      ← service unpacks DTO, returns what UI needs
    ← component updates state (useState / useReducer / Zustand)
```

Every step is a typed contract. No layer crosses another's boundary without conversion.

```
ui/                         knows: service interface, Zod schemas, response types, React state
application/                knows: response types, repository type, schema-inferred request types
repositories/               knows: axios apiClient, response types, request types
schemas/                    knows: nothing but Zod
types/                      knows: nothing but its own interfaces
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

Stores live in the module's `ui/` directory. They're UI state — they belong with the components that consume them.

```
src/modules/auth/
  ui/
    stores/
      auth.store.ts          # Auth state — user, tokens, login status
```

For cross-module state (e.g., current user), export the store via the module's barrel:

```ts
// src/modules/auth/index.ts
export { useAuthStore } from './ui/stores/auth.store';
```

Shared/generic stores live in `src/modules/shared/ui/stores/`:

```
src/modules/shared/
  ui/
    stores/
      notification.store.ts  # Toast notifications — used by any module
      theme.store.ts          # Dark/light mode
```

## Store Naming

- **Store file**: `${name}.store.ts` — `auth.store.ts`, `dashboard.store.ts`
- **Store hook**: `useXxxStore` — `useAuthStore`, `useDashboardStore`
- **State type**: `XxxState` — `AuthState`, `DashboardState`
- **Actions**: verb or `set` prefix — `login`, `logout`, `setUser`, `addItem`, `removeItem`
- **Selector hooks** (optional): `useXxx` on exported selectors — `useCurrentUser`, `useIsAuthenticated`

## Store Factory Pattern — Injecting Services

Stores that need services (API calls, business logic) receive them via a factory function. The store itself is created at DI initialization time, then registered as a singleton in the child container.

```ts
// src/modules/auth/ui/stores/auth.store.ts
import { create } from 'zustand';
import type { UserDto } from '../../types/responses';
import type { AuthService } from '../../application/auth.service';

interface AuthState {
  user: UserDto | null;
  isLoading: boolean;
  error: string | null;

  // Actions
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

**Why factory and not `useAuthStore.getState()` inside the service**: The service doesn't know about stores. The store knows about the service. Dependency direction: `ui/store → application/service`. Never the reverse. The store calls `authService.login()` — the service never calls `useAuthStore.getState().setUser()`.

## Register Store in DI Container

The store factory is called during module initialization. The resulting hook type is registered so components can resolve it.

```ts
// src/modules/auth/di/tokens.ts
import type { InjectionToken } from 'tsyringe';
import type { AuthStore } from '../ui/stores/auth.store';

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
import { createAuthStore } from '../ui/stores/auth.store';

export function initializeAuthModule(): void {
  const childContainer = rootContainer.createChildContainer();

  // Repositories
  childContainer.register(AUTH_TOKENS.AuthRepository, {
    useClass: AuthRepository,
  });

  // Application
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

## Using Stores in Components

Components resolve the store hook via `useResolve`, then use it like any Zustand store. For forms, combine Zustand store actions with TanStack Form:

```tsx
// src/modules/auth/ui/login-form.tsx
'use client';

import { useForm } from '@tanstack/react-form';
import { zodValidator } from '@tanstack/zod-form-adapter';
import { useResolve } from '@/app/_lib/di/use-resolve';
import { AUTH_TOKENS } from '../di/tokens';
import type { AuthStore } from './stores/auth.store';
import { loginSchema } from '../schemas/login.schema';

export function LoginForm() {
  const useAuthStore = useResolve<AuthStore>(AUTH_TOKENS.AuthStore);
  const { login, isLoading, error, clearError } = useAuthStore();

  const form = useForm({
    defaultValues: { email: '', password: '' },
    validatorAdapter: zodValidator(),
    validators: {
      onSubmit: loginSchema,
    },
    onSubmit: async ({ value }) => {
      clearError();
      await login(value.email, value.password);
    },
  });

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        e.stopPropagation();
        form.handleSubmit();
      }}
    >
      {error && <p role="alert">{error}</p>}

      <form.Field name="email" children={...} />
      <form.Field name="password" children={...} />

      <form.Subscribe
        selector={(state) => [state.canSubmit, state.isSubmitting]}
        children={([canSubmit, isSubmitting]) => (
          <button type="submit" disabled={!canSubmit || isLoading}>
            {isSubmitting || isLoading ? 'Signing in...' : 'Sign In'}
          </button>
        )}
      />
    </form>
  );
}
```

## Selectors — Fine-Grained Subscriptions

Use selectors to prevent unnecessary re-renders. A component subscribing to `useAuthStore()` re-renders on ANY state change. Selectors narrow the subscription:

```tsx
// src/modules/auth/ui/user-avatar.tsx
'use client';

import { useResolve } from '@/app/_lib/di/use-resolve';
import { AUTH_TOKENS } from '../di/tokens';
import type { AuthStore } from './stores/auth.store';

export function UserAvatar() {
  const useAuthStore = useResolve<AuthStore>(AUTH_TOKENS.AuthStore);
  const user = useAuthStore((s) => s.user);   // only re-renders when user changes
  const logout = useAuthStore((s) => s.logout); // action — stable reference

  if (!user) return null;

  return (
    <div>
      <span>{user.email}</span>
      <button onClick={logout}>Sign Out</button>
    </div>
  );
}
```

```tsx
// Selector for derived/computed values
export function AuthGuard({ children }: { children: React.ReactNode }) {
  const useAuthStore = useResolve<AuthStore>(AUTH_TOKENS.AuthStore);
  const isAuthenticated = useAuthStore((s) => s.user !== null);
  const isLoading = useAuthStore((s) => s.isLoading);

  if (isLoading) return <Spinner />;
  if (!isAuthenticated) return <Navigate to="/login" />;
  return <>{children}</>;
}
```

## Store + Service Together

Some components need both — actions from the store AND direct service calls:

```tsx
// Component that reads state from store, calls service directly for one-shot action
export function DashboardRefresh() {
  const useDashboardStore = useResolve<DashboardStore>(DASHBOARD_TOKENS.DashboardStore);
  const dashboardService = useResolve<DashboardService>(DASHBOARD_TOKENS.DashboardService);
  const stats = useDashboardStore((s) => s.stats);
  const setStats = useDashboardStore((s) => s.setStats);

  const handleRefresh = async () => {
    const fresh = await dashboardService.refreshStats(); // service call, not store action
    setStats(fresh);
  };

  return (
    <div>
      <p>Orders: {stats.orderCount}</p>
      <button onClick={handleRefresh}>Refresh</button>
    </div>
  );
}
```

**Rule**: If the action is "mutate store state based on API result" and it's called from one place — put it in the component. If it's called from multiple components — put it in the store action.

## When NOT to Put Logic in the Store

| Don't | Do Instead |
|-------|-----------|
| Business validation in store actions | Service validates, store calls service |
| API URL construction in store | Repository / axios handles URLs |
| Token management in store | Axios interceptor in shared handles tokens |
| Complex orchestration (call A, then B, then C) | Service orchestrates, store calls one service method |
| Cross-module data aggregation | Service depends on other service via DI, store calls this service |

Store actions should be thin — call service, set state. The service owns "how." The store owns "what just happened."

## Store Persistence — `zustand/middleware`

For persistent state (survives page reload), use `persist` middleware. The store factory stays the same:

```ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import type { AuthService } from '../../application/auth.service';

export function createAuthStore(authService: AuthService) {
  return create<AuthState>()(
    persist(
      (set) => ({
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
      }),
      {
        name: 'auth-store',        // localStorage key
        partialize: (state) => ({  // only persist these fields
          user: state.user,
        }),
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
| Module store | `modules/<name>/ui/stores/<name>.store.ts` | `modules/<name>/di/container.ts` | Module's `ui/` components |
| Shared store | `modules/shared/ui/stores/<name>.store.ts` | `modules/shared/di/container.ts` | Any module's `ui/` components |
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

TanStack Query needs no per-module DI registration. `QueryClient` is created once in `QueryProvider`. Hooks call `useQueryClient()` directly — it's a React context singleton.

## Query Keys — Per-Module Key Factory

Each module defines its query keys as a const object. Keys follow `[module, entity, ...params]` convention — same structure used by React Query for cache invalidation.

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

## Query Hooks — In `ui/hooks/`

Each module exposes `useXxx` hooks that wrap TanStack Query. Components never call `useQuery`/`useMutation` with raw API calls — they use the module's hooks.

```
src/modules/orders/
  ui/
    hooks/
      use-order-list.ts       # useOrderList — paginated list
      use-order-detail.ts     # useOrderDetail — single order
      use-create-order.ts     # useCreateOrder — mutation
```

### Typed Pagination — Shared Types

Mirrors the Go `pagination.Params` / `pagination.Metadata` split. Transport-agnostic input, transport-agnostic output.

```ts
// src/modules/shared/types/pagination.ts

/** Transport-agnostic pagination input. Extracted from query params / UI state. */
export interface PaginationParams {
  page: number;
  pageSize: number;
}

/** Creates validated params with defaults. Safety caps enforced. */
export function createPaginationParams(page?: number, pageSize?: number): PaginationParams {
  return {
    page: Math.max(1, Math.min(page ?? 1, 50)),
    pageSize: Math.max(1, Math.min(pageSize ?? 20, 100)),
  };
}

/** Transport-agnostic pagination output. Built from API response metadata. */
export interface PaginationMetadata {
  page: number;
  pageSize: number;
  totalItems: number;
  totalPages: number;
}

/** API response envelope for paginated lists. */
export interface PaginatedResponse<T> {
  data: T[];
  pagination: PaginationMetadata;
}
```

### List Hook — `useInfiniteQuery` for Cursor/Offset Pagination

Use `useInfiniteQuery` for paginated lists. It handles page accumulation, `hasNextPage`, and `fetchNextPage` natively. The API client returns `PaginatedResponse<T>` and the hook flattens it for the UI.

```ts
// src/modules/orders/ui/hooks/use-order-list.ts
import { useInfiniteQuery } from '@tanstack/react-query';
import { useResolve } from '@/app/_lib/di/use-resolve';
import { ORDER_TOKENS } from '../../di/tokens';
import type { OrderService } from '../../application/order.service';
import type { OrderFilters } from '../../types/filters';
import type { OrderDto } from '../../types/responses';
import type { PaginationParams } from '@/modules/shared/types/pagination';
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

### Service — Returns `PaginatedResponse<T>`

```ts
// src/modules/orders/application/order.service.ts
import type { PaginatedResponse, PaginationParams } from '@/modules/shared/types/pagination';
import type { OrderDto } from '../types/responses';
import type { OrderFilters } from '../types/filters';
import type { OrderApiClient } from './ports';

export interface OrderService {
  list(filters: OrderFilters, params: PaginationParams): Promise<PaginatedResponse<OrderDto>>;
  getById(id: string): Promise<OrderDto>;
}

@injectable()
export class OrderServiceImpl implements OrderService {
  constructor(private readonly orderApi: OrderApiClient) {}

  async list(filters: OrderFilters, params: PaginationParams): Promise<PaginatedResponse<OrderDto>> {
    const result = await this.orderApi.list({ ...filters, ...params });

    if (result.kind === 'error') {
      throw new Error(result.message);
    }

    return this.orderRepo.list({ ...filters, ...params }); // PaginatedResponse<OrderDto>
  }

  async getById(id: string): Promise<OrderDto> {
    return this.orderRepo.getById(id);
  }
}
```

### Component — Paginated List UI

```tsx
// src/modules/orders/ui/order-list-page.tsx
'use client';

import { useOrderList } from './hooks/use-order-list';
import { useState } from 'react';
import type { OrderFilters } from '../types/filters';

export function OrderListPage() {
  const [filters, setFilters] = useState<OrderFilters>({ status: 'all' });
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
    isError,
    refetch,
  } = useOrderList({ filters, pageSize: 20 });

  if (isLoading) return <OrderListSkeleton />;
  if (isError) return <ErrorFallback onRetry={refetch} />;

  const orders = data?.pages.flatMap((page) => page.data) ?? [];

  return (
    <div>
      <OrderFiltersBar filters={filters} onChange={setFilters} />

      {orders.length === 0 ? (
        <EmptyState message="No orders found" />
      ) : (
        <div>
          {orders.map((order) => (
            <OrderCard key={order.id} order={order} />
          ))}

          <button
            onClick={() => fetchNextPage()}
            disabled={!hasNextPage || isFetchingNextPage}
          >
            {isFetchingNextPage ? 'Loading...' : hasNextPage ? 'Load More' : 'No more orders'}
          </button>
        </div>
      )}
    </div>
  );
}
```

### Mutation Hook

```ts
// src/modules/orders/ui/hooks/use-create-order.ts
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
| Query hooks live in `ui/hooks/` | UI concern — components consume them directly. |
| One query hook per query type | `useOrderList`, `useOrderDetail` — not one generic `useQuery` wrapper. |
| Query keys use `[module, entity, ...params]` | Cache invalidation by prefix: `['orders', 'list']` invalidates all list queries. |
| `useInfiniteQuery` for paginated lists | `fetchNextPage` + `hasNextPage` built in. No manual page tracking. |
| `PaginationParams` / `PaginationMetadata` are transport-agnostic | Shared types. No HTTP details (no `_links`, no `next_cursor`). |
| API client returns `PaginatedResponse<T>` | `{ data: T[], pagination: PaginationMetadata }` — uniform shape. |
| `staleTime` per query type | Lists: 30s default. Detail views: 60s. Invalidate on mutations. |
| Mutation `onSuccess` invalidates affected query keys | Create/update/delete invalidates the list. Detail update invalidates detail. |
| Service layer unpacks response DTOs | Repository returns raw response, service picks what UI needs. |
| Query hooks don't call `useResolve` inside `queryFn` | `useResolve` is called once at component level, values captured in closure. |

---

# Tailwind CSS — Styling

## Principle

Tailwind is the project's styling layer. No CSS modules, no styled-components, no inline styles. Every component uses Tailwind utility classes. Shared design tokens live in `tailwind.config.ts` theme extensions — per-module custom colors/spacing/typography don't belong there.

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

**Tailwind v4 key differences from v3**:
- No `tailwind.config.ts` — everything is CSS `@theme`
- No `content` config — Tailwind auto-detects source files
- `@import 'tailwindcss'` replaces `@tailwind base/components/utilities`
- `@theme` block replaces `theme.extend` in JS config
- CSS variables for design tokens: `--color-*`, `--font-*`, `--spacing-*`
- Legacy JS config still supported via `@config` directive, but prefer native CSS

## Utility-First — Components

Every component uses Tailwind classes directly in JSX. No abstraction, no `@apply` in component layers.

```tsx
// src/modules/auth/ui/login-form.tsx
export function LoginForm() {
  // ... form logic ...

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        e.stopPropagation();
        form.handleSubmit();
      }}
      className="flex flex-col gap-4 w-full max-w-md mx-auto p-6 bg-white rounded-lg shadow-sm border border-gray-200"
    >
      {error && (
        <p role="alert" className="text-sm text-red-600 bg-red-50 px-3 py-2 rounded">
          {error}
        </p>
      )}

      <form.Field
        name="email"
        children={(field) => (
          <div className="flex flex-col gap-1">
            <label htmlFor="email" className="text-sm font-medium text-gray-700">
              Email
            </label>
            <input
              id="email"
              type="email"
              name={field.name}
              value={field.state.value}
              onBlur={field.handleBlur}
              onChange={(e) => field.handleChange(e.target.value)}
              className="w-full px-3 py-2 border border-gray-300 rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            />
            {field.state.meta.errors.length > 0 && (
              <em className="text-xs text-red-500">{field.state.meta.errors[0]}</em>
            )}
          </div>
        )}
      />

      /* ... password field ... */

      <form.Subscribe
        selector={(state) => [state.canSubmit, state.isSubmitting]}
        children={([canSubmit, isSubmitting]) => (
          <button
            type="submit"
            disabled={!canSubmit}
            className="w-full py-2 px-4 bg-blue-600 text-white font-medium rounded-md hover:bg-blue-700 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
          >
            {isSubmitting ? 'Signing in...' : 'Sign In'}
          </button>
        )}
      />
    </form>
  );
}
```

**Why no CSS modules / `styles.ts`**: One styling system = one mental model. Tailwind classes are co-located with markup — no file switching, no naming conflicts, dead code eliminated at build time.

## Theme Extension — `globals.css`

Theme is configured directly in CSS. Only project-wide design tokens. Module-specific colors don't belong here — use Tailwind's built-in palette or raw hex values in `className`.

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

Usage: `bg-brand-500`, `text-brand-700`, `font-sans` — Tailwind generates these from the `@theme` keys automatically.

## Layout & Spacing Patterns

| Pattern | Classes |
|--------|---------|
| Page container | `max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8` |
| Form card | `w-full max-w-md mx-auto p-6 bg-white rounded-lg shadow-sm border` |
| Two-column form | `grid grid-cols-1 sm:grid-cols-2 gap-4` |
| List → empty state | List has `divide-y`, empty shows `text-center py-12 text-gray-500` |
| Loading skeleton | `animate-pulse bg-gray-200 rounded` on skeleton blocks |
| Error banner | `bg-red-50 border border-red-200 text-red-700 px-4 py-3 rounded` |

## Tailwind Rules

| Rule | Why |
|------|-----|
| Utility classes in JSX only | No `@apply`, no CSS modules, no `styles.ts`. One system. |
| `className` order: layout → sizing → spacing → visual → state | Consistent scanning. `flex flex-col gap-4 w-full p-6 bg-white rounded shadow` |
| Theme in CSS `@theme` block — no JS config | Tailwind v4 is CSS-first. `@theme { --color-brand-500: #6366f1 }` in `globals.css`. |
| Design tokens in `@theme` are project-wide only | Module-specific colors use raw hex or built-in palette. |
| No arbitrary values in production | `w-[327px]` is prototyping only. Use design tokens or standard spacing scale. |
| Responsive: mobile-first | `sm:`, `md:`, `lg:` prefixes. Default classes = mobile. |
| `clsx` / `cn` for conditional classes | `cn('base-class', isActive && 'active-class')` — `clsx` is a dependency. |
| Focus states on every interactive element | `focus:outline-none focus:ring-2 focus:ring-blue-500` — accessibility baseline. |
| `disabled:` state on every button/input | `disabled:opacity-50 disabled:cursor-not-allowed` — visual feedback always. |
| Tailwind v4: no `content` globs | Auto-detects source. No `tailwind.config.ts` `content` array to maintain. |
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

# Naming Conventions

Follows the `typescript-react-naming-conventions` skill. Module-specific additions:

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
| Ports file | `ports.ts` | `ports.ts` |
| Repository file | `${name}.repository.ts` | `auth.repository.ts` |
| Repository class | PascalCase + `Repository` | `AuthRepository` |
| DI tokens file | `tokens.ts` | `tokens.ts` |
| DI container file | `container.ts` | `container.ts` |
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
- **Config** — lives in `next.config.ts`, `postcss.config.mjs`, `.env`. Modules read typed config from DI.
- **Raw `fetch` calls** — repositories only. Even then, use the shared `apiClient`. No bare network calls anywhere else.
- **Domain entities or value objects** — frontend only needs request validation (Zod) and response types (plain interfaces). The backend owns the domain model.

---

# Quick Rules Reference

| Rule | Why |
|------|-----|
| `app/` is a thin routing shell | No business logic, no API calls. Compose from modules. |
| Module = business capability under `modules/` | Not a technical layer. Extract by capability. |
| `schemas/` depends on nothing | Pure Zod. No React, no Next.js, no HTTP. |
| `types/` depends on nothing | Pure TypeScript interfaces. No React, no Next.js, no HTTP. |
| `application/` depends only on `types/` and `schemas/` | Thin orchestration. Receives repositories via constructor DI. |
| `repositories/` = data fetching | Axios calls to external backend. One repo per resource. |
| `ui/` depends on `application/` and `schemas/` | Validates with Zod, resolves services via `useResolve`. |
| `di/` wires everything | Only place that knows concrete classes. |
| `index.ts` = public API barrel | Re-exports only what other modules may use. |
| All components are `'use client'` | Client-first. No server data fetching. |
| `apiClient` (axios) is the only HTTP surface | Repositories wrap it. No raw `axios`/`fetch` anywhere else. |
| Child containers per module | Isolate registrations. Prevent circular resolution. |
| Token naming: `module.Type` | Grep-friendly. No magic strings. |
| Pages are one-liners | `export default ModulePage` — nothing more. |
| Never import another module's `repositories/` | Bypasses contracts. Use barrel + DI tokens. |
| Zod validates at the form boundary | `ui/` validates before `application/` — backend owns business rules. |

---

# Anti-Patterns

| Anti-pattern | Correct |
|-------------|---------|
| Business logic in `app/page.tsx` | Page re-exports module's page component |
| `axios`/`fetch` directly in a component | Service → Repository → `apiClient` |
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

---

# Enforcement

When building Next.js 16 with modular client-first architecture:

1. Every new feature starts as a module — `src/modules/<name>/`.
2. Every module has `schemas/`, `types/`, `application/`, `repositories/`, `ui/`, `di/` — skip only if a layer is genuinely empty.
3. Every component is `'use client'`. No server-side data fetching.
4. Every page in `app/` is one line: `export default ModulePage;`.
5. Every API call goes through a repository — never raw `axios`/`fetch`.
6. Every Client Component resolves services via `useResolve(token)` — never `new Service()` directly.
7. Every DI registration uses Symbol tokens: `Symbol.for('module.Type')`.
8. Every repository is a plain class injecting `AxiosInstance` — no custom wrapper, no `Result<T>`.
9. Every form validates with Zod before calling the service — `schema.safeParse()` at the UI boundary.
10. Every type from the API is a plain interface with `Dto` or `Response` suffix — no entities, no value objects, no domain classes.
11. When in doubt, apply the extraction test: "Can I move this module to a separate package without rewriting its business logic?"
