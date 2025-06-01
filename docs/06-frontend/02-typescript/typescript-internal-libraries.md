# TypeScript Internal Libraries

Building reusable TypeScript utility libraries for internal consumption. Project structure, API design, packaging, and testing patterns for shared code within an organization.

`keywords: typescript, library, internal, reusable, tsconfig, tsdoc, tree-shaking, private-registry, peer-dependencies, versioning`

## Principle

Internal libraries extract proven, shared logic into independently versioned and tested packages. Build a library only when the same code is used by two or more services. Design for tree-shakeability, explicit APIs, and minimal dependency surfaces so consumers pay only for what they use.

## When to Create an Internal Library

A shared library is justified when code meets all of these criteria:

- Used by two or more services or applications (actual usage, not hypothetical)
- Logic is truly generic and does not encode service-specific assumptions
- The abstraction has stabilized and is unlikely to change shape frequently

If only one service uses the code, keep it in that service. Premature extraction creates coordination overhead without delivering reuse benefits.

## Project Structure for Internal Libraries

A well-structured internal library separates source, tests, configuration, and build output.

```
packages/date-utils/
├── src/
│   ├── index.ts              # Public API barrel export
│   ├── format.ts             # Date formatting utilities
│   ├── parse.ts              # Date parsing utilities
│   ├── relative.ts           # Relative time calculations
│   └── types.ts              # Shared type definitions
├── tests/
│   ├── format.test.ts
│   ├── parse.test.ts
│   └── relative.test.ts
├── package.json
├── tsconfig.json
├── tsconfig.build.json
├── vitest.config.ts
└── README.md
```

The barrel export in `index.ts` defines the public API surface:

```ts
// src/index.ts - Only export what consumers should use
export { formatDate, formatDateTime, formatRelative } from './format'
export { parseISO, parseLocalDate } from './parse'
export { timeAgo, timeUntil } from './relative'
export type { DateFormatOptions, ParseOptions } from './types'
```

Internal helpers that are not part of the public API stay unexported:

```ts
// src/format.ts
import type { DateFormatOptions } from './types'

// Public: exported from index.ts
export function formatDate(date: Date, options?: DateFormatOptions): string {
  const normalized = normalizeDate(date)
  return applyFormat(normalized, options?.locale ?? 'en-US', options?.style ?? 'medium')
}

// Internal: not exported from index.ts
function normalizeDate(date: Date): Date {
  return new Date(date.getFullYear(), date.getMonth(), date.getDate())
}

function applyFormat(date: Date, locale: string, style: string): string {
  return new Intl.DateTimeFormat(locale, { dateStyle: style as 'medium' }).format(date)
}
```

## tsconfig for Libraries

Library tsconfig differs from application tsconfig. Libraries emit JavaScript and declaration files. Applications typically do not emit (bundlers handle it).

```json
// tsconfig.json - Used for type checking and IDE support
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022"],
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*.ts"],
  "exclude": ["src/**/*.test.ts", "src/**/*.spec.ts", "dist"]
}
```

```json
// tsconfig.build.json - Extends base, used only for build
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "declaration": true,
    "declarationDir": "./dist/types",
    "outDir": "./dist"
  },
  "include": ["src/**/*.ts"],
  "exclude": ["src/**/*.test.ts", "src/**/*.spec.ts"]
}
```

## Package.json Configuration

```json
{
  "name": "@myorg/date-utils",
  "version": "1.2.0",
  "description": "Date formatting and parsing utilities",
  "type": "module",
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/types/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/types/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    }
  },
  "files": [
    "dist",
    "README.md"
  ],
  "sideEffects": false,
  "scripts": {
    "build": "tsup src/index.ts --format esm,cjs --dts",
    "dev": "tsup src/index.ts --format esm,cjs --dts --watch",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint src/",
    "typecheck": "tsc --noEmit",
    "prepublishOnly": "npm run build"
  },
  "devDependencies": {
    "tsup": "^8.0.0",
    "typescript": "^5.3.0",
    "vitest": "^1.0.0"
  }
}
```

