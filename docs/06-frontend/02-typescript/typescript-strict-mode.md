# TypeScript Strict Mode

`keywords: strict, noImplicitAny, strictNullChecks, strictFunctionTypes, tsconfig, type-safety, compiler-options`

## Principle

**Enable all strict compiler options from day one.** Strict mode catches entire categories of bugs at compile time, eliminates implicit `any` leakage, and forces explicit handling of `null` and `undefined`. The cost is paid once during development; the benefit compounds over the lifetime of the codebase.

---

## Strict Mode Compiler Options

TypeScript's `strict` flag is a shorthand that enables a family of individual strict checks. Each option addresses a specific class of type safety issues.

### The `strict` Flag

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "strict": true
    // Equivalent to enabling ALL of the following:
    // "noImplicitAny": true,
    // "strictNullChecks": true,
    // "strictFunctionTypes": true,
    // "strictBindCallApply": true,
    // "strictPropertyInitialization": true,
    // "noImplicitThis": true,
    // "alwaysStrict": true,
    // "useUnknownInCatchVariables": true
  }
}
```

Always use the umbrella `strict: true` rather than individual flags. As TypeScript adds new strict checks in future versions, your project automatically opts in.

### Individual Options Explained

#### `noImplicitAny`

Requires explicit type annotations where TypeScript cannot infer a type. Prevents accidental `any` from silently spreading through the codebase.

```typescript
// ERROR with noImplicitAny: Parameter 'x' implicitly has an 'any' type.
function double(x) {
  return x * 2;
}

// FIXED: Explicit type annotation
function double(x: number): number {
  return x * 2;
}
```

#### `strictNullChecks`

Makes `null` and `undefined` their own distinct types rather than assignable to every type. This is the single most impactful strict option.

```typescript
// Without strictNullChecks: compiles fine, crashes at runtime
function getLength(str: string) {
  return str.length; // Runtime error if str is null
}
getLength(null); // No compile error!

// With strictNullChecks: caught at compile time
function getLength(str: string) {
  return str.length;
}
getLength(null); // ERROR: Argument of type 'null' is not assignable to parameter of type 'string'.

// Explicit nullable types require handling
function getLength(str: string | null): number {
  if (str === null) {
    return 0;
  }
  return str.length; // TypeScript knows str is string here
}
```

#### `strictFunctionTypes`

Enables contravariant checking of function parameter types. Without this, function parameters are checked bivariantly, which is unsound.

```typescript
interface Animal {
  name: string;
}
interface Dog extends Animal {
  breed: string;
}

type AnimalHandler = (animal: Animal) => void;
type DogHandler = (dog: Dog) => void;

const handleDog: DogHandler = (dog) => {
  console.log(dog.breed);
};

// ERROR with strictFunctionTypes: Type 'DogHandler' is not assignable to type 'AnimalHandler'.
const handler: AnimalHandler = handleDog;

// This is correct! Passing a Cat to handleDog would crash at runtime.
```

#### `strictBindCallApply`

Ensures that `bind`, `call`, and `apply` methods on functions are invoked with the correct argument types.

```typescript
function greet(name: string, age: number): string {
  return `Hello ${name}, you are ${age} years old`;
}

// ERROR: Argument of type 'boolean' is not assignable to parameter of type 'number'.
greet.call(undefined, "Alice", true);

// CORRECT
greet.call(undefined, "Alice", 30);
```

#### `strictPropertyInitialization`

Ensures class properties are initialized in the constructor or with a default value. Prevents accessing undefined properties.

```typescript
class User {
  name: string;   // ERROR: Property 'name' has no initializer
  email: string;  // ERROR: Property 'email' has no initializer

  constructor(name: string) {
    this.name = name;
    // Forgot to initialize email!
  }
}

// FIXED: Initialize all properties
class User {
  name: string;
  email: string;

  constructor(name: string, email: string) {
    this.name = name;
    this.email = email;
  }
}

// Or use definite assignment assertion when initialization happens outside constructor
// (e.g., framework lifecycle hooks). Use sparingly.
class Component {
  data!: string; // The ! tells TypeScript "I guarantee this will be assigned"
}
```

#### `noImplicitThis`

Raises an error when `this` has an implicit `any` type.

```typescript
// ERROR: 'this' implicitly has type 'any'
function getName() {
  return this.name;
}

