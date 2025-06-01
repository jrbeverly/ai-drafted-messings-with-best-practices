# TypeScript Monorepo Packages

Structuring, building, and type-checking TypeScript packages within a monorepo. Workspace configuration, shared tsconfig, internal package references, and build orchestration.

`keywords: typescript, monorepo, workspace, pnpm, npm, tsconfig, exports, turborepo, nx, packages, shared-types`

## Principle

A monorepo colocates multiple packages in a single repository, sharing tooling, configuration, and type definitions. Packages reference each other through workspace links rather than published versions, enabling instant feedback across the dependency graph while maintaining clear module boundaries.

## Monorepo Package Structure

A typical TypeScript monorepo separates applications from shared packages.

```
my-monorepo/
├── apps/
│   ├── web/                    # Vue/React frontend application
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── api/                    # Backend API service
│       ├── src/
│       ├── package.json
│       └── tsconfig.json
├── packages/
│   ├── shared-types/           # Shared TypeScript type definitions
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── ui-components/          # Shared UI component library
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── date-utils/             # Shared utility library
│       ├── src/
│       ├── package.json
│       └── tsconfig.json
├── tsconfig.base.json          # Shared TypeScript configuration
├── package.json                # Root workspace configuration
├── pnpm-workspace.yaml         # pnpm workspace definition
└── turbo.json                  # Build orchestration (optional)
```

## Workspace Setup

### pnpm Workspaces

pnpm workspaces are defined in `pnpm-workspace.yaml` at the repository root.

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

```json
// Root package.json
{
  "name": "my-monorepo",
  "private": true,
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev",
    "test": "turbo run test",
    "typecheck": "turbo run typecheck",
    "lint": "turbo run lint"
  },
  "devDependencies": {
    "turbo": "^2.0.0",
    "typescript": "^5.3.0"
  }
}
```

### npm Workspaces

npm workspaces are defined in the root `package.json`.

```json
// Root package.json
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": [
    "apps/*",
    "packages/*"
  ],
  "scripts": {
    "build": "npm run build --workspaces",
    "test": "npm run test --workspaces"
  },
  "devDependencies": {
    "typescript": "^5.3.0"
  }
}
```

## Shared TypeScript Configuration (tsconfig Base)

A shared base tsconfig at the repository root defines common compiler options. Each package extends it and adds package-specific settings.

```json
// tsconfig.base.json (repository root)
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
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "composite": true
  }
}
```

```json
// packages/date-utils/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*.ts"],
  "exclude": ["src/**/*.test.ts", "dist"]
}
```

```json
// apps/web/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "jsx": "preserve",
    "noEmit": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*.ts", "src/**/*.vue", "env.d.ts"],
  "references": [
    { "path": "../../packages/shared-types" },
    { "path": "../../packages/ui-components" },
    { "path": "../../packages/date-utils" }
  ]
}
```

The `composite: true` option in the base config enables TypeScript project references. This allows `tsc --build` to compile packages in dependency order and skip unchanged packages.

## Package.json exports Field

The `exports` field defines how other packages resolve imports from this package. It replaces the legacy `main` field for modern Node.js and bundlers.

```json
// packages/date-utils/package.json
{
  "name": "@myorg/date-utils",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    }
  },
  "scripts": {
    "build": "tsup src/index.ts --format esm,cjs --dts",
    "dev": "tsup src/index.ts --format esm,cjs --dts --watch",
    "typecheck": "tsc --noEmit",
    "test": "vitest run"
  },
  "devDependencies": {
    "tsup": "^8.0.0",
    "typescript": "^5.3.0",
    "vitest": "^1.0.0"
  }
}
```

For packages that expose multiple entry points:

```json
// packages/ui-components/package.json
{
  "name": "@myorg/ui-components",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    },
    "./button": {
      "types": "./dist/button.d.ts",
      "import": "./dist/button.js"
    },
    "./modal": {
      "types": "./dist/modal.d.ts",
      "import": "./dist/modal.js"
    }
  }
}
```

The `types` condition must come first in each export entry. TypeScript resolves conditions top to bottom and uses the first matching one.

## Internal Package References

