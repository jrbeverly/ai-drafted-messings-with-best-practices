# TypeScript Module Resolution

`keywords: module-resolution, bundler, NodeNext, path-aliases, barrel-exports, tree-shaking, isolatedModules, verbatimModuleSyntax, import-type, tsconfig-references, env.d.ts`

## Principle

**Configure module resolution to match your runtime and bundler, not the other way around.** Use `moduleResolution: "bundler"` for Vite/webpack projects, `"NodeNext"` for Node.js libraries. Keep imports explicit with `import type`, avoid barrel exports that harm tree shaking, and use path aliases to eliminate fragile relative paths.

---

## Module Resolution Strategies

TypeScript offers several module resolution strategies. The correct choice depends on your runtime environment and build toolchain.

### `moduleResolution: "bundler"`

Use for frontend projects built with Vite, webpack, esbuild, or any bundler.

```jsonc
// tsconfig.json for Vite/Vue projects
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true, // Allow .ts in import paths
    "noEmit": true                       // Bundler handles emit
  }
}
```

**Why `bundler`:**
- Matches how modern bundlers actually resolve modules
- Supports `package.json` `exports` field
- Allows extensionless imports (e.g., `import { foo } from "./utils"`)
- Supports `import` and `require` conditions in package.json
- Does not require file extensions in import paths (unlike `NodeNext`)

### `moduleResolution: "NodeNext"`

Use for Node.js libraries, CLI tools, and server-side code that runs directly in Node.js.

```jsonc
// tsconfig.json for Node.js library
{
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist"
  }
}
```

**Key differences from `bundler`:**
- Requires file extensions in relative imports: `import { foo } from "./utils.js"`
- Respects `package.json` `"type": "module"` or `"type": "commonjs"`
- Distinguishes between ESM and CJS based on file extension (`.mts`/`.cts`)
- Enforces Node.js resolution rules strictly

```typescript
// NodeNext requires extensions
import { helper } from "./utils.js";     // OK (even though source is .ts)
import { helper } from "./utils";         // ERROR: relative import needs extension

// Package imports use package.json exports
import { something } from "my-package";   // Resolved via exports field
```

### `moduleResolution: "Node"` (Legacy)

The pre-Node 12 resolution strategy. Matches the classic `require()` algorithm. Only use for legacy projects that cannot upgrade.

```jsonc
// Legacy -- avoid for new projects
{
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "Node"
  }
}
```

### Choosing the Right Strategy

| Project Type | `moduleResolution` | `module` |
|---|---|---|
| Vite + Vue frontend | `"bundler"` | `"ESNext"` |
| webpack frontend | `"bundler"` | `"ESNext"` |
| Node.js ESM library | `"NodeNext"` | `"NodeNext"` |
| Node.js CJS library | `"NodeNext"` | `"NodeNext"` |
| Legacy Node.js project | `"Node"` | `"CommonJS"` |

---

## Path Aliases

Path aliases replace long relative import paths with short, readable prefixes.

