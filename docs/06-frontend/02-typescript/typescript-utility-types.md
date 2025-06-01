# TypeScript Utility Types

`keywords: Partial, Required, Readonly, Pick, Omit, Record, Extract, Exclude, NonNullable, ReturnType, Parameters, Awaited, utility-types, custom-types`

## Principle

**Use built-in utility types to transform existing types rather than defining new ones from scratch.** TypeScript ships with a comprehensive set of utility types that handle common type transformations. Master these before writing custom type-level code. When built-in types are insufficient, compose custom utilities from the same primitives.

---

## Built-In Utility Types

### `Partial<T>`

Makes all properties of `T` optional. Use for update/patch operations where any subset of fields may be provided.

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  role: "admin" | "user";
}

// All properties become optional
type UserUpdate = Partial<User>;
// {
//   id?: string;
//   name?: string;
//   email?: string;
//   role?: "admin" | "user";
// }

function updateUser(id: string, changes: Partial<User>): void {
  // changes can have any subset of User properties
}

updateUser("1", { name: "Alice" });          // OK
updateUser("1", { name: "Alice", role: "admin" }); // OK
updateUser("1", {});                         // OK
```

### `Required<T>`

Makes all properties of `T` required (removes `?` modifier). The opposite of `Partial<T>`.

```typescript
interface Config {
  host?: string;
  port?: number;
  debug?: boolean;
}

// All properties become required
type ResolvedConfig = Required<Config>;
// {
//   host: string;
//   port: number;
//   debug: boolean;
// }

function startServer(config: ResolvedConfig): void {
  // All values guaranteed to exist
  console.log(`Starting on ${config.host}:${config.port}`);
}

// Resolve defaults before passing
function resolveConfig(partial: Config): ResolvedConfig {
  return {
    host: partial.host ?? "localhost",
    port: partial.port ?? 3000,
    debug: partial.debug ?? false,
  };
}
```

### `Readonly<T>`

Makes all properties of `T` readonly. Use for immutable data structures.

```typescript
interface State {
  count: number;
  items: string[];
}

type FrozenState = Readonly<State>;
// {
//   readonly count: number;
//   readonly items: string[];  // Note: array contents are still mutable!
// }

const state: FrozenState = { count: 0, items: ["a"] };
state.count = 1;        // ERROR: Cannot assign to 'count'
state.items.push("b");  // OK! Readonly is shallow.

// For deep readonly, use a recursive type or a library
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
```

### `Pick<T, K>`

Creates a type with only the specified properties from `T`. Use to create focused subtypes.

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// Only include specific properties
type PublicUser = Pick<User, "id" | "name" | "email">;
// {
//   id: string;
//   name: string;
//   email: string;
// }

// Use for API response types that expose a subset of fields
function toPublicUser(user: User): PublicUser {
  return {
    id: user.id,
    name: user.name,
    email: user.email,
  };
}
```

### `Omit<T, K>`

Creates a type with all properties from `T` except the specified ones. The inverse of `Pick`.

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// Exclude sensitive fields
type SafeUser = Omit<User, "password">;
// {
//   id: string;
//   name: string;
//   email: string;
//   createdAt: Date;
// }

// Create input type: omit auto-generated fields
type CreateUserInput = Omit<User, "id" | "createdAt">;
// {
//   name: string;
//   email: string;
//   password: string;
// }
```

### `Record<K, V>`

Creates a type with keys of type `K` and values of type `V`. Use for dictionaries and lookup maps.

```typescript
// String keys, specific value type
type UserMap = Record<string, User>;

const users: UserMap = {
  "user-1": { id: "user-1", name: "Alice", email: "a@b.com", password: "...", createdAt: new Date() },
};

// Union keys: ensures all keys are present
type Role = "admin" | "user" | "viewer";
type RolePermissions = Record<Role, string[]>;

const permissions: RolePermissions = {
  admin: ["read", "write", "delete"],
  user: ["read", "write"],
  viewer: ["read"],
  // ERROR if any role is missing
};

// Enum-like mapping
type HttpStatus = 200 | 400 | 404 | 500;
type StatusMessages = Record<HttpStatus, string>;