The `sideEffects: false` field tells bundlers the library is safe to tree-shake. Every export that is not imported by a consumer can be eliminated from the final bundle.

## Tree-Shakeable Library Design

Tree shaking eliminates unused code from the final bundle. Libraries must follow specific patterns to be tree-shakeable.

```ts
// DO: Named exports from separate files
// Each function in its own file or logically grouped

// src/format.ts
export function formatDate(date: Date): string { /* ... */ }
export function formatDateTime(date: Date): string { /* ... */ }

// src/parse.ts
export function parseISO(input: string): Date { /* ... */ }
```

```ts
// DON'T: Default export of an object containing all functions
// This defeats tree shaking because the entire object is referenced

export default {
  formatDate(date: Date): string { /* ... */ },
  formatDateTime(date: Date): string { /* ... */ },
  parseISO(input: string): Date { /* ... */ },
}
```

```ts
// DON'T: Side effects at the module level
// Module-level side effects prevent tree shaking

let instanceCount = 0 // Side effect: mutable module state

export function createParser() {
  instanceCount++ // References module-level state
  return { /* ... */ }
}

// Bundlers cannot remove createParser because instanceCount
// might be observed by other code
```

```ts
// DO: Pure functions with no side effects
export function createParser(options: ParserOptions): Parser {
  return {
    parse(input: string): Date {
      return new Date(input)
    },
  }
}
```

## API Design for Internal Consumption

Design APIs to be discoverable, composable, and hard to misuse.

```ts
// src/types.ts
export interface DateFormatOptions {
  /** Locale for formatting. Defaults to 'en-US'. */
  readonly locale?: string
  /** Date style: 'short' (1/1/24), 'medium' (Jan 1, 2024), 'long' (January 1, 2024). */
  readonly style?: 'short' | 'medium' | 'long'
  /** Whether to include the time component. Defaults to false. */
  readonly includeTime?: boolean
}

export interface ParseOptions {
  /** Assume dates without timezone info are in this timezone. Defaults to 'UTC'. */
  readonly timezone?: string
  /** Whether to throw on invalid input. Defaults to false (returns null). */
  readonly strict?: boolean
}
```

```ts
// src/format.ts
import type { DateFormatOptions } from './types'

/**
 * Format a Date object into a human-readable string.
 *
 * @param date - The date to format
 * @param options - Formatting options
 * @returns Formatted date string
 *
 * @example
 * ```ts
 * formatDate(new Date('2024-01-15'))
 * // => "Jan 15, 2024"
 *
 * formatDate(new Date('2024-01-15'), { style: 'long', locale: 'de-DE' })
 * // => "15. Januar 2024"
 * ```
 */
export function formatDate(date: Date, options?: DateFormatOptions): string {
  const {
    locale = 'en-US',
    style = 'medium',
    includeTime = false,
  } = options ?? {}

  const formatOptions: Intl.DateTimeFormatOptions = {
    dateStyle: style,
    ...(includeTime && { timeStyle: 'short' }),
  }

  return new Intl.DateTimeFormat(locale, formatOptions).format(date)
}
```

API design principles:
- Use options objects instead of positional parameters for functions with more than two arguments
- Make options readonly to signal immutability
- Provide sensible defaults so the simplest call does the right thing
- Use union types for constrained string arguments instead of bare `string`
- Return `null` or `Result` types rather than throwing for expected failures

## Documentation with TSDoc

TSDoc comments are consumed by IDEs and documentation generators. Write them for every public export.