// FIXED: Explicit this parameter
function getName(this: { name: string }): string {
  return this.name;
}
```

#### `alwaysStrict`

Emits `"use strict"` in every output file and parses in strict mode. This enables JavaScript strict mode, separate from TypeScript's strict checks.

#### `useUnknownInCatchVariables`

Types catch clause variables as `unknown` instead of `any`, forcing explicit type checking of caught errors.

```typescript
try {
  riskyOperation();
} catch (error) {
  // With useUnknownInCatchVariables: error is 'unknown'
  if (error instanceof Error) {
    console.log(error.message); // Safe access after type guard
  } else {
    console.log("Unknown error:", String(error));
  }
}
```

---

## Recommended tsconfig.json Configuration

```jsonc
{
  "compilerOptions": {
    // Strict type checking
    "strict": true,

    // Additional strictness beyond the strict flag
    "noUncheckedIndexedAccess": true,    // arr[0] is T | undefined
    "noImplicitReturns": true,           // All code paths must return
    "noFallthroughCasesInSwitch": true,  // Require break in switch
    "noImplicitOverride": true,          // Require override keyword
    "exactOptionalPropertyTypes": true,  // Distinguish undefined from missing

    // Module settings
    "module": "ESNext",
    "moduleResolution": "bundler",
    "isolatedModules": true,
    "verbatimModuleSyntax": true,

    // Output settings
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "skipLibCheck": true,

    // Path resolution
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*.ts", "src/**/*.vue"],
  "exclude": ["node_modules", "dist"]
}
```

### The `noUncheckedIndexedAccess` Option

This is not part of the `strict` flag but is strongly recommended. It makes index access on arrays and records return `T | undefined` instead of `T`.

```typescript
const items: string[] = ["a", "b", "c"];

// Without noUncheckedIndexedAccess
const first: string = items[0];   // OK, but items[99] is undefined at runtime!

// With noUncheckedIndexedAccess
const first: string = items[0];   // ERROR: Type 'string | undefined' is not assignable to type 'string'
const first = items[0];           // Type is string | undefined -- must handle

if (items[0] !== undefined) {
  const first: string = items[0]; // OK, narrowed
}
```

---

## Type Narrowing with `strictNullChecks`

When `strictNullChecks` is enabled, TypeScript requires explicit narrowing before accessing nullable values.

### Truthiness Narrowing

```typescript
function printName(name: string | null) {
  if (name) {
    console.log(name.toUpperCase()); // string (narrowed from string | null)
  }
  // Caution: empty string "" is falsy but may be valid
}
```

### Equality Narrowing

```typescript
function process(value: string | null | undefined) {
  if (value !== null && value !== undefined) {
    console.log(value.trim()); // string
  }

  // Shorthand: != null checks both null and undefined
  if (value != null) {
    console.log(value.trim()); // string
  }
}
```

### `in` Operator Narrowing

```typescript
interface Fish {
  swim: () => void;
}
interface Bird {
  fly: () => void;
}

function move(animal: Fish | Bird) {
  if ("swim" in animal) {
    animal.swim(); // Fish
  } else {
    animal.fly();  // Bird
  }
}
```

### Optional Chaining and Nullish Coalescing

```typescript
interface Config {
  database?: {
    host?: string;
    port?: number;
  };
}

function getDbHost(config: Config): string {
  // Optional chaining (?.) short-circuits to undefined
  // Nullish coalescing (??) provides default for null/undefined only
  return config.database?.host ?? "localhost";
}
```

---

## `@ts-expect-error` vs `@ts-ignore`

When suppressing type errors is unavoidable, always prefer `@ts-expect-error` over `@ts-ignore`.

```typescript
// BAD: @ts-ignore silently suppresses any error (or no error at all)
// @ts-ignore
const value: number = "not a number";

// GOOD: @ts-expect-error fails if there is no error to suppress
// This means it will alert you when the underlying issue is fixed.
// @ts-expect-error -- legacy API returns string, migration tracked in PROJ-1234
const value: number = legacyApi.getValue();
```

**Key difference:** `@ts-expect-error` produces a compiler error if the next line has no type error. This makes it self-cleaning -- when you fix the underlying issue, the suppression comment becomes an error reminding you to remove it.

**Rules for suppression comments:**
- Always include a reason after the directive
- Reference a ticket or issue number when possible
- Never use `@ts-ignore` in new code
- Treat every suppression as tech debt to be resolved

---

## Migrating to Strict Mode Incrementally

For existing projects without strict mode, enable it incrementally to avoid a massive migration.

### Strategy 1: Enable `strict` Globally, Suppress Per-File

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "strict": true
  }
}
```

Then add `// @ts-nocheck` at the top of files that have too many errors, and remove it file by file as you fix them. Track progress with a script:

```bash
# Count remaining @ts-nocheck files
grep -rl "@ts-nocheck" src/ | wc -l
```

### Strategy 2: Enable Individual Flags One at a Time

```jsonc
// Phase 1: Start with the least disruptive options
{
  "compilerOptions": {
    "alwaysStrict": true,
    "strictBindCallApply": true,
    "noImplicitThis": true
  }
}

// Phase 2: Add noImplicitAny (medium disruption)
{
  "compilerOptions": {
    "alwaysStrict": true,
    "strictBindCallApply": true,
    "noImplicitThis": true,
    "noImplicitAny": true
  }
}

// Phase 3: Add strictNullChecks (highest disruption, highest value)
{
  "compilerOptions": {
    "alwaysStrict": true,
    "strictBindCallApply": true,
    "noImplicitThis": true,
    "noImplicitAny": true,
    "strictNullChecks": true
  }
}

// Phase 4: Enable remaining flags, switch to strict: true
{
  "compilerOptions": {
    "strict": true
  }
}
```

### Strategy 3: Strict Per-Directory with Project References

```jsonc
// tsconfig.strict.json -- new code goes here
{
  "compilerOptions": {
    "strict": true
  },
  "include": ["src/new-modules/**/*.ts"]
}

// tsconfig.legacy.json -- existing code
{
  "compilerOptions": {
    "strict": false
  },
  "include": ["src/legacy/**/*.ts"]
}
```

---

## Common Strict Mode Errors and Fixes

### Error: Object is possibly 'undefined'

```typescript
// ERROR
interface User {
  address?: { city: string };
}
function getCity(user: User): string {
  return user.address.city; // ERROR: Object is possibly 'undefined'
}

// FIX 1: Optional chaining with default
function getCity(user: User): string {
  return user.address?.city ?? "Unknown";
}

// FIX 2: Early return guard
function getCity(user: User): string {
  if (!user.address) {
    throw new Error("User has no address");
  }
  return user.address.city; // Narrowed: address is defined
}
```

### Error: Type 'X' is not assignable to type 'Y'

```typescript
// ERROR
function processItems(items: string[]) { /* ... */ }
const mixed = [1, "two", 3]; // inferred as (number | string)[]
processItems(mixed); // ERROR

// FIX: Be explicit about the array type
const strings: string[] = ["one", "two", "three"];
processItems(strings);
```

### Error: Parameter implicitly has an 'any' type

```typescript
// ERROR: Event handlers in Vue/DOM
document.addEventListener("click", (e) => {
  // e is implicitly any in some contexts
});

// FIX: Explicit event type
document.addEventListener("click", (e: MouseEvent) => {
  console.log(e.clientX);
});
```

### Error: Property has no initializer and is not definitely assigned

```typescript
// ERROR in class
class Store {
  items: string[]; // ERROR: no initializer
}

// FIX 1: Initialize with default
class Store {
  items: string[] = [];
}

// FIX 2: Initialize in constructor
class Store {
  items: string[];
  constructor() {
    this.items = [];
  }
}
```

---

## Best Practices

### DO

- **DO** enable `strict: true` in every new project from the start
- **DO** enable `noUncheckedIndexedAccess` alongside `strict`
- **DO** use `@ts-expect-error` with a reason comment when suppression is needed
- **DO** narrow nullable types explicitly rather than using non-null assertion (`!`)
- **DO** treat strict mode errors as real bugs, not inconveniences
- **DO** migrate existing projects incrementally using one of the strategies above

### DON'T

- **DON'T** use `@ts-ignore` in new code -- always use `@ts-expect-error`
- **DON'T** use the non-null assertion operator (`!`) as a shortcut to silence errors
- **DON'T** disable individual strict flags after enabling `strict: true`
- **DON'T** cast to `any` to work around strict errors -- find the correct type
- **DON'T** leave `@ts-expect-error` comments without a reason or ticket reference
- **DON'T** use `as` type assertions when a type guard would be safer

---

## Guidelines

### Essential

- Enable `strict: true` in `tsconfig.json` for all projects
- Enable `noUncheckedIndexedAccess: true` for array/record safety
- Use type narrowing (guards, equality checks, `in` operator) instead of assertions
- Replace all `@ts-ignore` with `@ts-expect-error` plus a reason

### Recommended

- Enable `noImplicitReturns` and `noFallthroughCasesInSwitch` for additional safety
- Enable `exactOptionalPropertyTypes` to distinguish missing from undefined
- Enable `noImplicitOverride` for class method override safety
- Audit and reduce `@ts-expect-error` count regularly

### Advanced

- Use project references for incremental strict migration of large codebases
- Configure ESLint with `@typescript-eslint/strict` to catch patterns strict mode misses
- Track strict mode coverage metrics in CI (count of suppression comments)
- Enforce zero `any` in new code via ESLint `no-explicit-any` rule

---

## Benefits

- Catches null/undefined errors at compile time instead of runtime
- Eliminates implicit `any` leakage across module boundaries
- Forces explicit handling of edge cases and error states
- Makes refactoring safer with stronger type guarantees
- Self-documents code intent through required type annotations
- Reduces production bugs by shifting error detection to build time

---

## Related

- [typescript-type-safety.md](typescript-type-safety.md) -- Type guards, assertions, and runtime validation
- [typescript-vue-integration.md](typescript-vue-integration.md) -- TypeScript configuration for Vue projects
