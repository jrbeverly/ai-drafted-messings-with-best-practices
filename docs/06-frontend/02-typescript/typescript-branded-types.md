# TypeScript Branded Types

Nominal typing in a structural type system. Use branded types to prevent accidental interchange of semantically different values that share the same primitive shape.

`keywords: typescript, branded-types, nominal, opaque, newtype, type-safety, validation, zod, valibot, unique-symbol`

## Principle

TypeScript uses structural typing: two types are compatible if their shapes match. This means a `UserId` and a `BookId` are interchangeable if both are `string`. Branded types add a phantom property to create nominal distinction, making the type system reject accidental substitution of values that happen to share the same underlying type.

## What Are Branded Types

A branded type wraps a primitive with a unique tag that exists only at compile time. The tag has no runtime cost because it is erased during compilation.

```ts
// Without branding: all strings are interchangeable
function getUser(userId: string): User { /* ... */ }
function getBook(bookId: string): Book { /* ... */ }

const userId = 'user-123'
const bookId = 'book-456'

// TypeScript allows this - a bug!
getUser(bookId) // No error, but semantically wrong
getBook(userId) // No error, but semantically wrong
```

```ts
// With branding: each ID type is distinct
type UserId = string & { readonly __brand: unique symbol }
type BookId = string & { readonly __brand: unique symbol }

function getUser(userId: UserId): User { /* ... */ }
function getBook(bookId: BookId): Book { /* ... */ }

const userId = 'user-123' as UserId
const bookId = 'book-456' as BookId

getUser(bookId) // Type error: BookId is not assignable to UserId
getBook(userId) // Type error: UserId is not assignable to BookId
getUser(userId) // OK
getBook(bookId) // OK
```

## Creating Branded Types with Unique Symbol

The most type-safe approach uses `unique symbol` to guarantee uniqueness across the program.

```ts
// types/branded.ts

// Generic brand utility
declare const __brand: unique symbol
type Brand<T, B> = T & { readonly [__brand]: B }

// Define specific branded types
export type UserId = Brand<string, 'UserId'>
export type BookId = Brand<string, 'BookId'>
export type Email = Brand<string, 'Email'>
export type ISBN = Brand<string, 'ISBN'>

export type PositiveNumber = Brand<number, 'PositiveNumber'>
export type Percentage = Brand<number, 'Percentage'>
export type Currency = Brand<number, 'Currency'>
export type Latitude = Brand<number, 'Latitude'>
export type Longitude = Brand<number, 'Longitude'>
```

Alternative approach using interface merging for each type:

```ts
// Each branded type gets its own unique symbol
declare const UserIdBrand: unique symbol
export type UserId = string & { readonly [UserIdBrand]: typeof UserIdBrand }

declare const BookIdBrand: unique symbol
export type BookId = string & { readonly [BookIdBrand]: typeof BookIdBrand }

// This approach guarantees two branded types can never unify,
// even if they use the same base type and brand name
```

## Validation Functions for Branded Types

A branded type is only as trustworthy as the code that creates it. Validation functions act as gatekeepers, ensuring values meet semantic invariants before receiving the brand.

```ts
// validators/branded.ts
import type { UserId, Email, PositiveNumber, Percentage } from '../types/branded'

export function toUserId(value: string): UserId {
  if (!value.startsWith('user-')) {
    throw new Error(`Invalid UserId: must start with "user-", got "${value}"`)
  }
  return value as UserId
}

export function toEmail(value: string): Email {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  if (!emailRegex.test(value)) {
    throw new Error(`Invalid email: "${value}"`)
  }
  return value as Email
}

export function toPositiveNumber(value: number): PositiveNumber {
  if (value <= 0 || !Number.isFinite(value)) {
    throw new Error(`Expected positive number, got ${value}`)
  }
  return value as PositiveNumber
}

export function toPercentage(value: number): Percentage {
  if (value < 0 || value > 100) {
    throw new Error(`Percentage must be 0-100, got ${value}`)
  }
  return value as Percentage
}
```

For cases where validation might fail without throwing:

```ts
import type { Email } from '../types/branded'

export function tryParseEmail(value: string): Email | null {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  if (!emailRegex.test(value)) {
    return null
  }
  return value as Email
}

// Usage
const email = tryParseEmail(userInput)
if (email === null) {
  showError('Please enter a valid email address')
  return
}
// email is typed as Email from here on
sendVerification(email)
```

## Common Use Cases

### Entity Identifiers

```ts
type UserId = Brand<string, 'UserId'>
type BookId = Brand<string, 'BookId'>
type LoanId = Brand<string, 'LoanId'>
type OrganizationId = Brand<string, 'OrganizationId'>

interface Loan {
  readonly id: LoanId
  readonly userId: UserId
  readonly bookId: BookId
  readonly organizationId: OrganizationId
  readonly dueDate: string
}

// The type system prevents mixing up IDs in function calls
function returnBook(loanId: LoanId, userId: UserId): void {
  // ...
}

// Cannot accidentally pass bookId where loanId is expected
const loan: Loan = { /* ... */ }
returnBook(loan.id, loan.userId)     // OK
returnBook(loan.bookId, loan.userId) // Type error
```

