# TypeScript Generics

`keywords: generics, constraints, extends, infer, conditional-types, mapped-types, template-literal-types, Result, Maybe, AsyncData, composables`

## Principle

**Use generics to write type-safe abstractions that work across multiple types.** Generics allow you to define functions, interfaces, and classes that operate on types as parameters, enabling reuse without sacrificing type safety. Prefer constrained generics over `any`, and keep generic signatures as simple as possible.

---

## Generic Functions

Generic functions accept type parameters that are inferred from their arguments.

### Basic Generic Functions

```typescript
// Without generics: loses type information
function first(arr: unknown[]): unknown {
  return arr[0];
}
const item = first([1, 2, 3]); // unknown -- must cast

// With generics: preserves type information
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
const item = first([1, 2, 3]); // number | undefined
const str = first(["a", "b"]); // string | undefined

// Multiple type parameters
function zip<A, B>(a: A[], b: B[]): [A, B][] {
  const length = Math.min(a.length, b.length);
  const result: [A, B][] = [];
  for (let i = 0; i < length; i++) {
    result.push([a[i]!, b[i]!]);
  }
  return result;
}

const pairs = zip([1, 2, 3], ["a", "b", "c"]);
// [number, string][]
```

### Generic Arrow Functions

```typescript
// Standard syntax
const identity = <T>(value: T): T => value;

// In .tsx files, the <T> can conflict with JSX. Use extends to disambiguate:
const identity = <T extends unknown>(value: T): T => value;
```

---

## Generic Interfaces

```typescript
// Generic container
interface Box<T> {
  value: T;
  map<U>(fn: (value: T) => U): Box<U>;
}

function createBox<T>(value: T): Box<T> {
  return {
    value,
    map: (fn) => createBox(fn(value)),
  };
}

const numBox = createBox(42);          // Box<number>
const strBox = numBox.map(String);     // Box<string>

// Generic API response
interface ApiResponse<T> {
  data: T;
  meta: {
    page: number;
    total: number;
  };
}

interface User {
  id: string;
  name: string;
}

async function fetchUsers(): Promise<ApiResponse<User[]>> {
  const response = await fetch("/api/users");
  return response.json();
}
```

---

## Generic Classes

```typescript
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  get size(): number {
    return this.items.length;
  }
}

const numberStack = new Stack<number>();
numberStack.push(1);
numberStack.push(2);
const top = numberStack.pop(); // number | undefined
```

---

## Generic Constraints (`extends`)

Constraints limit the types that can be used with a generic, enabling you to access properties of the constrained type.

```typescript
// Without constraint: cannot access .length
function getLength<T>(value: T): number {
  return value.length; // ERROR: Property 'length' does not exist on type 'T'
}

// With constraint: T must have a length property
function getLength<T extends { length: number }>(value: T): number {
  return value.length; // OK
}

getLength("hello");     // 5
getLength([1, 2, 3]);   // 3
getLength(42);           // ERROR: number doesn't have .length

// keyof constraint: restrict to valid property names
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "Alice", age: 30 };
getProperty(user, "name");  // string
getProperty(user, "age");   // number
getProperty(user, "email"); // ERROR: "email" is not assignable to "name" | "age"

// Multiple constraints using intersection
interface Identifiable {
  id: string;
}
interface Timestamped {
  createdAt: Date;
}

function logEntity<T extends Identifiable & Timestamped>(entity: T): void {
  console.log(`Entity ${entity.id} created at ${entity.createdAt}`);
}
```

---

## Default Type Parameters

```typescript
// Default type parameter: T defaults to string if not specified
interface EventPayload<T = string> {
  type: string;
  data: T;
  timestamp: number;
}

const stringEvent: EventPayload = {
  type: "message",
  data: "hello",       // string (default)
  timestamp: Date.now(),
};

const numberEvent: EventPayload<number> = {
  type: "count",
  data: 42,             // number (explicit)
  timestamp: Date.now(),
};

// Defaults with constraints
interface Repository<T extends { id: string } = { id: string }> {
  findById(id: string): Promise<T | null>;
  save(entity: T): Promise<T>;
}
```

---

## Conditional Types

Conditional types select one of two types based on a condition, similar to a ternary expression at the type level.

```typescript
// Basic conditional type
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false

// Extracting element type from arrays
type ElementOf<T> = T extends (infer E)[] ? E : T;

type X = ElementOf<string[]>;  // string
type Y = ElementOf<number>;    // number (not an array, returns T)

// Practical: unwrap Promise types
type Unwrap<T> = T extends Promise<infer U> ? Unwrap<U> : T;

type A = Unwrap<Promise<string>>;           // string
type B = Unwrap<Promise<Promise<number>>>;  // number (recursive)

// Distributive conditional types: unions are distributed automatically
type ToArray<T> = T extends unknown ? T[] : never;

type Result = ToArray<string | number>; // string[] | number[]
// NOT (string | number)[] -- each union member is processed individually

// Prevent distribution with tuple wrapping
type ToArrayNonDist<T> = [T] extends [unknown] ? T[] : never;
type Result2 = ToArrayNonDist<string | number>; // (string | number)[]
```