const messages: StatusMessages = {
  200: "OK",
  400: "Bad Request",
  404: "Not Found",
  500: "Internal Server Error",
};
```

### `Extract<T, U>`

Extracts from `T` all members that are assignable to `U`.

```typescript
type AllTypes = string | number | boolean | null | undefined;

// Extract only string and number
type Primitives = Extract<AllTypes, string | number>;
// string | number

// Extract specific union members
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number }
  | { kind: "triangle"; base: number; height: number };

type CircleShape = Extract<Shape, { kind: "circle" }>;
// { kind: "circle"; radius: number }
```

### `Exclude<T, U>`

Removes from `T` all members that are assignable to `U`. The inverse of `Extract`.

```typescript
type AllTypes = string | number | boolean | null | undefined;

// Exclude null and undefined
type NonNullPrimitives = Exclude<AllTypes, null | undefined>;
// string | number | boolean

// Exclude specific union members
type NonCircleShape = Exclude<Shape, { kind: "circle" }>;
// { kind: "rectangle"; ... } | { kind: "triangle"; ... }

// Practical: remove specific event types
type AppEvent = "click" | "focus" | "blur" | "submit" | "reset";
type NonFormEvent = Exclude<AppEvent, "submit" | "reset">;
// "click" | "focus" | "blur"
```

### `NonNullable<T>`

Removes `null` and `undefined` from `T`.

```typescript
type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>;
// string

// Useful with mapped types
type NullableUser = {
  id: string | null;
  name: string | undefined;
  email: string | null | undefined;
};

type RequiredUser = {
  [K in keyof NullableUser]: NonNullable<NullableUser[K]>;
};
// { id: string; name: string; email: string }
```

### `ReturnType<T>`

Extracts the return type of a function type.

```typescript
function createUser(name: string, email: string) {
  return {
    id: crypto.randomUUID(),
    name,
    email,
    createdAt: new Date(),
  };
}

type User = ReturnType<typeof createUser>;
// {
//   id: string;
//   name: string;
//   email: string;
//   createdAt: Date;
// }

// Useful when you don't control the function definition
import { someLibraryFunction } from "some-library";
type LibResult = ReturnType<typeof someLibraryFunction>;
```

### `Parameters<T>`

Extracts the parameter types of a function type as a tuple.

```typescript
function greet(name: string, age: number, formal: boolean): string {
  return formal ? `Good day, ${name}.` : `Hey ${name}!`;
}

type GreetParams = Parameters<typeof greet>;
// [name: string, age: number, formal: boolean]

// Access individual parameters
type FirstParam = Parameters<typeof greet>[0]; // string
type SecondParam = Parameters<typeof greet>[1]; // number

// Useful for wrapping functions
function withLogging<T extends (...args: unknown[]) => unknown>(
  fn: T
): (...args: Parameters<T>) => ReturnType<T> {
  return (...args: Parameters<T>) => {
    console.log("Calling with:", args);
    return fn(...args) as ReturnType<T>;
  };
}
```

### `ConstructorParameters<T>`

Extracts the parameter types of a constructor function.

```typescript
class HttpClient {
  constructor(
    private baseUrl: string,
    private timeout: number,
    private headers: Record<string, string>
  ) {}
}

type HttpClientArgs = ConstructorParameters<typeof HttpClient>;
// [baseUrl: string, timeout: number, headers: Record<string, string>]

// Create factory function
function createClient(...args: ConstructorParameters<typeof HttpClient>): HttpClient {
  return new HttpClient(...args);
}
```

### `InstanceType<T>`

Extracts the instance type of a constructor function.

```typescript
class UserService {
  findById(id: string): User | null {
    return null;
  }
}

type UserServiceInstance = InstanceType<typeof UserService>;
// UserService

// Useful for generic factories
function createInstance<T extends new (...args: unknown[]) => unknown>(
  ctor: T
): InstanceType<T> {
  return new ctor() as InstanceType<T>;
}
```

### `Awaited<T>`

Recursively unwraps `Promise` types. Added in TypeScript 4.5.

```typescript
type A = Awaited<Promise<string>>;           // string
type B = Awaited<Promise<Promise<number>>>;  // number
type C = Awaited<string>;                     // string (non-promise passes through)

