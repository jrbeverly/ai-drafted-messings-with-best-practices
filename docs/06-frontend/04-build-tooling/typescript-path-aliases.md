# TypeScript Path Aliases

`keywords: path-aliases, tsconfig, paths, baseUrl, resolve-alias, vite-alias, vitest-alias, absolute-imports, module-resolution, at-alias`

> **Principle:** Path aliases replace fragile relative imports with stable, readable absolute paths. Configure them once in `tsconfig.json` and synchronize with your build tool so TypeScript, Vite, and Vitest all resolve modules identically.

---

## Path Aliases in tsconfig.json

TypeScript path aliases map import specifiers to file system locations. They are configured with `paths` and optionally `baseUrl` in `tsconfig.json`.

### Basic Setup

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

This maps `@/components/Button.vue` to `src/components/Button.vue`.

### Multiple Aliases

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@composables/*": ["src/composables/*"],
      "@types/*": ["src/types/*"],
      "@assets/*": ["src/assets/*"],
      "@stores/*": ["src/stores/*"],
      "@utils/*": ["src/utils/*"],
      "@test/*": ["tests/*"]
    }
  }
}
```

### baseUrl vs paths

`baseUrl` sets the root directory for non-relative imports. `paths` defines specific mappings relative to `baseUrl`.

```json
// Option A: baseUrl + paths (common)
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}

// Option B: paths only, no baseUrl (also works)
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

With `baseUrl: "."`, paths are relative to the project root. Without `baseUrl`, paths are relative to the `tsconfig.json` location (which is usually the project root anyway).

Note: Setting `baseUrl` also allows bare imports like `import x from 'components/Foo'` (resolved relative to `baseUrl`). This can be confusing, so prefer explicit `@/` aliases over bare imports.

---

## Vite resolve.alias Synchronization

TypeScript `paths` only affect type checking. Vite needs its own alias configuration to resolve imports at build time.

**Both must match.** If they diverge, TypeScript will accept an import that Vite cannot resolve (or vice versa).

### Matching Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

```ts
// vite.config.ts
import { resolve } from 'path';
import { defineConfig } from 'vite';

export default defineConfig({
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
    },
  },
});
```

### Multiple Aliases

```ts
// vite.config.ts
import { resolve } from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
      '@components': resolve(__dirname, 'src/components'),
      '@composables': resolve(__dirname, 'src/composables'),
      '@types': resolve(__dirname, 'src/types'),
      '@assets': resolve(__dirname, 'src/assets'),
      '@stores': resolve(__dirname, 'src/stores'),
      '@utils': resolve(__dirname, 'src/utils'),
      '@test': resolve(__dirname, 'tests'),
    },
  },
});
```

### Automated Synchronization

To avoid maintaining aliases in two places, use `vite-tsconfig-paths`:

```bash
pnpm add -D vite-tsconfig-paths
```

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [
    vue(),
    tsconfigPaths(),  // Reads paths from tsconfig.json automatically
  ],
});
```

This plugin reads `tsconfig.json` paths and configures Vite aliases automatically. One source of truth, no drift.

---

## The @ Alias Convention

The `@` alias pointing to `src/` is the most common convention in Vue projects:

```ts
// Instead of fragile relative imports:
import UserCard from '../../../components/UserCard.vue';
import { useAuth } from '../../composables/useAuth';
import type { User } from '../../../types/user';

// Use stable absolute imports:
import UserCard from '@/components/UserCard.vue';
import { useAuth } from '@/composables/useAuth';
import type { User } from '@/types/user';
```

The `@` convention is widely used across Vue CLI, Nuxt, and community projects. It is immediately recognizable to Vue developers.

### Other Common Alias Patterns

```ts
// Single @ for src root (most common)
import Foo from '@/components/Foo.vue';

// Scoped aliases for deep directories
import Foo from '@components/Foo.vue';
import { useBar } from '@composables/useBar';

// Tilde for assets (less common, used by some CSS tools)
import logo from '~/assets/logo.png';
```

Stick with the single `@` alias unless your project has deep nesting that makes scoped aliases genuinely clearer.

---

## When Path Aliases Help vs Hurt

### Aliases Help When

- **Deep nesting:** `@/components/Button.vue` is clearer than `../../../../components/Button.vue`
- **Refactoring:** Moving a file does not break imports from other files (the alias path stays the same)
- **Readability:** `@/stores/userStore` immediately communicates "this is from the stores directory in src"
- **Consistency:** Every import uses the same pattern regardless of the importing file's location

### Aliases Hurt When

- **Overused:** Creating an alias for every directory adds cognitive overhead without proportional benefit
- **Sibling imports:** `./UserCard.vue` is clearer than `@/components/users/UserCard.vue` when you are already in the `users` directory
- **Tooling gaps:** If a tool does not support your aliases, imports break silently
- **New developer confusion:** Too many aliases create a learning curve

### Guidelines for Alias Usage

```ts
// USE aliases for cross-directory imports
// (importing from a different top-level directory)
import { useAuth } from '@/composables/useAuth';
import type { User } from '@/types/user';

// USE relative imports for same-directory or sibling files
import UserAvatar from './UserAvatar.vue';
import { formatUserName } from './utils';

// AVOID deep relative imports — use aliases instead
// BAD:
import { useAuth } from '../../../composables/useAuth';
// GOOD:
import { useAuth } from '@/composables/useAuth';
```

A practical rule: use relative imports for files in the same directory or one level up. Use aliases for anything deeper.

---

## Module Resolution with Aliases

TypeScript must be configured with the correct `moduleResolution` for aliases to work:

```json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

### moduleResolution Options

| Value | Use When | Alias Support |
|-------|----------|---------------|
| `bundler` | Using Vite, webpack, or other bundlers | Yes |
| `node16` / `nodenext` | Writing Node.js packages | Yes (with extensions) |
| `node` (legacy) | Legacy projects | Yes |

For Vite projects, always use `"moduleResolution": "bundler"`. This matches how Vite resolves modules and supports path aliases without requiring file extensions in imports.

---

## Absolute vs Relative Imports

### Relative Imports

```ts
// Relative to the current file
import Sibling from './Sibling.vue';
import Parent from '../Parent.vue';
import Deep from '../../../shared/Deep.vue';  // Fragile
```

**Pros:** No configuration needed, clear locality for nearby files.
**Cons:** Break when files move, unreadable when deep.

### Absolute Imports (via Aliases)

```ts
// Resolved from project root via alias
import Deep from '@/shared/Deep.vue';
import { useAuth } from '@/composables/useAuth';
```

**Pros:** Stable across refactoring, readable, consistent.
**Cons:** Require tooling configuration, can obscure locality.

### Recommended Strategy

```ts
// Same directory or one level up: relative
import ChildComponent from './ChildComponent.vue';
import { helperFn } from '../utils';

// Everything else: alias
import { useAuth } from '@/composables/useAuth';
import type { User } from '@/types/user';
import MainLayout from '@/layouts/MainLayout.vue';
```

---

## Alias Configuration for Testing (Vitest)

Vitest uses Vite's config by default, so aliases defined in `vite.config.ts` work automatically in tests:

```ts
// vite.config.ts — aliases work in both app and tests
export default defineConfig({
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
    },
  },
  test: {
    // Vitest config — inherits resolve.alias automatically
    globals: true,
    environment: 'jsdom',
  },
});
```

### Separate Vitest Config

If you use a separate `vitest.config.ts`, ensure aliases are defined there too:

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import vue from '@vitejs/plugin-vue';
import { resolve } from 'path';

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
      '@test': resolve(__dirname, 'tests'),
    },
  },
  test: {
    globals: true,
    environment: 'jsdom',
  },
});
```

### Using vite-tsconfig-paths with Vitest

If you use `vite-tsconfig-paths`, include it in the test config:

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import vue from '@vitejs/plugin-vue';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [vue(), tsconfigPaths()],
  test: {
    globals: true,
    environment: 'jsdom',
  },
});
```

### Test-Specific Aliases

Add a `@test` alias for test utilities and fixtures:

```json
// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["src/*"],
      "@test/*": ["tests/*"]
    }
  }
}
```

```ts
// In a test file:
import { renderWithProviders } from '@test/helpers/render';
import { mockUser } from '@test/fixtures/user';
```

---

## Common Patterns

### Standard Vue Project Aliases

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

Usage:

```ts
import App from '@/App.vue';
import { useAuth } from '@/composables/useAuth';
import type { User } from '@/types/user';
import MainLayout from '@/layouts/MainLayout.vue';
import UserCard from '@/components/users/UserCard.vue';
import { userStore } from '@/stores/userStore';
import { formatDate } from '@/utils/date';
```

### Monorepo with Shared Packages

```json
// apps/portal/tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@shared/*": ["../../packages/shared/src/*"]
    }
  }
}
```

### Feature-Based Architecture

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@features/*": ["src/features/*"],
      "@shared/*": ["src/shared/*"]
    }
  }
}
```

```ts
import { UserProfile } from '@features/users/components/UserProfile.vue';
import { useProducts } from '@features/products/composables/useProducts';
import { BaseButton } from '@shared/components/BaseButton.vue';
```

---

## Troubleshooting

### "Cannot find module '@/...'"

1. Check `tsconfig.json` has `paths` configured
2. Check `vite.config.ts` has matching `resolve.alias`
3. Ensure `moduleResolution` is `"bundler"` for Vite projects
4. Restart the TypeScript language server in your editor

### Aliases Work in Editor but Fail at Build

Vite and TypeScript resolve independently. If TypeScript accepts the import but Vite fails:
- Verify `vite.config.ts` aliases match `tsconfig.json` paths
- Or use `vite-tsconfig-paths` to auto-sync

### Aliases Work in App but Fail in Tests

If Vitest uses a separate config, it needs its own alias definitions:
- Include aliases in `vitest.config.ts`
- Or use `vite-tsconfig-paths` plugin in the test config

---

## Best Practices

### DO

- Use the `@` alias for `src/` as the standard convention
- Keep `tsconfig.json` paths and `vite.config.ts` aliases synchronized
- Use `vite-tsconfig-paths` to maintain a single source of truth
- Use relative imports for same-directory files
- Use aliases for cross-directory imports (more than one level up)
- Include test directories in alias configuration

### DON'T

- Don't create an alias for every directory — one `@` alias covers most needs
- Don't use aliases for sibling files — `./Sibling.vue` is clearer
- Don't mix bare imports (`components/Foo`) with alias imports (`@/components/Foo`)
- Don't forget to configure aliases for your test runner
- Don't use `baseUrl` as a substitute for explicit aliases — it enables ambiguous bare imports
- Don't use aliases that shadow npm package names (e.g., don't alias `@types` if you use `@types/*` packages)

---

## Guidelines

### Essential

- `@/*` mapped to `src/*` in `tsconfig.json`
- Matching alias in `vite.config.ts` (or use `vite-tsconfig-paths`)
- `moduleResolution: "bundler"` in `tsconfig.json` for Vite projects
- Aliases work in both application code and tests

### Recommended

- Use `vite-tsconfig-paths` plugin for automatic synchronization
- Relative imports for same-directory, aliases for cross-directory
- `@test/*` alias for test utilities and fixtures
- ESLint rule to enforce alias usage over deep relative imports

### Advanced

- Feature-scoped aliases (`@features/*`) for large applications
- Monorepo cross-package aliases with workspace references
- Custom ESLint rule for import path conventions
- IDE snippet integration for common alias patterns

---

## Benefits

- Readable imports that communicate module location at a glance
- Stable import paths that survive file restructuring
- Consistent import style across the entire codebase
- Single source of truth with `vite-tsconfig-paths` synchronization
- Simplified refactoring without cascading import path changes

---

## Related

- [vite-configuration.md](vite-configuration.md) — Vite `resolve.alias` configuration and build tool setup
- [typescript-module-resolution.md](typescript-module-resolution.md) — TypeScript module resolution strategies and `moduleResolution` options
