# TypeScript Type Safety

`keywords: type-guards, type-assertions, unknown, any, never, satisfies, const-assertions, discriminated-unions, zod, runtime-validation`

## Principle

**Prefer type narrowing over type assertions.** Type guards prove safety at runtime; type assertions merely promise safety to the compiler. Use `unknown` instead of `any`, exhaustive checks with `never`, and runtime validation at system boundaries. The type system should work for you, not against you.

---

## Type Assertions vs Type Guards

Type assertions (`as`) tell the compiler "trust me." Type guards tell the compiler "I checked."

### Type Assertions (Use Sparingly)

```typescript
// Type assertion: you are overriding the compiler
const input = document.getElementById("name") as HTMLInputElement;
input.value = "Alice"; // Crashes if element is not an input

// Double assertion: a code smell indicating a design problem
const value = someValue as unknown as TargetType; // Almost always wrong
```

Type assertions are appropriate in exactly two scenarios:
1. You have external knowledge the compiler cannot infer (e.g., DOM element types)
2. Working with test mocks where full implementation is unnecessary

### Type Guards (Prefer Always)

```typescript
// Type guard: runtime check narrows the type
const element = document.getElementById("name");
if (element instanceof HTMLInputElement) {
  element.value = "Alice"; // Safe: proven to be HTMLInputElement
}

// typeof guard
function formatValue(value: string | number): string {
  if (typeof value === "string") {
    return value.toUpperCase(); // string
  }
  return value.toFixed(2); // number
}
```

---

## User-Defined Type Guards

### `is` Type Predicates

Custom functions that narrow types. The return type `x is T` tells TypeScript that a `true` return means the parameter is of type `T`.

```typescript
interface Cat {
  kind: "cat";
  purr: () => void;
}

interface Dog {
  kind: "dog";
  bark: () => void;
}

type Animal = Cat | Dog;

// User-defined type guard
function isCat(animal: Animal): animal is Cat {
  return animal.kind === "cat";
}

function interact(animal: Animal) {
  if (isCat(animal)) {
    animal.purr(); // TypeScript knows this is Cat
  } else {
    animal.bark(); // TypeScript knows this is Dog
  }
}
```

### `asserts` Type Predicates

Assertion functions throw if the condition is not met. After the call, TypeScript narrows the type in the remaining scope.

```typescript
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new Error(`Expected string, got ${typeof value}`);
  }
}

function processInput(input: unknown) {
  assertIsString(input);
  // TypeScript knows input is string from here onward
  console.log(input.toUpperCase());
}

// Assert non-null
function assertDefined<T>(value: T | null | undefined, name: string): asserts value is T {
  if (value === null || value === undefined) {
    throw new Error(`Expected ${name} to be defined`);
  }
}
```

---

## `unknown` vs `any`

### `any` -- The Escape Hatch (Avoid)

`any` disables all type checking. It is contagious: any expression involving `any` becomes `any`.

```typescript
// any silently spreads through your code
function processData(data: any) {
  const name = data.user.name;      // any -- no checking
  const upper = name.toUpperCase(); // any -- no checking
  const length = upper.length;      // any -- no checking
  return length;                    // any -- caller gets any too
}
```

### `unknown` -- The Safe Alternative (Prefer)

`unknown` is the type-safe counterpart to `any`. You cannot do anything with `unknown` until you narrow it.

```typescript
function processData(data: unknown) {
  // ERROR: Object is of type 'unknown'
  // const name = data.user.name;

  // Must narrow first
  if (
    typeof data === "object" &&
    data !== null &&
    "user" in data &&
    typeof (data as Record<string, unknown>).user === "object"
  ) {
    // Still cumbersome -- this is why Zod exists (see below)
  }
}

// Practical pattern: unknown + type guard
function isUser(value: unknown): value is { name: string; email: string } {
  return (
    typeof value === "object" &&
    value !== null &&
    "name" in value &&
    "email" in value &&
    typeof (value as Record<string, unknown>).name === "string" &&
    typeof (value as Record<string, unknown>).email === "string"
  );
}

function processData(data: unknown) {
  if (isUser(data)) {
    console.log(data.name); // Safe
  }
}
```

---

## The `never` Type

`never` represents values that should never occur. It is the bottom type -- a subtype of every type, but no type is a subtype of `never`.

### Exhaustive Checking

The most powerful use of `never` is ensuring all cases of a union are handled.

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number }
  | { kind: "triangle"; base: number; height: number };

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "rectangle":
      return shape.width * shape.height;
    case "triangle":
      return (shape.base * shape.height) / 2;
    default: {
      // If all cases are handled, shape is 'never' here.
      // If someone adds a new shape kind, this line will produce a compile error.
      const _exhaustive: never = shape;
      throw new Error(`Unhandled shape: ${JSON.stringify(_exhaustive)}`);
    }
  }
}
```

### Helper Function for Exhaustive Checks

```typescript
function assertNever(value: never, message?: string): never {
  throw new Error(message ?? `Unexpected value: ${JSON.stringify(value)}`);
}

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "rectangle":
      return shape.width * shape.height;
    case "triangle":
      return (shape.base * shape.height) / 2;
    default:
      return assertNever(shape); // Compile error if cases are missing
  }
}
```

---

## The `satisfies` Operator

Introduced in TypeScript 4.9, `satisfies` validates that an expression matches a type without widening the inferred type.

```typescript
type ColorMap = Record<string, [number, number, number] | string>;