// Practical: extract resolved type of async functions
async function fetchUsers(): Promise<User[]> {
  return [];
}

type Users = Awaited<ReturnType<typeof fetchUsers>>;
// User[]
```

---

## When to Use Each Utility Type

| Scenario | Utility Type | Example |
|----------|-------------|---------|
| Patch/update operations | `Partial<T>` | `updateUser(id, changes: Partial<User>)` |
| Resolve optional config | `Required<T>` | `Required<Config>` after applying defaults |
| Immutable state | `Readonly<T>` | `Readonly<State>` for store snapshots |
| Public API subset | `Pick<T, K>` | `Pick<User, "id" \| "name">` |
| Remove sensitive fields | `Omit<T, K>` | `Omit<User, "password">` |
| Dictionary/lookup | `Record<K, V>` | `Record<string, User>` |
| Filter union members | `Extract<T, U>` | `Extract<Event, { type: "click" }>` |
| Remove union members | `Exclude<T, U>` | `Exclude<Event, "internal">` |
| Remove null/undefined | `NonNullable<T>` | `NonNullable<string \| null>` |
| Get function return type | `ReturnType<T>` | `ReturnType<typeof fn>` |
| Get function param types | `Parameters<T>` | `Parameters<typeof fn>` |
| Unwrap promises | `Awaited<T>` | `Awaited<ReturnType<typeof asyncFn>>` |
| Auto-generate input type | `Omit<T, "id" \| "createdAt">` | Create forms from models |

---

## Creating Custom Utility Types

### Make Specific Properties Optional

```typescript
// Make only specified keys optional, keep the rest required
type PartialBy<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;

interface User {
  id: string;
  name: string;
  email: string;
  bio: string;
}

type CreateUserInput = PartialBy<User, "id" | "bio">;
// { name: string; email: string; id?: string; bio?: string }
```

### Make Specific Properties Required

```typescript
type RequiredBy<T, K extends keyof T> = T & Required<Pick<T, K>>;

interface Config {
  host?: string;
  port?: number;
  ssl?: boolean;
}

type ProductionConfig = RequiredBy<Config, "host" | "ssl">;
// { host: string; port?: number; ssl: boolean }
```

### Deep Partial

```typescript
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

interface AppConfig {
  database: {
    host: string;
    port: number;
    credentials: {
      username: string;
      password: string;
    };
  };
  cache: {
    ttl: number;
    maxSize: number;
  };
}

// All nested properties are optional
type ConfigOverride = DeepPartial<AppConfig>;
// Can specify { database: { port: 5433 } } without other fields
```

### Mutable (Remove Readonly)

```typescript
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

interface FrozenUser {
  readonly id: string;
  readonly name: string;
}

type EditableUser = Mutable<FrozenUser>;
// { id: string; name: string } -- no longer readonly
```

### Nullable Properties

```typescript
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

interface FormValues {
  name: string;
  email: string;
  age: number;
}

type FormState = Nullable<FormValues>;
// { name: string | null; email: string | null; age: number | null }
```

---

## Practical Examples

### API Response Types

```typescript
// Base entity
interface Entity {
  id: string;
  createdAt: string;
  updatedAt: string;
}

// Create input: omit auto-generated fields
type CreateInput<T extends Entity> = Omit<T, keyof Entity>;

// Update input: partial non-id fields
type UpdateInput<T extends Entity> = Partial<Omit<T, "id">> & Pick<T, "id">;

// List response: paginated
interface ListResponse<T> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
  hasMore: boolean;
}

// Usage
interface Product extends Entity {
  name: string;
  price: number;
  category: string;
}

type CreateProduct = CreateInput<Product>;
// { name: string; price: number; category: string }

type UpdateProduct = UpdateInput<Product>;
// { id: string; name?: string; price?: number; category?: string; createdAt?: string; updatedAt?: string }

type ProductList = ListResponse<Product>;
```

### Form Types

```typescript
// Generate form field metadata from a model
type FormField<T> = {
  value: T;
  error: string | null;
  touched: boolean;
  dirty: boolean;
};

type FormFields<T> = {
  [K in keyof T]: FormField<T[K]>;
};