---

## The `infer` Keyword

`infer` declares a type variable within a conditional type, allowing you to extract types from complex structures.

```typescript
// Extract return type of a function
type MyReturnType<T> = T extends (...args: unknown[]) => infer R ? R : never;

type A = MyReturnType<() => string>;           // string
type B = MyReturnType<(x: number) => boolean>; // boolean

// Extract parameter types
type FirstParam<T> = T extends (first: infer P, ...rest: unknown[]) => unknown ? P : never;

type C = FirstParam<(name: string, age: number) => void>; // string

// Extract from generic types
type UnwrapArray<T> = T extends Array<infer E> ? E : T;
type UnwrapPromise<T> = T extends Promise<infer V> ? V : T;

// Combine: unwrap arrays of promises
type UnwrapAll<T> = T extends Array<Promise<infer V>> ? V : T;

type D = UnwrapAll<Promise<string>[]>; // string

// Extract template literal parts
type ExtractRouteParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractRouteParams<Rest>
    : T extends `${string}:${infer Param}`
      ? Param
      : never;

type Params = ExtractRouteParams<"/users/:userId/posts/:postId">;
// "userId" | "postId"
```

---

## Mapped Types

Mapped types transform the properties of an existing type to produce a new type.

```typescript
// Make all properties optional
type MyPartial<T> = {
  [K in keyof T]?: T[K];
};

// Make all properties required
type MyRequired<T> = {
  [K in keyof T]-?: T[K]; // -? removes the optional modifier
};

// Make all properties readonly
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};

// Practical: create a "form state" type from a model
interface User {
  id: string;
  name: string;
  email: string;
}

// Each field has a value, error, and touched state
type FormState<T> = {
  [K in keyof T]: {
    value: T[K];
    error: string | null;
    touched: boolean;
  };
};

type UserForm = FormState<User>;
// {
//   id: { value: string; error: string | null; touched: boolean };
//   name: { value: string; error: string | null; touched: boolean };
//   email: { value: string; error: string | null; touched: boolean };
// }

// Key remapping with 'as' (TypeScript 4.1+)
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type UserGetters = Getters<User>;
// {
//   getId: () => string;
//   getName: () => string;
//   getEmail: () => string;
// }

// Filter keys by value type
type StringKeys<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};
```

---

## Template Literal Types with Generics

```typescript
// Type-safe event emitter
type EventMap = {
  click: { x: number; y: number };
  focus: { target: string };
  submit: { data: Record<string, unknown> };
};

type EventHandler<T> = (payload: T) => void;

class TypedEmitter<Events extends Record<string, unknown>> {
  private handlers = new Map<string, Function[]>();

  on<K extends keyof Events & string>(
    event: K,
    handler: EventHandler<Events[K]>
  ): void {
    const existing = this.handlers.get(event) ?? [];
    existing.push(handler);
    this.handlers.set(event, existing);
  }

  emit<K extends keyof Events & string>(event: K, payload: Events[K]): void {
    const handlers = this.handlers.get(event) ?? [];
    handlers.forEach((h) => h(payload));
  }
}

const emitter = new TypedEmitter<EventMap>();

emitter.on("click", (payload) => {
  console.log(payload.x, payload.y); // Fully typed
});

emitter.emit("click", { x: 10, y: 20 }); // Type-checked payload
emitter.emit("click", { target: "div" }); // ERROR: missing x, y
```

---

## Common Generic Patterns

### `Result<T, E>`

A discriminated union for operations that can succeed or fail, replacing thrown exceptions with typed returns.

```typescript
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}

function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}

// Usage
function parseJson<T>(raw: string): Result<T, string> {
  try {
    return ok(JSON.parse(raw) as T);
  } catch (e) {
    return err(`Failed to parse JSON: ${String(e)}`);
  }
}

const result = parseJson<{ name: string }>('{"name": "Alice"}');
if (result.ok) {
  console.log(result.value.name); // Fully typed
} else {
  console.error(result.error);     // string
}
```

### `Maybe<T>`

Explicit optional type that forces handling of the absent case.

```typescript
type Maybe<T> = T | null;

function findUser(id: string): Maybe<User> {
  const user = users.get(id);
  return user ?? null;
}

// Forces caller to handle null
const user = findUser("123");
if (user !== null) {
  console.log(user.name); // Narrowed to User
}
```

### `AsyncData<T>`

Represents data that is loaded asynchronously, modeling all possible states.

```typescript
type AsyncData<T, E = Error> =
  | { state: "idle" }
  | { state: "loading" }
  | { state: "success"; data: T }
  | { state: "error"; error: E };

function renderAsync<T>(
  asyncData: AsyncData<T>,
  renderer: {
    idle: () => string;
    loading: () => string;
    success: (data: T) => string;
    error: (error: Error) => string;
  }
): string {
  switch (asyncData.state) {
    case "idle":
      return renderer.idle();
    case "loading":
      return renderer.loading();
    case "success":
      return renderer.success(asyncData.data);
    case "error":
      return renderer.error(asyncData.error);
  }
}
```

