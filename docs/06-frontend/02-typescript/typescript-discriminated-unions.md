# TypeScript Discriminated Unions

Model distinct states and variants with discriminated unions. Type narrowing, exhaustive checking, and patterns for state machines, API responses, and error hierarchies.

`keywords: typescript, discriminated-union, tagged-union, type-narrowing, exhaustive-check, switch, pattern-matching, state-machine, result-type, never`

## Principle

A discriminated union uses a shared literal property (the discriminant) to distinguish between variants. TypeScript narrows the type automatically when you check the discriminant, giving each branch access to only the properties that exist for that variant. This eliminates invalid states from the type system entirely.

## What Are Discriminated Unions

A discriminated union is a union of object types where each member has a common property with a unique literal value. TypeScript uses that property to narrow the union to a specific member.

```ts
// Each variant has a 'status' discriminant with a unique literal value
type RequestState<T> =
  | { readonly status: 'idle' }
  | { readonly status: 'loading' }
  | { readonly status: 'success'; readonly data: T }
  | { readonly status: 'error'; readonly error: Error }

// TypeScript narrows based on the discriminant
function renderState(state: RequestState<User>) {
  switch (state.status) {
    case 'idle':
      // state is { status: 'idle' }
      return 'Ready to fetch'
    case 'loading':
      // state is { status: 'loading' }
      return 'Loading...'
    case 'success':
      // state is { status: 'success'; data: User }
      return `Hello, ${state.data.name}`
    case 'error':
      // state is { status: 'error'; error: Error }
      return `Error: ${state.error.message}`
  }
}
```

Without discriminated unions, you would use nullable fields and boolean flags, leading to impossible states that the type system cannot prevent:

```ts
// BAD: Impossible states are representable
interface RequestState<T> {
  isLoading: boolean
  data: T | null
  error: Error | null
}

// This is valid but nonsensical: loading with data and an error
const broken: RequestState<User> = {
  isLoading: true,
  data: someUser,
  error: new Error('what'),
}
```

## Type Narrowing with switch and if

### switch Statements

The most common way to narrow discriminated unions. Each `case` narrows the type to the matching variant.

```ts
type Shape =
  | { readonly kind: 'circle'; readonly radius: number }
  | { readonly kind: 'rectangle'; readonly width: number; readonly height: number }
  | { readonly kind: 'triangle'; readonly base: number; readonly height: number }

function area(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':
      return Math.PI * shape.radius ** 2
    case 'rectangle':
      return shape.width * shape.height
    case 'triangle':
      return (shape.base * shape.height) / 2
  }
}
```

### if/else Chains

Useful when the narrowing logic involves more than equality checks.

```ts
type ApiResult =
  | { readonly ok: true; readonly data: unknown }
  | { readonly ok: false; readonly error: string; readonly statusCode: number }

function handleResult(result: ApiResult) {
  if (result.ok) {
    // result is { ok: true; data: unknown }
    processData(result.data)
  } else {
    // result is { ok: false; error: string; statusCode: number }
    if (result.statusCode === 404) {
      showNotFound()
    } else {
      showError(result.error)
    }
  }
}
```

### in Operator Narrowing

TypeScript narrows based on the presence of a property using `in`.

```ts
type Event =
  | { readonly type: 'click'; readonly x: number; readonly y: number }
  | { readonly type: 'keypress'; readonly key: string }
  | { readonly type: 'scroll'; readonly scrollTop: number }

function logEvent(event: Event) {
  if ('key' in event) {
    // event is { type: 'keypress'; key: string }
    console.log(`Key pressed: ${event.key}`)
  }
}
```

## Exhaustive Checking

Exhaustive checking ensures every variant is handled. If a new variant is added to the union, TypeScript reports an error at every switch/if that does not handle it.

```ts
// The exhaustive check helper
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(value)}`)
}

type PaymentMethod =
  | { readonly type: 'credit_card'; readonly last4: string }
  | { readonly type: 'bank_transfer'; readonly bankName: string }
  | { readonly type: 'paypal'; readonly email: string }