// With 'as': widens the type, losing specific key information
const colors1 = {
  red: [255, 0, 0],
  green: "#00ff00",
} as ColorMap;
// colors1.red is [number, number, number] | string -- lost the specific type

// With 'satisfies': validates but preserves narrow types
const colors2 = {
  red: [255, 0, 0],
  green: "#00ff00",
} satisfies ColorMap;
// colors2.red is [number, number, number] -- preserved!
// colors2.green is string -- preserved!

colors2.red.map((c) => c / 255);    // Works: TypeScript knows it's a tuple
colors2.green.toUpperCase();          // Works: TypeScript knows it's a string
```

### Practical Use Cases

```typescript
// Validate configuration objects
interface AppConfig {
  apiUrl: string;
  timeout: number;
  features: Record<string, boolean>;
}

const config = {
  apiUrl: "https://api.example.com",
  timeout: 5000,
  features: {
    darkMode: true,
    beta: false,
  },
} satisfies AppConfig;
// Type error if config doesn't match AppConfig
// But config.features.darkMode is inferred as boolean (not Record<string, boolean>)

// Validate route definitions
const routes = {
  home: "/",
  user: "/users/:id",
  settings: "/settings",
} satisfies Record<string, string>;
// routes.home is string, but TypeScript will error if a value is not a string
```

---

## Const Assertions

`as const` makes literals deeply immutable and narrows their types to literal types.

```typescript
// Without as const: widened types
const config = {
  endpoint: "/api/users",
  method: "GET",
  retries: 3,
};
// config.method is string

// With as const: narrow literal types
const config = {
  endpoint: "/api/users",
  method: "GET",
  retries: 3,
} as const;
// config.method is "GET" (literal type)
// config is deeply readonly

// Useful for defining discriminated union values
const Status = {
  Active: "active",
  Inactive: "inactive",
  Pending: "pending",
} as const;

type Status = (typeof Status)[keyof typeof Status];
// type Status = "active" | "inactive" | "pending"
```

### Const Type Parameters (TypeScript 5.0+)

```typescript
// Without const type parameter
function createRoute<T extends readonly string[]>(parts: T) {
  return parts;
}
const route = createRoute(["users", "profile"]); // string[]

// With const type parameter: infers literal tuple
function createRoute<const T extends readonly string[]>(parts: T) {
  return parts;
}
const route = createRoute(["users", "profile"]); // readonly ["users", "profile"]
```

---

## Readonly Properties and Arrays

### Readonly Properties

```typescript
interface User {
  readonly id: string;
  readonly email: string;
  name: string; // Mutable -- only name can be changed after creation
}

const user: User = { id: "1", email: "a@b.com", name: "Alice" };
user.name = "Bob";   // OK
user.email = "new";  // ERROR: Cannot assign to 'email' because it is a read-only property
```

### Readonly Arrays and Tuples

```typescript
// Readonly array: cannot push, pop, splice, or reassign elements
function processItems(items: readonly string[]) {
  items.push("new");   // ERROR: Property 'push' does not exist on type 'readonly string[]'
  items[0] = "new";    // ERROR: Index signature in type 'readonly string[]' only permits reading

  // Allowed: non-mutating operations
  const filtered = items.filter((i) => i.length > 0);
  const mapped = items.map((i) => i.toUpperCase());
  const first = items[0];
}

// ReadonlyArray<T> is equivalent to readonly T[]
function processItems(items: ReadonlyArray<string>) { /* same constraints */ }
```

---

## Template Literal Types

Build string types from other types using template literal syntax.

```typescript
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";
type ApiPath = "/users" | "/posts" | "/comments";

// Generates all 12 combinations
type ApiEndpoint = `${HttpMethod} ${ApiPath}`;
// "GET /users" | "GET /posts" | "GET /comments" | "POST /users" | ...

// Event handler names
type EventName = "click" | "focus" | "blur";
type HandlerName = `on${Capitalize<EventName>}`;
// "onClick" | "onFocus" | "onBlur"

// CSS units
type CssUnit = "px" | "rem" | "em" | "%";
type CssValue = `${number}${CssUnit}`;
// e.g., "16px", "1.5rem"
```

---

## Discriminated Unions for Type Safety

Discriminated unions combine union types with a shared literal discriminant property, enabling safe narrowing.

```typescript
// API response pattern
type ApiResponse<T> =
  | { status: "success"; data: T; timestamp: number }
  | { status: "error"; error: string; code: number }
  | { status: "loading" };