### Configuration

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@composables/*": ["src/composables/*"],
      "@utils/*": ["src/utils/*"],
      "@types/*": ["src/types/*"]
    }
  }
}
```

### Vite Configuration (Must Match tsconfig)

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import { resolve } from "path";

export default defineConfig({
  resolve: {
    alias: {
      "@": resolve(__dirname, "src"),
      "@components": resolve(__dirname, "src/components"),
      "@composables": resolve(__dirname, "src/composables"),
      "@utils": resolve(__dirname, "src/utils"),
      "@types": resolve(__dirname, "src/types"),
    },
  },
});
```

### Usage

```typescript
// WITHOUT path aliases: fragile, hard to read
import { useAuth } from "../../../composables/useAuth";
import { formatDate } from "../../../../utils/date";
import type { User } from "../../../types/user";

// WITH path aliases: stable, readable
import { useAuth } from "@composables/useAuth";
import { formatDate } from "@utils/date";
import type { User } from "@types/user";
```

**Rules for path aliases:**
- Keep the number of aliases small (3-6 for most projects)
- Always configure aliases in both tsconfig and your bundler
- Use `@/` as the root alias for `src/`
- Only create sub-aliases for directories used frequently across the codebase

---

## Barrel Exports (`index.ts`)

Barrel files re-export from multiple modules through a single entry point.

### When Barrel Exports Help

```typescript
// src/components/index.ts (barrel file)
export { Button } from "./Button.vue";
export { Input } from "./Input.vue";
export { Modal } from "./Modal.vue";
export { Tooltip } from "./Tooltip.vue";

// Consumer: clean single import
import { Button, Input, Modal } from "@components";
```

Barrel exports improve developer experience when:
- The module is a library with a stable public API
- Consumers typically import multiple items from the same directory
- The directory is small (under 20 exports)

### When Barrel Exports Hurt

#### Tree Shaking Problems

```typescript
// src/utils/index.ts -- barrel re-exports EVERYTHING
export { formatDate } from "./date";
export { formatCurrency } from "./currency";
export { heavyMathLibrary } from "./math";     // 50KB
export { chartRenderer } from "./charts";       // 100KB

// Consumer only needs formatDate, but bundler may include everything
import { formatDate } from "@utils";
// In development: all 4 modules are loaded and evaluated
// In production: depends on bundler's tree-shaking capability
```

**Direct imports avoid this problem entirely:**

```typescript
import { formatDate } from "@utils/date";
// Only date.ts is loaded -- no ambiguity
```

#### Circular Dependency Risks

```typescript
// src/models/index.ts
export { User } from "./User";
export { Post } from "./Post";

// src/models/User.ts
import { Post } from "./index"; // Circular! User -> index -> Post -> index -> User

// FIXED: Direct import
import { Post } from "./Post"; // No cycle
```

#### Slow Development Builds

Large barrel files force the bundler to process every re-exported module even when only one export is used. This slows down HMR (Hot Module Replacement) in development.

### Barrel Export Guidelines

```
DO use barrels for:
  - Library public APIs (explicit, curated surface area)
  - Type-only re-exports (no runtime cost)
  - Small, cohesive module groups (under 10 exports)

DON'T use barrels for:
  - Utility directories with many independent modules
  - Directories where modules have large transitive dependencies
  - Internal implementation details (only for public-facing APIs)
  - When you observe slow HMR or large bundle sizes
```

---

## `import type` Syntax

TypeScript can distinguish between value imports (needed at runtime) and type imports (erased at compile time).

### Explicit Type Imports

```typescript
// Value import: included in runtime bundle
import { UserService } from "./UserService";

// Type-only import: erased at compile time, zero runtime cost
import type { User } from "./types";
import type { Config } from "./config";

// Mixed: some values, some types
import { createUser, type User, type CreateUserInput } from "./users";
```

### Why Use `import type`

1. **Bundle size:** Type imports are guaranteed to be erased; no risk of accidentally bundling type-only modules
2. **Circular dependency safety:** Type imports cannot create runtime circular dependencies
3. **Clarity:** Makes it obvious which imports are types vs runtime values
4. **Required by `isolatedModules`:** Some bundlers process files individually and cannot determine if an import is type-only

---

## `isolatedModules`

When enabled, TypeScript ensures each file can be independently transpiled (as Vite, esbuild, and swc do).

```jsonc
{
  "compilerOptions": {
    "isolatedModules": true
  }
}
```

### What `isolatedModules` Disallows

```typescript
// ERROR: Re-exporting a type without 'type' keyword
// The transpiler cannot know if 'User' is a type or value
export { User } from "./types";

// FIXED: Explicit type re-export
export type { User } from "./types";

// ERROR: const enum (requires cross-file analysis)
const enum Direction {
  Up,
  Down,
}

// FIXED: Regular enum or union type
enum Direction {
  Up,
  Down,
}
// Or: type Direction = "up" | "down";
```

**Always enable `isolatedModules`** for Vite/esbuild projects. These tools transpile files individually and cannot perform cross-file type analysis.

---

## `verbatimModuleSyntax`

Introduced in TypeScript 5.0, this replaces `isolatedModules` and `importsNotUsedAsValues` with a simpler rule: what you write is what you get.

```jsonc
{
  "compilerOptions": {
    "verbatimModuleSyntax": true
  }
}
```

### How It Works

```typescript
// With verbatimModuleSyntax:

// This import is KEPT in output (runtime import)
import { something } from "./module";

// This import is REMOVED from output (type-only)
import type { SomeType } from "./module";

// Mixed: value is kept, type is removed
import { createUser, type User } from "./users";
// Output: import { createUser } from "./users";
```

**The rule is simple:** `import type` is always erased; `import` (without `type`) is always kept. No guessing, no heuristics.

**Recommendation:** Use `verbatimModuleSyntax` instead of `isolatedModules` for TypeScript 5.0+ projects. It is stricter and more predictable.

---

## Resolving `.vue` Files (`env.d.ts`)

TypeScript does not understand `.vue` files by default. A declaration file tells TypeScript how to treat Vue single-file components.

### The `env.d.ts` File

```typescript
// src/env.d.ts (or src/vite-env.d.ts)

/// <reference types="vite/client" />

// Declare .vue module type so TypeScript can import .vue files
declare module "*.vue" {
  import type { DefineComponent } from "vue";
  const component: DefineComponent<
    Record<string, unknown>,
    Record<string, unknown>,
    unknown
  >;
  export default component;
}
```

**Explanation:**
- `/// <reference types="vite/client" />` provides types for Vite-specific features (`import.meta.env`, asset imports)
- The `declare module "*.vue"` block tells TypeScript that every `.vue` import is a Vue component
- This is a coarse type -- for precise component prop types, use `vue-tsc` or IDE extensions (Volar)

### Ensuring `env.d.ts` Is Included

```jsonc
// tsconfig.json
{
  "include": [
    "src/**/*.ts",
    "src/**/*.vue",
    "src/env.d.ts"     // Explicit inclusion (usually covered by src/**/*.ts)
  ]
}
```

---

## TypeScript Project References

Project references enable splitting a large codebase into smaller, independently compiled TypeScript projects. Each project has its own `tsconfig.json` and can reference other projects.

### When to Use Project References

- Monorepos with multiple packages
- Projects with separate `src` and `test` configurations
- Large codebases where full recompilation is slow
- Shared libraries used by multiple applications

### Configuration

```jsonc
// tsconfig.json (root)
{
  "files": [],
  "references": [
    { "path": "./src" },
    { "path": "./tests" }
  ]
}