function describePayment(method: PaymentMethod): string {
  switch (method.type) {
    case 'credit_card':
      return `Credit card ending in ${method.last4}`
    case 'bank_transfer':
      return `Bank transfer via ${method.bankName}`
    case 'paypal':
      return `PayPal (${method.email})`
    default:
      // If a new variant is added to PaymentMethod,
      // this line will produce a compile error:
      // "Argument of type '{ type: "new_method"; ... }' is not assignable to parameter of type 'never'"
      return assertNever(method)
  }
}
```

Without the `assertNever` default case, adding a new variant to the union compiles silently, and the new variant falls through unhandled at runtime.

An alternative using `satisfies never`:

```ts
function describePayment(method: PaymentMethod): string {
  switch (method.type) {
    case 'credit_card':
      return `Credit card ending in ${method.last4}`
    case 'bank_transfer':
      return `Bank transfer via ${method.bankName}`
    case 'paypal':
      return `PayPal (${method.email})`
    default: {
      const _exhaustive: never = method
      throw new Error(`Unhandled payment method: ${JSON.stringify(_exhaustive)}`)
    }
  }
}
```

## Common Patterns

### Result Type

Represent success or failure without exceptions.

```ts
type Result<T, E = Error> =
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: E }

// Constructor helpers
function ok<T>(value: T): Result<T, never> {
  return { ok: true, value }
}

function err<E>(error: E): Result<never, E> {
  return { ok: false, error }
}

// Usage
function parseJSON(input: string): Result<unknown, string> {
  try {
    return ok(JSON.parse(input))
  } catch {
    return err(`Invalid JSON: ${input.slice(0, 50)}`)
  }
}

const result = parseJSON('{"name": "Alice"}')
if (result.ok) {
  console.log(result.value) // unknown
} else {
  console.error(result.error) // string
}
```

### RemoteData

Model the full lifecycle of an async data fetch.

```ts
type RemoteData<T, E = Error> =
  | { readonly status: 'not-asked' }
  | { readonly status: 'loading' }
  | { readonly status: 'success'; readonly data: T }
  | { readonly status: 'failure'; readonly error: E }

// Helper constructors
const RemoteData = {
  notAsked: <T, E = Error>(): RemoteData<T, E> => ({ status: 'not-asked' }),
  loading: <T, E = Error>(): RemoteData<T, E> => ({ status: 'loading' }),
  success: <T, E = Error>(data: T): RemoteData<T, E> => ({ status: 'success', data }),
  failure: <T, E = Error>(error: E): RemoteData<T, E> => ({ status: 'failure', error }),
} as const

// Usage in a Vue composable
function useRemoteData<T>(fetchFn: () => Promise<T>) {
  const state = ref<RemoteData<T>>(RemoteData.notAsked())

  async function execute() {
    state.value = RemoteData.loading()
    try {
      const data = await fetchFn()
      state.value = RemoteData.success(data)
    } catch (e) {
      state.value = RemoteData.failure(e instanceof Error ? e : new Error(String(e)))
    }
  }

  return { state, execute }
}
```

### FormState

Model form submission lifecycle.

```ts
type FormState<T> =
  | { readonly status: 'editing'; readonly values: T; readonly errors: Partial<Record<keyof T, string>> }
  | { readonly status: 'submitting'; readonly values: T }
  | { readonly status: 'submitted'; readonly values: T }
  | { readonly status: 'failed'; readonly values: T; readonly error: string }

interface LoginForm {
  email: string
  password: string
}

function handleFormState(state: FormState<LoginForm>) {
  switch (state.status) {
    case 'editing':
      // Can display validation errors
      if (state.errors.email) {
        showFieldError('email', state.errors.email)
      }
      break
    case 'submitting':
      // Disable form, show spinner
      disableForm()
      break
    case 'submitted':
      // Show success, redirect
      redirect('/dashboard')
      break
    case 'failed':
      // Show error, allow retry
      showError(state.error)
      break
  }
}
```

## Discriminated Unions for State Machines

State machines map naturally to discriminated unions. Each state is a variant, and transitions are functions that accept one variant and return another.

```ts
type OrderState =
  | { readonly status: 'draft'; readonly items: readonly OrderItem[] }
  | { readonly status: 'placed'; readonly items: readonly OrderItem[]; readonly placedAt: Date }
  | { readonly status: 'paid'; readonly items: readonly OrderItem[]; readonly placedAt: Date; readonly paidAt: Date; readonly paymentId: string }
  | { readonly status: 'shipped'; readonly items: readonly OrderItem[]; readonly placedAt: Date; readonly paidAt: Date; readonly paymentId: string; readonly trackingNumber: string }
  | { readonly status: 'delivered'; readonly items: readonly OrderItem[]; readonly deliveredAt: Date }
  | { readonly status: 'cancelled'; readonly reason: string; readonly cancelledAt: Date }

