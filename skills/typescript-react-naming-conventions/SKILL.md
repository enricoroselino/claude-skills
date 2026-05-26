---
name: typescript-react-naming-conventions
description: Enforce idiomatic TypeScript + React naming conventions across variables, functions, types, components, files, hooks, events, props, and tests.
version: 1.0.0
author: User
---

# TypeScript + React Naming Conventions

## Purpose

Enforce idiomatic naming conventions for TypeScript and React codebases. Covers every naming surface following:

- [Airbnb React/JSX Style Guide](https://airbnb.io/javascript/react/)
- [TypeScript Deep Dive — Naming](https://basarat.gitbook.io/typescript/styleguide)
- React documentation and community precedent

---

# File Names

## Rules

- **PascalCase** for component files: `UserProfile.tsx`, `SearchBar.tsx`.
- **camelCase** for utilities, hooks, services, types: `useAuth.ts`, `apiClient.ts`, `formatDate.ts`.
- **kebab-case** for config/setup files: `tailwind.config.ts`, `vite.config.ts`.
- **index.ts / index.tsx** only for barrel re-exports — never put implementation inside.
- Test files: co-locate with `*.test.ts(x)` or `*.spec.ts(x)`, or mirror in `__tests__/`.
- Story files: co-locate with `*.stories.tsx`.
- Style files: co-locate with `*.module.css`, `*.module.scss`, or `styles.ts`.

## Examples

```
src/
  components/
    UserProfile/
      UserProfile.tsx
      UserProfile.test.tsx
      UserProfile.stories.tsx
      UserProfile.module.css
      index.ts                  # barrel: export { UserProfile } from './UserProfile'
  hooks/
    useAuth.ts
    useAuth.test.ts
  utils/
    formatDate.ts
    formatDate.test.ts
  types/
    user.ts
    api.ts
```

---

# Components

## Rules

- **PascalCase** — always, both function name and file name.
- Component name must match file name.
- **No** anonymous default exports — name every component.
- ForwardRef components: prefix with `ForwardRef` or use the wrapped name: `Input = forwardRef(InputImpl)`.
- Higher-Order Components: `with` prefix → `withAuth`, `withTheme`.
- Provider components: `XxxProvider` → `AuthProvider`, `ThemeProvider`.
- Context-consumer components: descriptive noun → `AuthConsumer` or, more commonly, hooks (see Hooks section).

## Examples

```tsx
// UserProfile.tsx — PascalCase file, PascalCase component
export function UserProfile({ userId }: UserProfileProps) {
  return <div>...</div>;
}

// HOC
export function withAuth<P>(Component: React.ComponentType<P>) {
  return function WithAuth(props: P) { ... };
}

// Provider
export function ThemeProvider({ children }: { children: React.ReactNode }) {
  return <ThemeContext.Provider value={theme}>{children}</ThemeContext.Provider>;
}
```

### Never

```tsx
// Bad: default anonymous export
export default function () { ... }

// Bad: lowercase component
export function userProfile() { ... }

// Bad: file doesn't match component name
// File: User.tsx
export function UserProfile() { ... }
```

---

# Props

## Rules

- **Props type**: component name + `Props` → `UserProfileProps`.
- Inline vs separate: separate type/interface is preferred for readability; inline OK for trivial one-liners.
- Callback props: `on` + past-tense or event noun → `onClick`, `onChange`, `onSubmit`, `onClose`, `onSelect`.
- Boolean props: no `is` for visible states — use adjectives; `disabled`, `required`, `readOnly`. For feature flags, `is` prefix OK: `isLoading`, `isError`.
- Children: `React.ReactNode`, no custom naming unless multiple children slots.
- Ref: `ref` forwarded via `forwardRef`, typed with `React.Ref<HTMLElement>`.
- Destructure in function signature, not inside body.

## Examples

```tsx
// Props type = ComponentName + Props
interface UserProfileProps {
  userId: string;
  onEdit?: (userId: string) => void;
  isLoading?: boolean;
  className?: string;
}

export function UserProfile({ userId, onEdit, isLoading, className }: UserProfileProps) { ... }

// Callback naming
interface ModalProps {
  onClose: () => void;
  onSubmit: (data: FormData) => void;
}

// Boolean props
interface ButtonProps {
  disabled?: boolean;        // adjective, no 'is'
  fullWidth?: boolean;       // adjective
  isLoading?: boolean;       // 'is' OK for loading/error flags
}
```

---

# Hooks

## Rules

- **`use` prefix** — React enforces this for rules-of-hooks linting: `useAuth`, `useForm`, `useDebounce`.
- **camelCase**: `useLocalStorage`, `useMediaQuery`.
- File name matches hook name: `useAuth.ts`, `useDebounce.ts`.
- One hook per file (exception: tightly coupled internal hooks).
- Custom hooks return a noun object or array; destructure consistently.

## Examples

```tsx
// useAuth.ts
export function useAuth(): { user: User | null; login: (c: Credentials) => Promise<void>; logout: () => void } {
  // ...
}

// useDebounce.ts
export function useDebounce<T>(value: T, delay: number): T {
  // ...
}

// Usage
const { user, login, logout } = useAuth();
const debouncedValue = useDebounce(input, 300);
```

### Never

```typescript
// Bad: no 'use' prefix
function auth() { ... }

// Bad: PascalCase hook
function UseAuth() { ... }
```

---

# Events and Handlers

## Rules

- **Handler functions**: `handle` + event noun → `handleClick`, `handleChange`, `handleSubmit`, `handleKeyDown`.
- Props that receive handlers: `on` + event noun → `onClick`, `onChange`, `onSubmit`.
- DOM event types: `React.MouseEvent`, `React.ChangeEvent<HTMLInputElement>`, `React.KeyboardEvent`, `React.FormEvent`.
- Never name handlers after what they do — name them after what triggers them. `handleClick` not `saveUser`.

## Examples

```tsx
// Handler naming: handle + Event
function UserForm() {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    // ...
  };

  const handleEmailChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    // ...
  };

  return (
    <form onSubmit={handleSubmit}>
      <input onChange={handleEmailChange} />
    </form>
  );
}

// When passing as prop — on + Event
interface FormProps {
  onSubmit: (data: FormData) => void;
  onCancel: () => void;
}
```

### Never

```typescript
// Bad: handler named after behavior, not trigger
const saveUser = () => { ... };
<button onClick={saveUser} />

// Good:
const handleClick = () => { ... };
<button onClick={handleClick} />

// Bad: prop doesn't match handler pattern
interface Props {
  submitForm: () => void;       // → onSubmit
  closeModal: () => void;       // → onClose
}
```

---

# Types and Interfaces

## Rules

- **PascalCase** for all types, interfaces, enums, type aliases: `User`, `ApiResponse`, `ButtonVariant`.
- **No** `I` prefix for interfaces (`IUser`). **No** `T` prefix for types (`TUser`).
- **No** Hungarian notation or type-encoding in names.
- Prefer `interface` over `type` for object shapes (extensibility, better error messages).
- Use `type` for unions, intersections, tuples, mapped types.
- Generic parameters: single uppercase letter for simple cases (`T`, `K`, `V`); descriptive PascalCase for complex ones (`TData`, `TError`, `TItem`).
- Props interfaces: `XxxProps` (see Props section).
- Enums: PascalCase for enum name and members: `enum Status { Active, Inactive }`. No `enum` suffix.
- Discriminated unions: shared `type` (literal) discriminant field: `type: 'success' | 'error' | 'pending'`.

## Examples

```typescript
// Interface — no I prefix
interface User {
  id: string;
  email: string;
  role: UserRole;
}

// Type — for unions, intersections, tuples
type UserRole = 'admin' | 'member' | 'viewer';
type ApiResult<T> = { data: T; error: null } | { data: null; error: string };

// Generic parameters
function identity<T>(value: T): T { return value; }
function useQuery<TData, TError>(query: string): QueryResult<TData, TError> { ... }

// Enum
enum ButtonVariant {
  Primary,
  Secondary,
  Ghost,
}

// Discriminated union
type FetchState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

// Never
interface IUser { ... }    // No I prefix
type TUser = { ... };      // No T prefix
type userType = { ... };   // No camelCase for types
```

---

# Variables and Functions

## Rules

- **camelCase** for variables, functions, parameters: `userName`, `fetchUsers`, `isActive`.
- **PascalCase** for React components and classes only.
- Constants: **camelCase** (same as variables). Use `UPPER_CASE` only for truly immutable env-level constants that won't change across deployments: `API_BASE_URL`, `MAX_RETRY_COUNT`.
- Boolean variables: `is`, `has`, `should`, `can` prefix → `isLoading`, `hasError`, `shouldRedirect`, `canEdit`.
- Arrays: plural noun → `users`, `items`, `selectedIds`. Not `userList` or `itemsArray`.
- Functions: verb or verb + noun → `fetchUsers`, `computeTotal`, `validateEmail`.
- Factory/constructor functions: `create` prefix → `createUser`, `createApiClient`.
- Async functions: no special suffix. The `async` keyword and return type (Promise) signal it. Historically `*Async` was used — not needed in modern TS.

## Examples

```typescript
// Variables
const userName = 'alice';
const isLoading = false;
const hasPermission = true;
const users: User[] = [];
const selectedIds: string[] = [];

// Functions
function fetchUsers(): Promise<User[]> { ... }
function validateEmail(email: string): boolean { ... }
function createApiClient(baseUrl: string): ApiClient { ... }

// Env-level constants only — UPPER_CASE
const API_BASE_URL = 'https://api.example.com';
const MAX_RETRY_COUNT = 3;

// Never
const userNameList = [];        // → users
const userArray: User[] = [];   // → users
const isDisabled = false;       // OK — boolean prefix
const getUsers = () => {};      // → fetchUsers (get implies getter, not async I/O)
```

---

# Classes

## Rules

- **PascalCase** for classes: `UserService`, `ApiClient`, `EventEmitter`.
- **camelCase** for class instances: `const userService = new UserService()`.
- Private fields: no `_` prefix; use TypeScript `private` keyword (or JS `#` for hard privacy).
- Static members: same casing as their instance counterparts.
- Abstract classes: PascalCase, no `Abstract` prefix.

## Examples

```typescript
class ApiClient {
  private baseUrl: string;        // private keyword, not _baseUrl

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }

  static create(url: string): ApiClient {
    return new ApiClient(url);
  }
}

const apiClient = new ApiClient('https://api.example.com');
```

---

# Enums and Union Types

## Rules

- **PascalCase** for enum name and members.
- Prefer **string enums** over numeric for readability and debugging.
- Consider `const enum` for zero-cost enums (with `preserveConstEnums` disabled in tsconfig).
- String literal unions preferred over enums for simpler cases (better type inference, fewer imports).

## Examples

```typescript
// String enum — PascalCase name + members
enum OrderStatus {
  Pending = 'pending',
  Shipped = 'shipped',
  Delivered = 'delivered',
  Cancelled = 'cancelled',
}

// String literal union — simpler alternative
type OrderStatus = 'pending' | 'shipped' | 'delivered' | 'cancelled';

// Union preferred when:
// - No need to iterate members
// - No need for reverse mapping
// - Simpler serialization/API boundary
```

---

# Constants and Config

## Rules

- **camelCase** for runtime constants defined in modules.
- **UPPER_CASE** only for build-time / environment-level values.
- Group related config in a single object with PascalCase basename.

## Examples

```typescript
// Runtime config — camelCase
const defaultPageSize = 20;
const maxUploadSizeMb = 10;

// Build-time / env — UPPER_CASE
const API_BASE_URL = import.meta.env.VITE_API_URL;
const IS_DEV = import.meta.env.DEV;

// Grouped config — PascalCase object, camelCase keys
const AppConfig = {
  apiBaseUrl: 'https://api.example.com',
  maxRetries: 3,
  timeoutMs: 5000,
} as const;
```

---

# State and Stores

## Rules

- **React state**: `[thing, setThing]` pattern → `const [user, setUser] = useState<User | null>(null)`.
- **Zustand stores**: `useXxxStore` → `useAuthStore`, `useCartStore`.
- **Redux slices**: `xxxSlice` → `userSlice`, `cartSlice`.
- **Context**: `XxxContext` → `AuthContext`, `ThemeContext`.
- **Reducers**: `xxxReducer` → `userReducer`, `cartReducer`.
- **Actions**: `xxxAction` or verb noun → `loginAction`, `addItemAction`, or follow toolkit conventions.

## Examples

```typescript
// useState — thing + setThing
const [isOpen, setIsOpen] = useState(false);
const [users, setUsers] = useState<User[]>([]);

// Zustand
const useCartStore = create<CartState>((set) => ({ ... }));

// Redux slice
const userSlice = createSlice({
  name: 'user',
  initialState,
  reducers: {
    login: (state, action: PayloadAction<User>) => { ... },
  },
});

// Context
const AuthContext = createContext<AuthContextType | null>(null);

// Reducer
function userReducer(state: UserState, action: UserAction): UserState { ... }
```

---

# Async and API

## Rules

- **Service modules**: PascalCase class or camelCase module of functions → `userService.ts`, `apiClient.ts`.
- **API functions**: verb + resource → `fetchUser`, `createOrder`, `updateProfile`, `deleteItem`.
- **API response types**: resource + `Response` or `Result` → `UserResponse`, `OrderListResponse`.
- **API error types**: resource + `Error` → `ApiError`, `ValidationError`.
- **React Query / SWR keys**: PascalCase enum or `UPPER_CASE` constants → `QueryKeys.Users` or `USER_QUERY_KEY`.

## Examples

```typescript
// Service — camelCase file, exported functions
// file: userService.ts
export async function fetchUser(id: string): Promise<User> { ... }
export async function updateUser(id: string, data: UpdateUserInput): Promise<User> { ... }
export async function deleteUser(id: string): Promise<void> { ... }

// Response types
interface UserResponse {
  user: User;
}

interface ApiError {
  code: string;
  message: string;
  field?: string;
}

// Query keys
export const userKeys = {
  all: ['users'] as const,
  detail: (id: string) => ['users', id] as const,
  list: (filters: UserFilters) => ['users', 'list', filters] as const,
};
```

---

# Barrel Exports (index.ts)

## Rules

- `index.ts` files only re-export — no implementation.
- Use named exports, avoid `export { default } from ...` — default exports don't tree-shake well.
- Don't re-export everything; be intentional about the public API surface.

## Examples

```typescript
// components/UserProfile/index.ts
export { UserProfile } from './UserProfile';
export type { UserProfileProps } from './UserProfile';

// hooks/index.ts
export { useAuth } from './useAuth';
export { useDebounce } from './useDebounce';

// Never
// Bad: implementation in index.ts
export function UserProfile() { ... }

// Bad: re-export all
export * from './UserProfile';
```

---

# Test Files

## Rules

- **Describe blocks**: feature or component name → `describe('UserProfile', () => { ... })`.
- **It blocks**: present-tense behavior, not implementation → `it('renders user name when loaded', () => { ... })`.
- **Test variables**: follow same conventions as production code.
- **Mocks/fixtures**: `mockXxx` or descriptive → `mockUser`, `mockApiResponse`, `createMockUser()`.
- **Test utilities**: `renderWithProviders`, `setup`, `arrange`.

## Examples

```typescript
// UserProfile.test.tsx
import { render, screen } from '@testing-library/react';
import { UserProfile } from './UserProfile';

describe('UserProfile', () => {
  it('renders the user name when loading completes', () => {
    render(<UserProfile userId="1" />);
    expect(screen.getByText('Alice')).toBeInTheDocument();
  });

  it('shows a loading spinner while fetching', () => {
    render(<UserProfile userId="1" />);
    expect(screen.getByRole('progressbar')).toBeInTheDocument();
  });

  it('calls onEdit when the edit button is clicked', () => {
    const handleEdit = vi.fn();
    render(<UserProfile userId="1" onEdit={handleEdit} />);
    // ...
  });
});
```

### Never

```typescript
// Bad: test describes implementation
it('calls setState with the user object', () => { ... });

// Bad: present tense not used
it('rendered user name', () => { ... });

// Good: describes observable behavior
it('renders user name when data loads successfully', () => { ... });
```

---

# Quick Reference

| Thing | Convention | Example |
|-------|-----------|---------|
| Component file | PascalCase | `UserProfile.tsx` |
| Component function | PascalCase | `UserProfile` |
| Hook file | camelCase, `use` prefix | `useAuth.ts` |
| Hook function | camelCase, `use` prefix | `useAuth()` |
| Utility file | camelCase | `formatDate.ts` |
| Props type | PascalCase + `Props` | `UserProfileProps` |
| Callback prop | `on` + event | `onClick`, `onChange` |
| Handler function | `handle` + event | `handleClick` |
| Boolean variable | `is`/`has`/`should` prefix | `isLoading`, `hasError` |
| Interface | PascalCase, no `I` | `User`, `ApiResponse` |
| Type alias | PascalCase, no `T` | `UserRole`, `FetchState` |
| Enum | PascalCase | `OrderStatus` |
| Generic param | single letter or PascalCase | `T`, `TData` |
| State variable | thing + setThing | `[user, setUser]` |
| Context | PascalCase + `Context` | `AuthContext` |
| Reducer | camelCase + `Reducer` | `userReducer` |
| Service | `XxxService` or camelCase module | `userService.ts`, `apiClient.ts` |
| Constant | camelCase (UPPER_CASE for env) | `defaultTimeout`, `API_URL` |
| Array | plural noun | `users`, `selectedIds` |
| Test file | `*.test.ts(x)` or `*.spec.ts(x)` | `UserProfile.test.tsx` |
| Story file | `*.stories.tsx` | `UserProfile.stories.tsx` |
| Style file | `*.module.css` | `UserProfile.module.css` |

---

# What to Avoid

| Anti-pattern | Correct |
|-------------|---------|
| `IUser` interface | `User` |
| `TUser` type | `User` |
| `user_list` or `user-list` in code | `userList` |
| `UPPER_CASE` for non-env constants | `camelCase` |
| `GetUsers` function | `fetchUsers` or `getUsers` |
| Default anonymous export | Named export |
| `this`, `that` variable names | Descriptive noun |
| `data`, `item`, `obj` in wide scope | Descriptive name |
| `handleClick` prop (should be `onClick`) | Props use `on`, handlers use `handle` |
| `_privateField` in class | `private` keyword |
| Implementation in `index.ts` | Barrel re-exports only |
| `ComponentName` mismatch with file name | File == component name |
| `useMyHook` in non-hook file | Hook file == hook name |

---

# Enforcement

When writing TypeScript + React code:

1. Follow these conventions as defaults — no justification needed.
2. Deviate only when following a well-known library's precedent (e.g., Next.js page conventions, Redux Toolkit patterns).
3. When in doubt, check the [Airbnb React Style Guide](https://airbnb.io/javascript/react/) or the TypeScript handbook.
