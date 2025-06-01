# TypeScript Declaration Files

Type declarations for non-TypeScript code, ambient modules, and publishable type definitions. How to write, generate, and consume `.d.ts` files.

`keywords: typescript, declaration, d.ts, declare, ambient, module-augmentation, triple-slash, definitelytyped, types, global`

## Principle

Declaration files describe the shape of code that exists elsewhere. They bridge the gap between TypeScript's type system and JavaScript libraries, environment globals, non-code assets, and shared type contracts. Every external surface that TypeScript cannot infer on its own needs a declaration.

## What Are Declaration Files

A `.d.ts` file contains only type information with no runtime code. TypeScript uses these files to understand the types of values that exist at runtime but were not written in TypeScript.

```ts
// math-utils.d.ts
// Declares types for a JavaScript module without rewriting it in TypeScript

export declare function add(a: number, b: number): number
export declare function multiply(a: number, b: number): number
export declare const PI: number
```

The `declare` keyword tells TypeScript that the implementation exists elsewhere. The compiler trusts the declaration and uses it for type checking without emitting any JavaScript.

## The declare Keyword

`declare` introduces type information for values that exist at runtime but are defined outside TypeScript's compilation scope.

```ts
// Declare a global variable injected by the server
declare const __APP_VERSION__: string

// Declare a global function available in the environment
declare function gtag(command: string, ...args: unknown[]): void

// Declare a class that exists at runtime (e.g., from a script tag)
declare class Analytics {
  track(event: string, properties?: Record<string, unknown>): void
  identify(userId: string): void
}

// Declare a namespace for grouping related declarations
declare namespace NodeJS {
  interface ProcessEnv {
    NODE_ENV: 'development' | 'production' | 'test'
    API_BASE_URL: string
    VITE_APP_TITLE: string
  }
}
```

## Ambient Declarations

Ambient declarations describe types that are available globally without importing. They typically live in files included by `tsconfig.json` but not directly imported.

```ts
// env.d.ts - Global environment types
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string
  readonly VITE_APP_TITLE: string
  readonly VITE_ENABLE_ANALYTICS: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

```ts
// global.d.ts - Extend the global scope
declare global {
  interface Window {
    __INITIAL_STATE__: Record<string, unknown>
    dataLayer: Array<Record<string, unknown>>
  }

  // Global type available without import
  type Nullable<T> = T | null
  type Optional<T> = T | undefined
}

// Required to make this file a module so `declare global` works
export {}
```

The `declare global` block extends the global scope from within a module file. The trailing `export {}` ensures TypeScript treats the file as a module rather than a script.

## Declaring Modules for Non-TypeScript Files

When importing non-code assets like `.vue` files, `.svg` files, or `.css` modules, TypeScript needs declarations to understand what those imports resolve to.

```ts
// shims-vue.d.ts
declare module '*.vue' {
  import type { DefineComponent } from 'vue'
  const component: DefineComponent<
    Record<string, never>,
    Record<string, never>,
    unknown
  >
  export default component
}
```

```ts
// assets.d.ts
// SVG imports resolve to a URL string
declare module '*.svg' {
  const content: string
  export default content
}

// PNG and JPG imports
declare module '*.png' {
  const content: string
  export default content
}

declare module '*.jpg' {
  const content: string
  export default content
}

// CSS modules resolve to a record of class names
declare module '*.module.css' {
  const classes: Record<string, string>
  export default classes
}

// Plain CSS imports have no exports
declare module '*.css' {
  const content: string
  export default content
}

// JSON imports
declare module '*.json' {
  const value: unknown
  export default value
}
```

```ts
// markdown.d.ts - Declare a module for markdown file imports
declare module '*.md' {
  const content: string
  export default content
}
```

## Module Augmentation

Module augmentation extends existing module declarations without modifying the original source. This is useful for adding properties to third-party library types.

```ts
// vue-router.d.ts - Add typed route meta fields
import 'vue-router'

declare module 'vue-router' {
  interface RouteMeta {
    requiresAuth?: boolean
    title?: string
    roles?: string[]
  }
}
```

```ts
// pinia.d.ts - Add custom properties to Pinia stores
import 'pinia'

declare module 'pinia' {
  export interface PiniaCustomProperties {
    $analytics: {
      track(event: string): void
    }
  }
}
```

```ts
// axios.d.ts - Extend Axios request config
import 'axios'