interface LoginData {
  email: string;
  password: string;
  rememberMe: boolean;
}

type LoginForm = FormFields<LoginData>;
// {
//   email: FormField<string>;
//   password: FormField<string>;
//   rememberMe: FormField<boolean>;
// }

// Extract just the values from a form
type FormValues<T extends Record<string, FormField<unknown>>> = {
  [K in keyof T]: T[K] extends FormField<infer V> ? V : never;
};

type LoginValues = FormValues<LoginForm>;
// { email: string; password: string; rememberMe: boolean }
```

### Event Handler Types

```typescript
// Type-safe event handlers derived from an event map
type EventMap = {
  "user:created": { userId: string; email: string };
  "user:deleted": { userId: string };
  "order:placed": { orderId: string; total: number };
  "order:cancelled": { orderId: string; reason: string };
};

type EventHandler<T extends keyof EventMap> = (payload: EventMap[T]) => void;

// Type-safe event registration
type EventHandlers = {
  [K in keyof EventMap]?: EventHandler<K>[];
};

// Extract event names by prefix
type UserEvents = Extract<keyof EventMap, `user:${string}`>;
// "user:created" | "user:deleted"

type OrderEvents = Extract<keyof EventMap, `order:${string}`>;
// "order:placed" | "order:cancelled"
```

### Discriminated Union Helpers

```typescript
// Extract a specific variant from a discriminated union
type VariantOf<T, K> = Extract<T, { kind: K }>;

type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rect"; width: number; height: number };

type Circle = VariantOf<Shape, "circle">;
// { kind: "circle"; radius: number }

// Get all discriminant values
type KindOf<T extends { kind: string }> = T["kind"];

type ShapeKind = KindOf<Shape>;
// "circle" | "rect"
```

---

## Best Practices

### DO

- **DO** use `Pick` and `Omit` to create focused subtypes from larger interfaces
- **DO** use `Partial<T>` for update/patch operations and `Required<T>` for resolved configuration
- **DO** use `Record<K, V>` with union keys to ensure all keys are handled
- **DO** compose utility types: `Readonly<Pick<User, "id" | "name">>` is valid and useful
- **DO** use `ReturnType` and `Parameters` to derive types from existing functions
- **DO** create project-specific utility types when built-in types are insufficient

### DON'T

- **DON'T** redefine built-in utility types -- use the ones TypeScript provides
- **DON'T** nest utility types more than 2-3 levels deep -- extract named types
- **DON'T** use `Record<string, any>` -- use `Record<string, unknown>` instead
- **DON'T** use `Omit` when `Pick` would be clearer (prefer positive selection for small subsets)
- **DON'T** create overly clever utility types that teammates cannot understand
- **DON'T** forget that `Readonly<T>` is shallow -- use `DeepReadonly` for nested objects

---

## Guidelines

### Essential

- Use `Omit<T, "id" | "createdAt">` for create-input types derived from entity models
- Use `Partial<T>` for update/patch operations
- Use `Record<K, V>` with union keys to enforce completeness
- Use `ReturnType<typeof fn>` to derive types from functions you don't control

### Recommended

- Create `CreateInput<T>`, `UpdateInput<T>` utility types for consistent API patterns
- Use `Pick` to create public-facing types that hide internal fields
- Compose utility types rather than writing one-off interfaces
- Use `NonNullable<T>` in mapped types to strip null from all properties

### Advanced

- Build form type generators using mapped types and `FormField<T>` pattern
- Use `Extract` and `Exclude` with template literal types for event filtering
- Create recursive utility types (`DeepPartial`, `DeepReadonly`) for nested structures
- Use `infer` in conditional types to extract types from generic wrappers

---

## Benefits

- Reduces boilerplate by deriving types from existing definitions
- Ensures type consistency across create, update, and read operations
- Makes type transformations explicit and self-documenting
- Catches missing cases with `Record<UnionKey, V>` completeness checking
- Enables single-source-of-truth type definitions
- Leverages compiler-verified transformations instead of manual type duplication

---

## Related

- [typescript-generics.md](typescript-generics.md) -- Generic functions, constraints, and conditional types
- [typescript-type-safety.md](typescript-type-safety.md) -- Type guards, narrowing, and runtime validation