// Transitions are typed: you can only ship a paid order
function shipOrder(
  order: Extract<OrderState, { status: 'paid' }>,
  trackingNumber: string
): Extract<OrderState, { status: 'shipped' }> {
  return {
    ...order,
    status: 'shipped',
    trackingNumber,
  }
}

// You cannot ship a draft order - TypeScript prevents it
function processOrder(order: OrderState) {
  if (order.status === 'paid') {
    const shipped = shipOrder(order, 'TRACK-123') // OK
  }

  if (order.status === 'draft') {
    // shipOrder(order, 'TRACK-123') // Type error: 'draft' not assignable to 'paid'
  }
}
```

`Extract<Union, Shape>` pulls out the variant(s) matching a shape, which is useful for typing transition functions.

## API Response Modeling

Model different API response shapes as discriminated unions.

```ts
type ApiResponse<T> =
  | { readonly status: 200; readonly data: T }
  | { readonly status: 201; readonly data: T; readonly location: string }
  | { readonly status: 400; readonly errors: readonly ValidationError[] }
  | { readonly status: 401; readonly message: string }
  | { readonly status: 403; readonly message: string; readonly requiredPermission: string }
  | { readonly status: 404; readonly message: string }
  | { readonly status: 500; readonly message: string; readonly traceId: string }

interface ValidationError {
  readonly field: string
  readonly message: string
}

async function handleResponse<T>(response: ApiResponse<T>): Promise<T> {
  switch (response.status) {
    case 200:
    case 201:
      return response.data
    case 400:
      throw new ValidationException(response.errors)
    case 401:
      redirectToLogin()
      throw new Error('Unauthorized')
    case 403:
      throw new ForbiddenError(response.requiredPermission)
    case 404:
      throw new NotFoundError(response.message)
    case 500:
      reportError(response.traceId)
      throw new ServerError(response.message)
    default:
      return assertNever(response)
  }
}
```

## Error Type Hierarchies

Model domain errors as discriminated unions instead of exception class hierarchies.

```ts
type AppError =
  | { readonly type: 'validation'; readonly field: string; readonly message: string }
  | { readonly type: 'not_found'; readonly resource: string; readonly id: string }
  | { readonly type: 'unauthorized'; readonly reason: string }
  | { readonly type: 'rate_limited'; readonly retryAfterMs: number }
  | { readonly type: 'network'; readonly originalError: Error }

function userFacingMessage(error: AppError): string {
  switch (error.type) {
    case 'validation':
      return `${error.field}: ${error.message}`
    case 'not_found':
      return `${error.resource} not found`
    case 'unauthorized':
      return 'You do not have permission to perform this action'
    case 'rate_limited':
      return `Too many requests. Try again in ${Math.ceil(error.retryAfterMs / 1000)} seconds`
    case 'network':
      return 'Network error. Check your connection and try again'
    default:
      return assertNever(error)
  }
}

function isRetryable(error: AppError): boolean {
  return error.type === 'network' || error.type === 'rate_limited'
}
```

## Discriminated Unions vs Class Hierarchies

```ts
// CLASS HIERARCHY: requires instanceof checks, harder to serialize
class AppError extends Error {
  constructor(message: string) { super(message) }
}

class ValidationError extends AppError {
  constructor(readonly field: string, message: string) { super(message) }
}

class NotFoundError extends AppError {
  constructor(readonly resource: string, readonly id: string) {
    super(`${resource} ${id} not found`)
  }
}

// Must use instanceof (fragile across module boundaries)
if (error instanceof ValidationError) {
  console.log(error.field)
}
```

```ts
// DISCRIMINATED UNION: plain objects, easy to serialize, exhaustive checking
type AppError =
  | { readonly type: 'validation'; readonly field: string; readonly message: string }
  | { readonly type: 'not_found'; readonly resource: string; readonly id: string }

// Uses property checks (works across module boundaries, serializable)
if (error.type === 'validation') {
  console.log(error.field)
}
```

| Aspect | Class Hierarchies | Discriminated Unions |
|--------|------------------|---------------------|
| Narrowing | `instanceof` (fragile) | Property check (robust) |
| Serialization | Manual `toJSON` needed | Plain objects, JSON-ready |
| Exhaustive check | Not possible | `assertNever` in default |
| Extension | Open (anyone can subclass) | Closed (union is fixed) |
| Best for | When behavior varies per type | When data varies per type |

Prefer discriminated unions for data modeling. Use classes only when each variant needs genuinely different method implementations.

## Pattern Matching with Discriminated Unions

TypeScript does not have built-in pattern matching, but you can build a simple matcher utility.

```ts
type Matcher<U extends { readonly type: string }, R> = {
  [K in U['type']]: (variant: Extract<U, { type: K }>) => R
}