```ts
/**
 * Calculate the relative time between a date and now.
 *
 * @remarks
 * Uses `Intl.RelativeTimeFormat` for localized output. Falls back to
 * English for unsupported locales.
 *
 * @param date - The date to compare against the current time
 * @param options - Optional configuration
 * @returns A human-readable relative time string like "2 hours ago"
 *
 * @example
 * ```ts
 * // Two hours ago
 * const past = new Date(Date.now() - 2 * 60 * 60 * 1000)
 * timeAgo(past) // => "2 hours ago"
 *
 * // In 3 days
 * const future = new Date(Date.now() + 3 * 24 * 60 * 60 * 1000)
 * timeAgo(future) // => "in 3 days"
 * ```
 *
 * @see {@link formatDate} for absolute date formatting
 *
 * @public
 */
export function timeAgo(date: Date, options?: { locale?: string }): string {
  // implementation
}
```

Key TSDoc tags:
- `@param` - Document each parameter
- `@returns` - Describe the return value
- `@example` - Provide usage examples (in fenced code blocks)
- `@remarks` - Additional details beyond the summary line
- `@see` - Cross-reference related functions
- `@public` / `@internal` - Visibility markers for API reports
- `@throws` - Document exceptions the function may throw
- `@deprecated` - Mark functions scheduled for removal

## Peer Dependencies Management

When a library depends on a framework or runtime that the consuming application also provides, use peer dependencies to avoid version conflicts and duplicate bundles.

```json
{
  "name": "@myorg/vue-data-table",
  "peerDependencies": {
    "vue": "^3.4.0"
  },
  "peerDependenciesMeta": {
    "vue": {
      "optional": false
    }
  },
  "devDependencies": {
    "vue": "^3.4.0"
  }
}
```

When to use peer dependencies:
- Framework packages (`vue`, `react`) that must be a single instance
- Plugin host libraries (`@tanstack/vue-query`) that share context
- Packages where version mismatches cause runtime errors

When to use regular dependencies:
- Utility libraries with no shared state (`lodash-es`, `date-fns`)
- Packages where multiple versions can coexist without conflict

## Versioning Strategy

Follow semantic versioning for internal libraries to communicate the impact of changes.

```
MAJOR.MINOR.PATCH
  │      │     │
  │      │     └── Bug fixes, internal refactors (no API change)
  │      └──────── New features, new exports (backward compatible)
  └─────────────── Breaking changes (removed exports, changed signatures)
```

```ts
// PATCH: Fix a bug without changing the API
// Before: formatDate returned wrong month for December
// After: formatDate returns correct month
// 1.2.0 → 1.2.1

// MINOR: Add a new export
// Added: formatRelative function
// 1.2.1 → 1.3.0

// MAJOR: Change an existing function signature
// Before: formatDate(date: Date, locale?: string)
// After: formatDate(date: Date, options?: DateFormatOptions)
// 1.3.0 → 2.0.0
```

For internal libraries used via monorepo workspaces (not published), versioning is optional since consumers always get the latest build. Document breaking changes in the library's changelog regardless.

## Publishing to Private Registry

For organizations that publish internal packages:

```bash
# .npmrc - Configure private registry
@myorg:registry=https://npm.myorg.com/
//npm.myorg.com/:_authToken=${NPM_TOKEN}
```

```json
// package.json
{
  "name": "@myorg/date-utils",
  "publishConfig": {
    "registry": "https://npm.myorg.com/",
    "access": "restricted"
  }
}
```

Build and publish workflow:

```bash
# Build, test, then publish
npm run build
npm run test
npm run typecheck
npm publish
```

## Testing Internal Libraries

Test the public API surface. Internal helpers are tested indirectly through the public functions they support.