Workspace packages reference each other by name. The package manager resolves them to the local filesystem path.

```json
// apps/web/package.json
{
  "name": "@myorg/web",
  "dependencies": {
    "@myorg/date-utils": "workspace:*",
    "@myorg/shared-types": "workspace:*",
    "@myorg/ui-components": "workspace:*"
  }
}
```

pnpm uses the `workspace:` protocol to indicate a local package. `workspace:*` links to whatever version exists in the workspace.

For npm workspaces, reference by name only:

```json
{
  "dependencies": {
    "@myorg/date-utils": "*"
  }
}
```

Usage in application code:

```ts
// apps/web/src/utils/format.ts
import { formatDate, formatRelative } from '@myorg/date-utils'
import type { User } from '@myorg/shared-types'

export function formatUserJoinDate(user: User): string {
  return formatDate(new Date(user.createdAt))
}

export function formatUserLastSeen(user: User): string {
  return formatRelative(new Date(user.lastLoginAt))
}
```

## Shared Types Package Pattern

A dedicated shared types package provides type definitions used across multiple applications without any runtime code.

```json
// packages/shared-types/package.json
{
  "name": "@myorg/shared-types",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": {
    ".": {
      "types": "./src/index.ts"
    }
  },
  "devDependencies": {
    "typescript": "^5.3.0"
  }
}
```

```ts
// packages/shared-types/src/index.ts
export type { User, CreateUserRequest, UpdateUserRequest } from './user'
export type { Book, CreateBookRequest, BookSearchParams } from './book'
export type { ApiResponse, PaginatedResponse, ApiError } from './api'
export type { Permission, Role } from './auth'
```

```ts
// packages/shared-types/src/user.ts
export interface User {
  readonly id: string
  readonly email: string
  readonly name: string
  readonly role: Role
  readonly createdAt: string
  readonly lastLoginAt: string
}

export interface CreateUserRequest {
  readonly email: string
  readonly name: string
  readonly role: Role
}

export interface UpdateUserRequest {
  readonly name?: string
  readonly email?: string
  readonly role?: Role
}
```

```ts
// packages/shared-types/src/api.ts
export interface ApiResponse<T> {
  readonly data: T
  readonly status: number
  readonly message: string
}

export interface PaginatedResponse<T> {
  readonly items: readonly T[]
  readonly total: number
  readonly page: number
  readonly pageSize: number
  readonly hasNextPage: boolean
}

export interface ApiError {
  readonly code: string
  readonly message: string
  readonly details?: Record<string, string[]>
}
```

Because the shared types package has no runtime code, its `exports` can point directly to the TypeScript source (`./src/index.ts`). No build step is required. TypeScript resolves the types at compile time.

## Build Order and Dependencies

In a monorepo, packages must be built in dependency order. If `apps/web` depends on `packages/date-utils`, then `date-utils` must be built before `web`.

### TypeScript Project References

TypeScript project references enforce build order with `tsc --build`.

```json
// apps/web/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "noEmit": true
  },
  "references": [
    { "path": "../../packages/shared-types" },
    { "path": "../../packages/date-utils" }
  ]
}
```

```bash
# Build all projects in dependency order
tsc --build apps/web/tsconfig.json

# Build with verbose output
tsc --build --verbose

# Clean build artifacts
tsc --build --clean
```

### Turborepo

Turborepo understands workspace dependency graphs and runs tasks in the correct order with caching.

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "dev": {
      "dependsOn": ["^build"],
      "cache": false,
      "persistent": true
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"]
    },
    "typecheck": {
      "dependsOn": ["^build"]
    },
    "lint": {
      "dependsOn": []
    }
  }
}
```

The `^build` syntax means "run `build` in all dependencies before running `build` in this package." This ensures packages are compiled before their consumers.

```bash
# Build everything in dependency order
turbo run build

# Build only what changed
turbo run build --filter=...[HEAD^1]

# Build a specific app and its dependencies
turbo run build --filter=@myorg/web...
```

### Nx

Nx provides similar build orchestration with a different configuration model.

```json
// nx.json
{
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["{projectRoot}/dist"]
    },
    "test": {
      "dependsOn": ["build"]
    },
    "typecheck": {
      "dependsOn": ["^build"]
    }
  }
}
```

```bash
# Build everything affected by changes
nx affected --target=build