declare module 'axios' {
  export interface AxiosRequestConfig {
    skipAuth?: boolean
    retryCount?: number
  }
}
```

Module augmentation requires the file to be a module (has at least one `import` or `export`). The `import` of the target module at the top is what makes augmentation work rather than replacement.

## Triple-Slash Directives

Triple-slash directives are single-line comments at the top of a file that instruct the compiler to include additional type files.

```ts
// env.d.ts
/// <reference types="vite/client" />
/// <reference types="vue/macros-global" />

// The reference directive tells TypeScript to include type definitions
// from the specified package without an explicit import
```

```ts
// worker.d.ts
/// <reference lib="webworker" />

// Use lib directive to include built-in library definitions
// This makes WebWorker globals available (self, postMessage, etc.)

declare const self: DedicatedWorkerGlobalScope

self.onmessage = (event: MessageEvent<string>) => {
  self.postMessage(`Received: ${event.data}`)
}
```

When to use triple-slash directives:
- Reference type packages that do not have runtime imports (`vite/client`, `jest`)
- Include built-in lib definitions for specific environments (`webworker`, `dom`)
- Reference other `.d.ts` files in non-module scripts

When NOT to use triple-slash directives:
- When a regular `import` or `import type` achieves the same result
- In files that already have `import` statements (use `import type` instead)

## Global Type Declarations (env.d.ts)

A common pattern is an `env.d.ts` file in the project root or `src/` directory that declares environment-specific globals.

```ts
// src/env.d.ts
/// <reference types="vite/client" />

// Environment variables
interface ImportMetaEnv {
  readonly VITE_API_URL: string
  readonly VITE_APP_TITLE: string
  readonly VITE_SENTRY_DSN: string
  readonly VITE_ENABLE_MOCKS: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}

// Build-time constants replaced by Vite
declare const __APP_VERSION__: string
declare const __BUILD_DATE__: string
declare const __COMMIT_HASH__: string
```

Ensure the file is included in your `tsconfig.json`:

```json
{
  "compilerOptions": { /* ... */ },
  "include": [
    "src/**/*.ts",
    "src/**/*.vue",
    "src/env.d.ts"
  ]
}
```

## Generating Declarations from Source

When building a TypeScript library, the compiler can generate `.d.ts` files automatically from your source code.

```json
// tsconfig.json for a library
{
  "compilerOptions": {
    "declaration": true,
    "declarationDir": "./dist/types",
    "declarationMap": true,
    "emitDeclarationOnly": false,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*.ts"],
  "exclude": ["src/**/*.test.ts", "src/**/*.spec.ts"]
}
```

Key compiler options for declaration generation:

```ts
// declaration: true
//   Generates .d.ts files alongside .js output

// declarationDir: "./dist/types"
//   Places declaration files in a separate directory

// declarationMap: true
//   Generates .d.ts.map files for "Go to Definition" to navigate
//   to the original .ts source instead of the .d.ts file

// emitDeclarationOnly: true
//   Only generates .d.ts files, no .js output
//   Useful when another tool (esbuild, swc) handles JS compilation
```

Example library structure after build:

```
my-library/
├── src/
│   ├── index.ts
│   ├── utils.ts
│   └── types.ts
├── dist/
│   ├── index.js
│   ├── utils.js
│   └── types/
│       ├── index.d.ts
│       ├── utils.d.ts
│       └── types.d.ts
├── package.json
└── tsconfig.json
```

## Publishable Type Declarations

When publishing a package with type declarations, configure `package.json` to point consumers to the correct type files.

```json
{
  "name": "@myorg/shared-types",
  "version": "1.0.0",
  "main": "./dist/index.js",
  "module": "./dist/index.mjs",
  "types": "./dist/types/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/types/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.js"
    },
    "./utils": {
      "types": "./dist/types/utils.d.ts",
      "import": "./dist/utils.mjs",
      "require": "./dist/utils.js"
    }
  },
  "files": [
    "dist"
  ]
}
```

The `types` condition in `exports` must come first. TypeScript resolves conditions in order and stops at the first match.

## DefinitelyTyped (@types/)

DefinitelyTyped is the repository for community-maintained type declarations for JavaScript packages that do not ship their own types.

```bash
# Install types for a package that lacks built-in declarations
npm install --save-dev @types/lodash
npm install --save-dev @types/node
npm install --save-dev @types/express
```

How TypeScript resolves `@types/` packages:

```ts
// When you write:
import _ from 'lodash'