// src/tsconfig.json
{
  "compilerOptions": {
    "composite": true,          // Required for project references
    "outDir": "../dist",
    "rootDir": ".",
    "declaration": true,        // Required for composite projects
    "declarationMap": true,     // Enables "go to definition" across projects
    "strict": true,
    "module": "ESNext",
    "moduleResolution": "bundler"
  },
  "include": ["./**/*.ts", "./**/*.vue"]
}

// tests/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "outDir": "../dist-tests",
    "rootDir": ".",
    "strict": true,
    "module": "ESNext",
    "moduleResolution": "bundler"
  },
  "include": ["./**/*.ts"],
  "references": [
    { "path": "../src" }        // Tests reference src
  ]
}
```

### Vue/Vite Project References (Recommended Structure)

Vue projects commonly split configuration for the app and Node-side tooling:

```jsonc
// tsconfig.json (root -- references only)
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ]
}

// tsconfig.app.json (Vue app code)
{
  "compilerOptions": {
    "composite": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "jsx": "preserve",
    "noEmit": true,
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*.ts", "src/**/*.vue", "src/env.d.ts"]
}

// tsconfig.node.json (Vite config, scripts, etc.)
{
  "compilerOptions": {
    "composite": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noEmit": true,
    "lib": ["ES2022"]
  },
  "include": ["vite.config.ts", "scripts/**/*.ts"]
}
```

### Building with Project References

```bash
# Build all projects in dependency order
tsc --build

# Build with verbose output
tsc --build --verbose

# Clean build artifacts
tsc --build --clean

# Force rebuild
tsc --build --force
```

---

## Best Practices

### DO

- **DO** use `moduleResolution: "bundler"` for all Vite/webpack frontend projects
- **DO** use `verbatimModuleSyntax` (or `isolatedModules`) for bundler-based projects
- **DO** use `import type` for all type-only imports
- **DO** configure path aliases in both `tsconfig.json` and your bundler config
- **DO** create an `env.d.ts` file for Vue projects to declare `.vue` module types
- **DO** use project references to separate app code from tooling configuration

### DON'T

- **DON'T** use `moduleResolution: "Node"` for new projects -- use `"bundler"` or `"NodeNext"`
- **DON'T** create barrel exports (`index.ts`) for large utility directories
- **DON'T** import from barrel files within the same package (use direct imports)
- **DON'T** mix `require()` and `import` in the same project without `"NodeNext"` module resolution
- **DON'T** forget to sync path aliases between tsconfig and bundler configuration
- **DON'T** use `const enum` when `isolatedModules` or `verbatimModuleSyntax` is enabled

---

## Guidelines

### Essential

- Set `moduleResolution: "bundler"` and `module: "ESNext"` for Vite projects
- Enable `verbatimModuleSyntax` or `isolatedModules` to ensure per-file transpilability
- Create `env.d.ts` with `declare module "*.vue"` for Vue TypeScript support
- Use `import type` for all type-only imports to guarantee compile-time erasure

### Recommended

- Configure 3-6 path aliases (`@/`, `@components/`, `@composables/`, etc.)
- Use direct imports instead of barrel files for utility modules
- Split tsconfig into `tsconfig.app.json` and `tsconfig.node.json` with project references
- Enable `declarationMap: true` in composite projects for cross-project navigation

### Advanced

- Use `package.json` `exports` field with `"types"` condition for library packages
- Configure `typesVersions` in `package.json` for libraries supporting multiple TS versions
- Use `tsc --build --watch` for incremental compilation in monorepos
- Audit barrel exports with bundle analysis tools to detect tree-shaking failures

---

## Benefits

- Eliminates module resolution mismatches between TypeScript and the runtime/bundler
- Reduces bundle size through explicit type imports and proper tree shaking
- Improves developer experience with readable path aliases
- Enables incremental compilation through project references
- Prevents circular dependencies by using direct imports instead of barrels
- Ensures `.vue` file imports are properly typed in the IDE and at build time

---

## Related

- [typescript-strict-mode.md](typescript-strict-mode.md) -- Compiler options including strict and module settings
- [typescript-declaration-files.md](typescript-declaration-files.md) -- Writing and consuming `.d.ts` files