```ts
// tests/format.test.ts
import { describe, it, expect } from 'vitest'
import { formatDate, formatDateTime } from '../src'

describe('formatDate', () => {
  it('formats with default options', () => {
    const date = new Date('2024-01-15T00:00:00Z')
    expect(formatDate(date)).toBe('Jan 15, 2024')
  })

  it('respects locale option', () => {
    const date = new Date('2024-01-15T00:00:00Z')
    expect(formatDate(date, { locale: 'de-DE', style: 'long' }))
      .toBe('15. Januar 2024')
  })

  it('uses short style when specified', () => {
    const date = new Date('2024-01-15T00:00:00Z')
    expect(formatDate(date, { style: 'short' })).toBe('1/15/24')
  })

  it('handles edge case: epoch date', () => {
    const date = new Date(0)
    expect(formatDate(date)).toBeTruthy()
  })

  it('handles edge case: far future date', () => {
    const date = new Date('2099-12-31')
    expect(formatDate(date)).toBe('Dec 31, 2099')
  })
})

describe('formatDateTime', () => {
  it('includes time component', () => {
    const date = new Date('2024-01-15T14:30:00Z')
    const result = formatDateTime(date)
    expect(result).toContain('2024')
    expect(result).toMatch(/\d{1,2}:\d{2}/)
  })
})
```

```ts
// tests/types.test.ts - Verify types compile correctly
import { describe, it, expectTypeOf } from 'vitest'
import { formatDate } from '../src'
import type { DateFormatOptions } from '../src'

describe('type safety', () => {
  it('accepts valid options', () => {
    expectTypeOf(formatDate).toBeCallableWith(new Date())
    expectTypeOf(formatDate).toBeCallableWith(new Date(), { locale: 'en-US' })
  })

  it('returns a string', () => {
    expectTypeOf(formatDate(new Date())).toBeString()
  })

  it('enforces option types', () => {
    const options: DateFormatOptions = {
      style: 'medium',
      locale: 'en-US',
    }
    expectTypeOf(options.style).toEqualTypeOf<'short' | 'medium' | 'long' | undefined>()
  })
})
```

## Best Practices

**DO:**
- Export only the public API surface from the barrel `index.ts`
- Mark packages as `sideEffects: false` for tree shaking
- Use TSDoc comments on every public export
- Include `declarationMap: true` for IDE source navigation
- Test the public API, not internal helpers
- Use options objects for functions with more than two parameters
- Pin peer dependency version ranges to match supported versions
- Run `typecheck`, `test`, and `build` before publishing

**DON'T:**
- Create a shared library until two or more consumers exist
- Export a default object containing all functions (breaks tree shaking)
- Include test files in the published package
- Use mutable module-level state (prevents tree shaking and introduces hidden coupling)
- Depend on application-specific types in a shared library
- Skip semantic versioning for published packages
- Include `node_modules` or source maps in published `files`
- Use `any` in public API signatures

## Guidelines

**Essential:**
- Barrel export (`index.ts`) defines the public API boundary
- `sideEffects: false` in `package.json` for tree-shakeable bundles
- `declaration: true` and `declarationMap: true` in tsconfig
- Tests cover the public API surface

**Recommended:**
- TSDoc comments with `@example` blocks on all public exports
- Options objects with readonly properties for complex function signatures
- Separate `tsconfig.build.json` to exclude tests from output
- Type-level tests with `expectTypeOf` from Vitest

**Advanced:**
- Conditional exports with separate `types`, `import`, and `require` entries
- Peer dependency management for framework-adjacent libraries
- Private registry publishing with scoped packages
- API Extractor or similar tool for generating API reports and enforcing API surface

## Benefits

Reuse without duplication. Shared logic lives in one place with one test suite.

Independent versioning. Libraries evolve on their own timeline without blocking consumers.

Tree shaking. Consumers include only the functions they use in their final bundle.

Type safety. Generated declarations give consumers full autocompletion and type checking.

Clear boundaries. The barrel export defines an explicit contract between library and consumer.

Testability. Libraries are small, focused units that are straightforward to test in isolation.

## Related

- [typescript-monorepo-packages.md](./typescript-monorepo-packages.md) - Monorepo workspace setup for internal packages
- [typescript-module-resolution.md](./typescript-module-resolution.md) - How TypeScript resolves module imports