// TypeScript looks for types in this order:
// 1. lodash/package.json "types" or "typings" field
// 2. lodash/index.d.ts
// 3. @types/lodash/index.d.ts (from node_modules/@types/)
```

Control which `@types/` packages are included:

```json
// tsconfig.json
{
  "compilerOptions": {
    // Only include specific @types packages (default: all)
    "types": ["node", "vite/client"],

    // Or control where TypeScript looks for @types
    "typeRoots": ["./node_modules/@types", "./src/types"]
  }
}
```

When to use `types` vs `typeRoots`:
- Use `types` to limit which `@types/` packages are auto-included (prevents ambient type pollution)
- Use `typeRoots` to add custom directories alongside `node_modules/@types`
- Omit both to include all `@types/` packages (the default)

## Writing Declaration Files for JavaScript Libraries

When wrapping an untyped JavaScript library that lacks `@types/` coverage, write a local declaration file.

```ts
// types/legacy-chart-lib.d.ts
declare module 'legacy-chart-lib' {
  export interface ChartOptions {
    type: 'bar' | 'line' | 'pie'
    data: number[]
    labels: string[]
    colors?: string[]
    animate?: boolean
  }

  export interface ChartInstance {
    render(): void
    update(data: number[]): void
    destroy(): void
  }

  export default function createChart(
    element: HTMLElement,
    options: ChartOptions
  ): ChartInstance
}
```

```ts
// Usage - now fully typed
import createChart from 'legacy-chart-lib'

const chart = createChart(document.getElementById('chart')!, {
  type: 'bar',
  data: [10, 20, 30],
  labels: ['A', 'B', 'C'],
})

chart.update([40, 50, 60])
```

## Best Practices

**DO:**
- Place global declarations in a dedicated `env.d.ts` or `global.d.ts` file
- Use `declare module` for non-TypeScript file imports (`.vue`, `.svg`, `.css`)
- Enable `declarationMap` in libraries for better IDE navigation
- Put the `types` condition first in `package.json` exports
- Use module augmentation to extend third-party types rather than patching
- Include declaration files in `tsconfig.json` via the `include` array
- Generate declarations from source with `declaration: true` instead of hand-writing them
- Install `@types/` packages as `devDependencies`

**DON'T:**
- Use `declare` in regular `.ts` files that contain runtime code (use it only in `.d.ts` files or for truly ambient values)
- Replace an entire module's types with `declare module` when augmentation suffices
- Use triple-slash directives when `import type` works
- Forget `export {}` when using `declare global` in a file that has no other imports/exports
- Publish packages without a `types` field in `package.json`
- Use `any` in declaration files as a shortcut (use `unknown` if the type is truly unknown)
- Commit auto-generated `.d.ts` files to source control for application code (only for published libraries)

## Guidelines

**Essential:**
- Every project has an `env.d.ts` for environment variables and build-time constants
- All non-TypeScript imports (`.vue`, `.svg`, `.css`) have corresponding `declare module` entries
- Library packages include generated `.d.ts` files with `declaration: true`
- `@types/` packages installed for all untyped dependencies

**Recommended:**
- Use module augmentation for extending third-party library types
- Enable `declarationMap` in library tsconfig for source navigation
- Restrict `@types` auto-inclusion with the `types` compiler option in large projects
- Organize custom declarations in a `types/` or `src/types/` directory

**Advanced:**
- Write local declaration files for untyped internal JavaScript libraries
- Use `typeRoots` to add custom type directories alongside `@types`
- Combine `emitDeclarationOnly` with esbuild/swc for faster builds with correct types
- Publish conditional exports with separate `types` entries per subpath

## Benefits

Type safety across boundaries. Non-TypeScript code gets type coverage without rewriting.

IDE support everywhere. Autocompletion and hover documentation for globals, assets, and third-party libraries.

Publishable contracts. Generated declarations let consumers of your library get full type checking.

Incremental adoption. Declaration files let TypeScript coexist with JavaScript during migration.

Explicit environment. `env.d.ts` documents every global and environment variable in one place.

## Related

- [typescript-module-resolution.md](./typescript-module-resolution.md) - How TypeScript finds and resolves modules
- [typescript-strict-mode.md](./typescript-strict-mode.md) - Strict compiler options that affect declarations