---

## Generics in Vue Composables

### Type-Safe Composable with Generic Return

```typescript
import { ref, computed, type Ref, type ComputedRef } from "vue";

interface UseListReturn<T> {
  items: Ref<T[]>;
  count: ComputedRef<number>;
  add: (item: T) => void;
  remove: (predicate: (item: T) => boolean) => void;
  clear: () => void;
}

function useList<T>(initialItems: T[] = []): UseListReturn<T> {
  const items = ref<T[]>([...initialItems]) as Ref<T[]>;

  const count = computed(() => items.value.length);

  function add(item: T) {
    items.value.push(item);
  }

  function remove(predicate: (item: T) => boolean) {
    items.value = items.value.filter((item) => !predicate(item));
  }

  function clear() {
    items.value = [];
  }

  return { items, count, add, remove, clear };
}

// Usage in component
const { items, count, add } = useList<string>(["apple", "banana"]);
add("cherry"); // Type-checked: must be string
```

### Generic Fetch Composable

```typescript
import { ref, type Ref } from "vue";

interface UseFetchReturn<T> {
  data: Ref<T | null>;
  error: Ref<string | null>;
  isLoading: Ref<boolean>;
  execute: () => Promise<void>;
}

function useFetch<T>(
  url: string | Ref<string>,
  parser: (raw: unknown) => T
): UseFetchReturn<T> {
  const data = ref<T | null>(null) as Ref<T | null>;
  const error = ref<string | null>(null);
  const isLoading = ref(false);

  async function execute() {
    isLoading.value = true;
    error.value = null;

    try {
      const resolvedUrl = typeof url === "string" ? url : url.value;
      const response = await fetch(resolvedUrl);

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }

      const json: unknown = await response.json();
      data.value = parser(json);
    } catch (e) {
      error.value = e instanceof Error ? e.message : String(e);
    } finally {
      isLoading.value = false;
    }
  }

  return { data, error, isLoading, execute };
}

// Usage with Zod for runtime validation
import { z } from "zod";

const UserSchema = z.object({ id: z.string(), name: z.string() });
type User = z.infer<typeof UserSchema>;

const { data, error, isLoading, execute } = useFetch<User[]>(
  "/api/users",
  (raw) => z.array(UserSchema).parse(raw)
);
```

---

## Best Practices

### DO

- **DO** constrain generics with `extends` to communicate what capabilities are required
- **DO** let TypeScript infer generic type arguments when possible rather than specifying explicitly
- **DO** use `Result<T, E>` for operations that can fail instead of throwing exceptions
- **DO** use exhaustive checking with `never` in discriminated union handlers
- **DO** name type parameters descriptively when there are multiple (`TKey`, `TValue` or `K`, `V`)
- **DO** use mapped types to derive related types from a single source of truth

### DON'T

- **DON'T** over-genericize: if a function only works with one type, don't make it generic
- **DON'T** use `T extends any` -- it is meaningless; use `T extends unknown` or remove the constraint
- **DON'T** use more than 3-4 type parameters on a single function or type -- simplify the design
- **DON'T** nest conditional types deeper than 2-3 levels -- extract into named utility types
- **DON'T** use generics to replace simple union types where no abstraction is needed
- **DON'T** forget default type parameters when a common case exists

---

## Guidelines

### Essential

- Use generic constraints to express the minimum required shape
- Let type inference do the work: avoid redundant explicit type arguments at call sites
- Use `Result<T, E>` or discriminated unions for error handling in composables
- Test generic functions with multiple concrete types to verify correctness

### Recommended

- Create project-level utility types (`Result`, `Maybe`, `AsyncData`) in a shared types file
- Use mapped types to generate form types, getter types, and API types from domain models
- Use `infer` in conditional types to extract nested type information
- Prefer single-letter names (`T`, `K`, `V`) for simple generics, descriptive names for complex ones

### Advanced

- Use recursive conditional types for deep type transformations (e.g., `DeepReadonly<T>`)
- Use template literal types with generics for type-safe routing or event systems
- Use const type parameters (TypeScript 5.0+) for literal tuple inference
- Implement higher-kinded type patterns using generic interfaces and mapped types

---

## Benefits

- Enables code reuse without sacrificing type safety
- Infers types automatically at call sites, reducing annotation burden
- Catches misuse at compile time through generic constraints
- Enables powerful type transformations via mapped and conditional types
- Reduces boilerplate by deriving related types from a single source
- Provides self-documenting contracts through constrained type parameters

---

## Related

- [typescript-utility-types.md](typescript-utility-types.md) -- Built-in and custom utility types
- [typescript-type-safety.md](typescript-type-safety.md) -- Type guards, narrowing, and runtime validation
