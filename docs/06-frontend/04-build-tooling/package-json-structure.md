# Package.json Structure

`keywords: package-json, scripts, dependencies, devDependencies, engines, exports, package-manager, lock-files, overrides, resolutions, npm-scripts`

> **Principle:** `package.json` is the manifest of your project. Keep it intentional: every field serves a purpose, every script is documented by its name, and every dependency is in the correct category.

---

## Essential Fields

A well-structured `package.json` for a Vue/TypeScript project:

```json
{
  "name": "@myorg/portal-app",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc --noEmit && vite build",
    "preview": "vite preview",
    "test": "vitest",
    "test:coverage": "vitest run --coverage",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "typecheck": "vue-tsc --noEmit"
  },
  "dependencies": {
    "vue": "^3.4.0",
    "vue-router": "^4.3.0",
    "pinia": "^2.2.0",
    "@tanstack/vue-query": "^5.0.0"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.0.0",
    "typescript": "^5.3.0",
    "vite": "^6.0.0",
    "vitest": "^2.0.0",
    "vue-tsc": "^2.0.0",
    "eslint": "^9.0.0",
    "prettier": "^3.0.0"
  },
  "engines": {
    "node": ">=20.0.0"
  },
  "packageManager": "pnpm@9.15.0"
}
```

### Field-by-Field Explanation

**`name`** — Package identifier. Use scoped names (`@org/name`) for private packages in monorepos. Use lowercase, hyphens only.

**`version`** — Follows semver. For applications (not libraries), this is less critical but still useful for tracking deployments.

**`private: true`** — Prevents accidental publishing to npm. Set this for all applications.

**`type: "module"`** — Enables ESM (`import`/`export`) as the default module system. Required for modern tooling (Vite, ESLint flat config).

---

## Scripts Convention

Scripts should follow predictable naming so any developer (or CI system) knows what to run without reading documentation.