### Validated Strings

```ts
type Email = Brand<string, 'Email'>
type PhoneNumber = Brand<string, 'PhoneNumber'>
type URL = Brand<string, 'URL'>
type Slug = Brand<string, 'Slug'>
type HexColor = Brand<string, 'HexColor'>

function sendNotification(email: Email, subject: string): void {
  // Can trust that email is validated - no need to re-validate
  fetch('/api/notify', {
    method: 'POST',
    body: JSON.stringify({ email, subject }),
  })
}
```

### Constrained Numbers

```ts
type PositiveNumber = Brand<number, 'PositiveNumber'>
type NonNegativeInteger = Brand<number, 'NonNegativeInteger'>
type Percentage = Brand<number, 'Percentage'>
type Currency = Brand<number, 'Currency'>

function calculateDiscount(
  price: Currency,
  discount: Percentage
): Currency {
  const result = (price as number) * (1 - (discount as number) / 100)
  return result as Currency
}

// Callers must pass validated values
const price = toCurrency(29.99)
const discount = toPercentage(15)
calculateDiscount(price, discount) // OK
calculateDiscount(15, 29.99)       // Type error: number not assignable to Currency
```

### Units of Measure

```ts
type Meters = Brand<number, 'Meters'>
type Kilometers = Brand<number, 'Kilometers'>
type Celsius = Brand<number, 'Celsius'>
type Fahrenheit = Brand<number, 'Fahrenheit'>

function metersToKilometers(m: Meters): Kilometers {
  return ((m as number) / 1000) as Kilometers
}

function celsiusToFahrenheit(c: Celsius): Fahrenheit {
  return ((c as number) * 9 / 5 + 32) as Fahrenheit
}

// Prevents Mars Climate Orbiter-style unit confusion
const distance = 5000 as Meters
metersToKilometers(distance)         // OK
celsiusToFahrenheit(distance)        // Type error: Meters not assignable to Celsius
```

## Branded Types vs Plain Strings/Numbers

```ts
// WITHOUT branding: type system cannot help
function transferMoney(
  fromAccount: string,
  toAccount: string,
  amount: number,
  fee: number
): void {
  // Easy to swap fromAccount and toAccount by mistake
  // Easy to swap amount and fee by mistake
}

transferMoney('acc-1', 'acc-2', 100, 5)   // Correct
transferMoney('acc-2', 'acc-1', 5, 100)   // Bug: swapped accounts AND amounts
// TypeScript says both calls are fine
```

```ts
// WITH branding: type system catches mistakes
type AccountId = Brand<string, 'AccountId'>
type Currency = Brand<number, 'Currency'>
type Fee = Brand<number, 'Fee'>

function transferMoney(
  fromAccount: AccountId,
  toAccount: AccountId,
  amount: Currency,
  fee: Fee
): void {
  // Parameters are distinguishable by type
}

const from = toAccountId('acc-1')
const to = toAccountId('acc-2')
const amount = toCurrency(100)
const fee = toFee(5)

transferMoney(from, to, amount, fee)   // OK
transferMoney(from, to, fee, amount)   // Type error: Fee not assignable to Currency
```

Branded types add value when:
- Multiple parameters share the same primitive type
- Mixing up parameters would be a silent bug
- Values have semantic meaning beyond their shape (an email is not just any string)
- Values require validation before use

Branded types are unnecessary when:
- The primitive type is used in a single context with no ambiguity
- The overhead of validation functions outweighs the safety benefit
- The value has no domain-specific invariants

## Combining with Zod for Runtime Validation

Zod schemas can produce branded types, connecting runtime validation to compile-time type safety.

```ts
import { z } from 'zod'

// Define schema with brand
const EmailSchema = z
  .string()
  .email('Invalid email format')
  .toLowerCase()
  .brand<'Email'>()

const PositiveNumberSchema = z
  .number()
  .positive('Must be positive')
  .finite('Must be finite')
  .brand<'PositiveNumber'>()

const PercentageSchema = z
  .number()
  .min(0, 'Must be >= 0')
  .max(100, 'Must be <= 100')
  .brand<'Percentage'>()

// Extract the branded type from the schema
type Email = z.infer<typeof EmailSchema>
type PositiveNumber = z.infer<typeof PositiveNumberSchema>
type Percentage = z.infer<typeof PercentageSchema>

// Parse and validate in one step
function processSignup(rawEmail: string, rawAge: number) {
  const email = EmailSchema.parse(rawEmail)       // Email (branded)
  const age = PositiveNumberSchema.parse(rawAge)   // PositiveNumber (branded)

  createUser(email, age) // Fully typed
}

// Safe parse for form validation
function validateForm(data: unknown) {
  const result = EmailSchema.safeParse(data)
  if (!result.success) {
    return { error: result.error.format() }
  }
  return { email: result.data } // result.data is Email (branded)
}
```

## Combining with Valibot for Runtime Validation

Valibot achieves the same branded type pattern with a smaller bundle footprint.