function handleResponse(response: ApiResponse<User>) {
  switch (response.status) {
    case "success":
      console.log(response.data.name);  // Safe: data exists
      break;
    case "error":
      console.error(response.error);     // Safe: error exists
      break;
    case "loading":
      console.log("Loading...");
      // response.data would be an error here
      break;
  }
}

// State machine pattern
type ConnectionState =
  | { state: "disconnected" }
  | { state: "connecting"; attempt: number }
  | { state: "connected"; socket: WebSocket }
  | { state: "error"; error: Error; retryAfter: number };

function renderStatus(conn: ConnectionState): string {
  switch (conn.state) {
    case "disconnected":
      return "Not connected";
    case "connecting":
      return `Connecting (attempt ${conn.attempt})...`;
    case "connected":
      return `Connected via ${conn.socket.url}`;
    case "error":
      return `Error: ${conn.error.message}. Retry in ${conn.retryAfter}s`;
  }
}
```

---

## Runtime Validation with Zod

TypeScript types are erased at runtime. At system boundaries (API responses, user input, configuration files), use a runtime validation library.

### Zod Schema Definition

```typescript
import { z } from "zod";

// Define schema (runtime) and infer type (compile time) from a single source
const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string().min(1).max(100),
  role: z.enum(["admin", "user", "viewer"]),
  createdAt: z.string().datetime(),
  metadata: z.record(z.string(), z.unknown()).optional(),
});

// Infer TypeScript type from schema -- single source of truth
type User = z.infer<typeof UserSchema>;
// {
//   id: string;
//   email: string;
//   name: string;
//   role: "admin" | "user" | "viewer";
//   createdAt: string;
//   metadata?: Record<string, unknown>;
// }
```

### Parsing API Responses

```typescript
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const json: unknown = await response.json();

  // Throws ZodError with detailed messages if validation fails
  return UserSchema.parse(json);
}

// Safe parsing (returns result instead of throwing)
async function fetchUserSafe(id: string): Promise<User | null> {
  const response = await fetch(`/api/users/${id}`);
  const json: unknown = await response.json();

  const result = UserSchema.safeParse(json);
  if (result.success) {
    return result.data; // Fully typed User
  }
  console.error("Validation failed:", result.error.issues);
  return null;
}
```

### Valibot as a Lighter Alternative

```typescript
import * as v from "valibot";

const UserSchema = v.object({
  id: v.pipe(v.string(), v.uuid()),
  email: v.pipe(v.string(), v.email()),
  name: v.pipe(v.string(), v.minLength(1), v.maxLength(100)),
  role: v.picklist(["admin", "user", "viewer"]),
});

type User = v.InferOutput<typeof UserSchema>;
```

Valibot is tree-shakeable, making it significantly smaller in bundle size when only a subset of validators is used.

---

## Best Practices

### DO

- **DO** use `unknown` instead of `any` for values of uncertain type
- **DO** write user-defined type guards (`is` predicates) for complex type narrowing
- **DO** use exhaustive `never` checks in switch statements over discriminated unions
- **DO** use `satisfies` to validate types without widening inferred types
- **DO** use `as const` for literal object and array values that should not be mutated
- **DO** validate data at system boundaries with Zod or Valibot
- **DO** use `readonly` for arrays and properties that should not be mutated

### DON'T

- **DON'T** use `any` unless interfacing with a JavaScript library that has no types
- **DON'T** use type assertions (`as`) when a type guard is possible
- **DON'T** use double assertions (`as unknown as T`) -- redesign the types instead
- **DON'T** skip exhaustive checks in switch statements over unions
- **DON'T** rely on TypeScript types alone for data from external sources (APIs, files, user input)
- **DON'T** use the non-null assertion operator (`!`) as a general-purpose null suppressor

---

## Guidelines

### Essential

- Enable `strict: true` and `noUncheckedIndexedAccess: true` in tsconfig
- Use `unknown` for all external data (API responses, parsed JSON, user input)
- Validate all data at system boundaries with a schema library
- Use discriminated unions for state that can be in one of several states

### Recommended

- Write assertion functions for preconditions (`asserts value is T`)
- Use `satisfies` for configuration objects and route definitions
- Prefer `readonly` arrays in function parameters
- Use `as const` for enum-like constant objects

### Advanced

- Build type-safe event systems using template literal types
- Use conditional types and `infer` for advanced generic patterns
- Implement branded types for domain identifiers (see typescript-branded-types.md)
- Use Zod's `.transform()` for parsing and transforming in a single step

---

## Benefits

- Eliminates entire categories of runtime errors at compile time
- Makes impossible states impossible through discriminated unions
- Provides self-documenting code through explicit type narrowing
- Enables safe refactoring by catching type mismatches immediately
- Validates external data before it enters the application
- Reduces debugging time by catching errors earlier in the development cycle

---

## Related

- [typescript-strict-mode.md](typescript-strict-mode.md) -- Compiler options that enforce type safety
- [typescript-discriminated-unions.md](typescript-discriminated-unions.md) -- Deep dive into union type patterns
- [typescript-branded-types.md](typescript-branded-types.md) -- Nominal typing for domain safety