# Build a specific project and its dependencies
nx run @myorg/web:build
```

## Type Checking Across Packages

Type check the entire monorepo to catch cross-package type errors.

```bash
# Check all packages with Turborepo
turbo run typecheck

# Check with tsc project references
tsc --build --noEmit
```

Each package's tsconfig must include its references so TypeScript understands the dependency graph:

```json
// packages/date-utils/tsconfig.json - No references (leaf package)
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*.ts"]
}
```

```json
// packages/ui-components/tsconfig.json - Depends on shared-types
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "jsx": "preserve"
  },
  "include": ["src/**/*.ts", "src/**/*.vue"],
  "references": [
    { "path": "../shared-types" }
  ]
}
```

When a type changes in `shared-types`, running `typecheck` in `ui-components` will catch any incompatibilities at compile time before they reach production.

## Development Workflow

A typical development session in a monorepo runs watch mode across packages:

```bash
# Start dev mode for all packages in parallel
turbo run dev

# Start dev for a specific app and its dependencies
turbo run dev --filter=@myorg/web...
```

For packages that need a build step, run `dev` (watch mode) so changes propagate automatically:

```json
// packages/date-utils/package.json
{
  "scripts": {
    "dev": "tsup src/index.ts --format esm,cjs --dts --watch",
    "build": "tsup src/index.ts --format esm,cjs --dts"
  }
}
```

For types-only packages, no build step is needed. The consuming app's bundler reads the TypeScript source directly through the `exports` field pointing to `.ts` files.

## Best Practices

**DO:**
- Use a shared `tsconfig.base.json` for consistent compiler options across all packages
- Put the `types` condition first in every `exports` entry
- Use `workspace:*` (pnpm) for internal package references
- Enable `composite: true` in the base tsconfig for project references
- Create a dedicated shared-types package for cross-package type definitions
- Use `turbo run typecheck` (or equivalent) to check the entire monorepo
- Point types-only packages directly to TypeScript source in `exports`
- Run builds in dependency order with `dependsOn: ["^build"]`

**DON'T:**
- Duplicate type definitions across packages (use shared-types)
- Import from another package's internal files (import only from the package name)
- Skip the `types` condition in `exports` (consumers lose type checking)
- Use relative paths like `../../packages/date-utils/src/index` to import from sibling packages
- Build types-only packages (point `exports` to source `.ts` files instead)
- Forget to add `references` entries when one package depends on another
- Put all packages at the root level (separate `apps/` and `packages/`)
- Use `path` aliases in shared packages (they are not portable across consumers)

## Guidelines

**Essential:**
- Shared `tsconfig.base.json` extended by all packages
- `exports` field with `types` condition first in every shared package
- Workspace protocol for internal dependencies (`workspace:*`)
- Type checking runs across the entire monorepo in CI

**Recommended:**
- Turborepo or Nx for build orchestration and caching
- Shared types package for cross-application type definitions
- TypeScript project references for incremental builds
- Separate `apps/` and `packages/` top-level directories

**Advanced:**
- Remote caching with Turborepo for CI/CD speed
- Filtered builds (`--filter`) for changed packages only
- Composite project references with `tsc --build` for zero-config incremental compilation
- Conditional exports with separate ESM and CJS entry points

## Benefits

Single source of truth. Shared types and utilities live in one repository with one version of the truth.

Instant cross-package feedback. Changing a type in a shared package immediately surfaces errors in all consumers.

Incremental builds. Turborepo and TypeScript project references skip unchanged packages for faster iteration.

Atomic changes. A single commit can update a shared library and all its consumers together.

Consistent tooling. Shared tsconfig, linting, and formatting rules across all packages.

Dependency clarity. The workspace dependency graph makes package relationships explicit and auditable.

## Related

- [typescript-internal-libraries.md](./typescript-internal-libraries.md) - Designing and building individual shared libraries
- [typescript-module-resolution.md](./typescript-module-resolution.md) - How TypeScript resolves imports across packages