```ts
import * as v from 'valibot'

const EmailSchema = v.pipe(
  v.string(),
  v.email('Invalid email format'),
  v.toLowerCase(),
  v.brand('Email')
)

const PositiveNumberSchema = v.pipe(
  v.number(),
  v.minValue(0, 'Must be positive'),
  v.brand('PositiveNumber')
)

type Email = v.InferOutput<typeof EmailSchema>
type PositiveNumber = v.InferOutput<typeof PositiveNumberSchema>

// Usage
const email = v.parse(EmailSchema, 'user@example.com') // Email (branded)
```

## Branded Types at API Boundaries

Branded types are especially valuable at the boundary between external data and internal code. Validate and brand at the edge, then trust the types throughout the application.

```ts
// api/users.ts
import type { UserId, Email } from '../types/branded'
import { toUserId, toEmail } from '../validators/branded'

interface RawUserResponse {
  id: string
  email: string
  name: string
}

interface User {
  id: UserId
  email: Email
  name: string
}

function parseUserResponse(raw: RawUserResponse): User {
  return {
    id: toUserId(raw.id),
    email: toEmail(raw.email),
    name: raw.name,
  }
}

// All downstream code receives validated, branded types
async function fetchUser(id: UserId): Promise<User> {
  const response = await fetch(`/api/users/${id}`)
  const raw: RawUserResponse = await response.json()
  return parseUserResponse(raw) // Brand at the boundary
}
```

```ts
// Downstream code trusts the brands without re-validating
function sendWelcomeEmail(email: Email): void {
  // No need to validate - the Email type guarantees it was validated
  mailer.send({ to: email, template: 'welcome' })
}

function getUserProfile(userId: UserId): Promise<Profile> {
  // No need to check format - UserId type guarantees it
  return db.get(`USER#${userId}`)
}
```

## Comparison with Discriminated Unions

Branded types and discriminated unions solve different problems:

```ts
// Branded type: same shape, different semantics
type UserId = Brand<string, 'UserId'>    // Still a string at runtime
type BookId = Brand<string, 'BookId'>    // Still a string at runtime

// Discriminated union: different shapes, different data
type Result<T> =
  | { readonly status: 'success'; readonly data: T }
  | { readonly status: 'error'; readonly error: Error }
```

| Aspect | Branded Types | Discriminated Unions |
|--------|--------------|---------------------|
| Runtime cost | Zero (erased at compile time) | Carries discriminant property |
| Use case | Distinguish same-shaped values | Model distinct states/variants |
| Pattern | Wraps primitives with phantom tag | Object with literal discriminant |
| Narrowing | N/A (one shape) | Switch/if on discriminant |
| When to use | IDs, validated strings, units | States, results, polymorphic data |

Use branded types when the underlying data is identical but the meaning differs. Use discriminated unions when the data itself varies by variant.

## Best Practices

**DO:**
- Create branded types for entity identifiers (UserId, BookId, OrderId)
- Validate at system boundaries and brand the result
- Use the `Brand<T, B>` utility pattern for consistency
- Provide both throwing (`toEmail`) and safe (`tryParseEmail`) constructors
- Combine with Zod or Valibot for integrated runtime + compile-time validation
- Document what invariants a brand guarantees
- Use branded numbers for domain-specific units (Currency, Percentage, Meters)

**DON'T:**
- Brand every primitive in the codebase (focus on high-risk interchange points)
- Use `as` casts to bypass validation (always go through constructor functions)
- Expose the brand tag in public APIs (it is an implementation detail)
- Create branded types for values used in a single, unambiguous context
- Forget to handle the unbranded input at system boundaries
- Mix branded and unbranded versions of the same concept in the same module

## Guidelines

**Essential:**
- Define a `Brand<T, B>` utility type used consistently across the codebase
- Provide validation constructor functions for every branded type
- Brand external input at API boundaries, trust brands internally
- Use branded IDs to prevent entity ID confusion in functions with multiple ID parameters

**Recommended:**
- Integrate with Zod or Valibot schemas using `.brand()` for unified validation
- Provide both throwing and safe (returns `null`) parse variants
- Use branded types for all entity identifiers in domain models
- Keep branded type definitions in a central `types/branded.ts` file

**Advanced:**
- Branded types for units of measure to prevent unit confusion
- Composable brands (a value can carry multiple brands via intersection)
- Type-level arithmetic constraints (NonNegativeInteger, BoundedNumber)
- Branded types in serialization/deserialization boundaries (JSON parse layer)

## Benefits

Compile-time safety. The type system prevents mixing up semantically different values.

Zero runtime cost. Brand tags are erased during compilation with no performance impact.

Self-documenting. A function accepting `Email` communicates that validation has occurred.

Validated by construction. Once branded, downstream code trusts the value without re-validating.

Reduced bug surface. Eliminates an entire category of "wrong parameter order" bugs.

IDE support. Autocompletion distinguishes between `UserId` and `BookId` instead of showing `string` for both.

## Related

- [typescript-type-safety.md](./typescript-type-safety.md) - Broader type safety patterns and practices
- [typescript-discriminated-unions.md](./typescript-discriminated-unions.md) - Discriminated unions for modeling variant data