function match<U extends { readonly type: string }, R>(
  value: U,
  handlers: Matcher<U, R>
): R {
  const handler = handlers[value.type as U['type']]
  return handler(value as Extract<U, { type: U['type'] }>)
}

// Usage
type Notification =
  | { readonly type: 'email'; readonly to: string; readonly subject: string }
  | { readonly type: 'sms'; readonly phone: string; readonly body: string }
  | { readonly type: 'push'; readonly deviceId: string; readonly title: string }

function describeNotification(notification: Notification): string {
  return match(notification, {
    email: (n) => `Email to ${n.to}: ${n.subject}`,
    sms: (n) => `SMS to ${n.phone}: ${n.body}`,
    push: (n) => `Push to ${n.deviceId}: ${n.title}`,
  })
}
```

The `Matcher` type ensures every variant has a handler. Adding a new variant to `Notification` causes a compile error in every `match` call that does not handle it.

## Combining Discriminated Unions with Generics

```ts
type AsyncState<T> =
  | { readonly status: 'idle' }
  | { readonly status: 'pending' }
  | { readonly status: 'resolved'; readonly value: T }
  | { readonly status: 'rejected'; readonly error: Error }

// Generic function that works with any AsyncState
function mapAsyncState<T, U>(
  state: AsyncState<T>,
  fn: (value: T) => U
): AsyncState<U> {
  if (state.status === 'resolved') {
    return { status: 'resolved', value: fn(state.value) }
  }
  return state // idle, pending, and rejected don't contain T
}

// Usage
const userState: AsyncState<User> = { status: 'resolved', value: someUser }
const nameState: AsyncState<string> = mapAsyncState(userState, (u) => u.name)
```

## Best Practices

**DO:**
- Use a consistent discriminant property name across related unions (`type`, `status`, `kind`)
- Add an `assertNever` or `satisfies never` default case in every switch for exhaustive checking
- Use `readonly` on all properties in union variants
- Use `Extract<Union, Shape>` to reference specific variants in function signatures
- Model state machines as discriminated unions with typed transitions
- Prefer discriminated unions over class hierarchies for data modeling
- Keep variant names as string literals for serialization and logging

**DON'T:**
- Use boolean flags or nullable fields to represent states (use explicit variants instead)
- Forget the exhaustive check in switch statements
- Use `instanceof` when a discriminant property check works
- Create deeply nested unions (flatten to a single level of discrimination)
- Mix discriminant property names within a single union (`type` on some, `kind` on others)
- Use discriminated unions when a simple enum suffices (no per-variant data needed)
- Omit the discriminant property default in switch (lose exhaustive checking)

## Guidelines

**Essential:**
- Every discriminated union switch has an exhaustive `assertNever` default case
- Discriminant property uses string literal types, not `string`
- All variants in a union use the same discriminant property name
- Variants use `readonly` properties

**Recommended:**
- `Result<T, E>` type for operations that can fail
- `RemoteData<T>` pattern for async data lifecycle
- Error hierarchies modeled as discriminated unions, not class hierarchies
- `Extract<Union, Shape>` for referencing specific variants in function types

**Advanced:**
- Generic `match` function for pattern matching across unions
- State machine transitions typed with `Extract` to constrain valid input states
- `mapAsyncState` and similar generic transformers over union types
- Recursive discriminated unions for tree structures (ASTs, file systems)

## Benefits

Impossible states eliminated. The type system prevents combinations of fields that do not make sense together.

Exhaustive checking. Adding a new variant surfaces every location that needs to handle it.

Self-documenting. Each variant explicitly names the state and its associated data.

Serializable. Plain objects with string discriminants survive JSON serialization without loss.

Safe narrowing. TypeScript automatically narrows the type inside each branch.

Composable. Generic functions like `map`, `match`, and `fold` work across all discriminated unions.

## Related

- [typescript-type-safety.md](./typescript-type-safety.md) - Broader type safety patterns and practices
- [typescript-branded-types.md](./typescript-branded-types.md) - Nominal typing for same-shaped values
- [typescript-generics.md](./typescript-generics.md) - Generic type patterns used with discriminated unions