### Standard Script Names

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc --noEmit && vite build",
    "preview": "vite preview",
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "typecheck": "vue-tsc --noEmit"
  }
}
```

| Script | Purpose | When |
|--------|---------|------|
| `dev` | Start development server | Local development |
| `build` | Type-check and produce production build | CI/CD, deployment |
| `preview` | Preview production build locally | Pre-deployment verification |
| `test` | Run tests in watch mode | Local development |
| `test:run` | Run tests once (no watch) | CI/CD |
| `test:coverage` | Run tests with coverage report | CI/CD, quality gates |
| `lint` | Check for linting errors | CI/CD |
| `lint:fix` | Auto-fix linting errors | Local development |
| `format` | Format all files | Local development |
| `format:check` | Check formatting without changes | CI/CD |
| `typecheck` | Run TypeScript type checking | CI/CD |

### Script Naming Conventions

- Use the base name for the most common variant (`test` runs watch mode)
- Use colons for sub-commands (`test:run`, `test:coverage`, `lint:fix`)
- CI scripts should be non-interactive and non-modifying (`format:check`, not `format`)
- Keep scripts simple — avoid long shell pipelines in `package.json`

### Complex Scripts

For complex build steps, create a script file:

```json
{
  "scripts": {
    "build:analyze": "node scripts/analyze-bundle.js"
  }
}
```

```ts
// scripts/analyze-bundle.js
import { build } from 'vite';
import { visualizer } from 'rollup-plugin-visualizer';
// ...
```

---

## Dependency Management

### dependencies vs devDependencies

```json
{
  "dependencies": {
    "vue": "^3.4.0",
    "vue-router": "^4.3.0",
    "pinia": "^2.2.0",
    "@tanstack/vue-query": "^5.0.0",
    "axios": "^1.7.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "vite": "^6.0.0",
    "@vitejs/plugin-vue": "^5.0.0",
    "typescript": "^5.3.0",
    "vue-tsc": "^2.0.0",
    "vitest": "^2.0.0",
    "@vue/test-utils": "^2.4.0",
    "eslint": "^9.0.0",
    "prettier": "^3.0.0",
    "@types/node": "^20.0.0"
  }
}
```

**`dependencies`** — Packages required at runtime in the browser or server:
- Framework code (Vue, Vue Router, Pinia)
- Data fetching (Axios, TanStack Query)
- Utility libraries used in application code (Zod, date-fns)

**`devDependencies`** — Packages needed only during development and build:
- Build tools (Vite, TypeScript, vue-tsc)
- Test frameworks (Vitest, Vue Test Utils)
- Linting and formatting (ESLint, Prettier)
- Type definitions (`@types/*`)

**Rule of thumb:** If the code runs in the user's browser, it is a `dependency`. If it only runs on the developer's machine or in CI, it is a `devDependency`.

For bundled frontend applications, the distinction is primarily organizational — Vite bundles from `dependencies` regardless. But correct categorization matters for library packages and for understanding your runtime footprint.

### Version Ranges

```json
{
  "vue": "^3.4.0",     // ^major: allow minor + patch updates (3.4.x, 3.5.x)
  "vite": "~6.0.0",    // ~minor: allow patch updates only (6.0.x)
  "typescript": "5.3.3" // Exact: no updates without manual change
}
```

- Use `^` (caret) for most packages — allows non-breaking updates
- Use `~` (tilde) for packages with unstable minor releases
- Use exact versions for critical build tools where any change could break builds

---

## Engines Field

Declare the Node.js version your project requires:

```json
{
  "engines": {
    "node": ">=20.0.0"
  }
}
```

This is advisory by default. To enforce it, add an `.npmrc`:

```ini
# .npmrc
engine-strict=true
```

Or with pnpm, it is enforced automatically when the `packageManager` field is set.

---

## Exports and Main Field

For library packages (not applications), define entry points:

```json
{
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    },
    "./components": {
      "types": "./dist/components/index.d.ts",
      "import": "./dist/components/index.mjs"
    }
  },
  "files": [
    "dist"
  ]
}
```

- `exports` is the modern way to define entry points (Node 12+)
- `main` and `module` are fallbacks for older tools
- `types` must come first in each exports condition (TypeScript requires this)
- `files` controls what gets published to npm

For applications (`"private": true`), you generally do not need `exports` or `main`.

---

## Package Manager Field

Lock the package manager version for the project:

```json
{
  "packageManager": "pnpm@9.15.0"
}
```

This field is used by Corepack (built into Node.js) to automatically use the correct package manager version:

```bash
# Enable Corepack (one time)
corepack enable

# Now pnpm commands use the exact version from packageManager field
pnpm install
```

Benefits:
- Every developer and CI system uses the same package manager version
- No "works on my machine" issues from version differences
- Corepack downloads the correct version automatically

---

## Lock Files

Every package manager produces a lock file. Commit it to version control.

| Manager | Lock File | Commit? |
|---------|-----------|---------|
| npm | `package-lock.json` | Yes |
| pnpm | `pnpm-lock.yaml` | Yes |
| yarn | `yarn.lock` | Yes |

**Rules:**
- Commit exactly one lock file (the one for your chosen package manager)
- Add other lock files to `.gitignore`
- Never manually edit lock files
- Regenerate by deleting the lock file and running install (last resort only)

```bash
# .gitignore (when using pnpm)
package-lock.json
yarn.lock
```

---

## Sorting Conventions

Keep `package.json` fields in a consistent order. Use `sort-package-json` to enforce:

```bash
pnpm add -D sort-package-json
```

```json
{
  "scripts": {
    "sort-package": "sort-package-json"
  }
}
```

Standard field order:
1. `name`, `version`, `private`, `description`
2. `type`, `main`, `module`, `types`, `exports`
3. `scripts`
4. `dependencies`
5. `devDependencies`
6. `peerDependencies`
7. `engines`, `packageManager`
8. Tool-specific config (`eslintConfig`, `prettier`, etc.)

Dependencies within each section should be sorted alphabetically (most package managers do this automatically).

---

## Overrides and Resolutions

Fix transitive dependency issues without waiting for upstream fixes.

### npm overrides

```json
{
  "overrides": {
    "vulnerable-package": "^2.0.1",
    "some-package>transitive-dep": "^1.5.0"
  }
}
```

### pnpm overrides

```json
{
  "pnpm": {
    "overrides": {
      "vulnerable-package": "^2.0.1",
      "some-package>transitive-dep": "^1.5.0"
    }
  }
}
```

### yarn resolutions

```json
{
  "resolutions": {
    "vulnerable-package": "^2.0.1",
    "some-package/transitive-dep": "^1.5.0"
  }
}
```

**Use overrides/resolutions for:**
- Security vulnerabilities in transitive dependencies
- Deduplicating multiple versions of the same package
- Fixing broken transitive dependencies

**Always add a comment** (in a nearby `README` or inline if your tooling supports it) explaining why the override exists and when it can be removed.

---

## Best Practices

### DO

- Set `"private": true` for all applications to prevent accidental publishing
- Set `"type": "module"` for ESM-first projects
- Use predictable script names (`dev`, `build`, `test`, `lint`, `typecheck`)
- Keep `dependencies` and `devDependencies` correctly categorized
- Commit your lock file to version control
- Use the `engines` field to document required Node.js version
- Use the `packageManager` field with Corepack for version consistency
- Sort dependencies alphabetically

### DON'T

- Don't put build tools in `dependencies` — they belong in `devDependencies`
- Don't use `*` or empty version ranges — always specify a version constraint
- Don't commit multiple lock files — pick one package manager and stick with it
- Don't embed complex shell scripts in `scripts` — use separate script files
- Don't leave unused dependencies — audit with `depcheck` or `knip` periodically
- Don't use `prepare` or `postinstall` scripts for build steps — they slow down `npm install` for every consumer

---

## Guidelines

### Essential

- Every project has `name`, `version`, `private`, `type`, `scripts`, `dependencies`, `devDependencies`
- Standard script names: `dev`, `build`, `test`, `lint`, `typecheck`
- Lock file committed to version control
- `"type": "module"` for modern ESM tooling

### Recommended

- `engines` field specifying minimum Node.js version
- `packageManager` field with exact version for Corepack
- Separate `:check` variants for CI (non-modifying)
- Periodic dependency auditing with `pnpm audit` or `knip`

### Advanced

- `exports` field for library packages with multiple entry points
- `overrides`/`resolutions` for transitive dependency fixes
- Custom `sort-package-json` integration
- Workspace-level dependency hoisting configuration

---

## Benefits

- Single source of truth for project metadata, scripts, and dependencies
- Predictable script names enable CI automation without documentation
- Correct dependency categorization communicates runtime footprint
- Lock files guarantee reproducible installs across environments
- Package manager pinning eliminates version drift

---

## Related

- [npm-vs-pnpm-vs-yarn.md](npm-vs-pnpm-vs-yarn.md) — Choosing and configuring a package manager
- [vite-configuration.md](vite-configuration.md) — Build tool configuration that `scripts` invoke
