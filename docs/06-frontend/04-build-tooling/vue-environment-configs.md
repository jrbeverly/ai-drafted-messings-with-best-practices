# Vue Environment Configs

`keywords: environment, env-files, configuration, import-meta-env, feature-flags, runtime-config, Vite-define, staging`

## Principle

**Separate configuration from code using environment-specific variables so the same build artifact can behave correctly across development, staging, and production.** Use build-time variables for values that affect bundle output and runtime injection for values that must change without rebuilding.

---

## Multi-Environment Builds

Most applications need at least three environments with different configuration:

| Environment | Purpose | API URL | Debug | Analytics |
|---|---|---|---|---|
| Development | Local development | `http://localhost:3001` | Enabled | Disabled |
| Staging | Pre-production testing | `https://api.staging.example.com` | Enabled | Disabled |
| Production | Live users | `https://api.example.com` | Disabled | Enabled |

```bash
# Build for each environment using Vite's --mode flag
vite build --mode development
vite build --mode staging
vite build --mode production
```

---

## .env Files Hierarchy

Vite loads environment files in a specific order. More specific files override less specific ones.

### File Loading Order (for `--mode production`)

```
.env                  # Loaded in all modes
.env.local            # Loaded in all modes, gitignored
.env.production       # Loaded only in production mode
.env.production.local # Loaded only in production mode, gitignored
```

Later files override earlier ones. `.local` files are always gitignored and never committed.

### File Structure

```
project-root/
  .env                    # Shared defaults (committed)
  .env.local              # Local overrides (gitignored)
  .env.development        # Development config (committed)
  .env.staging            # Staging config (committed)
  .env.production         # Production config (committed)
  .env.production.local   # Local production overrides (gitignored)
```

### Example Files

```bash
# .env (shared defaults, committed to repo)
VITE_APP_NAME=MyApplication
VITE_DEFAULT_LOCALE=en
VITE_PAGINATION_SIZE=25
```

```bash
# .env.development (committed)
VITE_API_BASE_URL=http://localhost:3001
VITE_ENABLE_DEBUG=true
VITE_ENABLE_ANALYTICS=false
VITE_ENABLE_MOCK_API=true
```

```bash
# .env.staging (committed)
VITE_API_BASE_URL=https://api.staging.example.com
VITE_ENABLE_DEBUG=true
VITE_ENABLE_ANALYTICS=false
VITE_ENABLE_MOCK_API=false
```

```bash
# .env.production (committed)
VITE_API_BASE_URL=https://api.example.com
VITE_ENABLE_DEBUG=false
VITE_ENABLE_ANALYTICS=true
VITE_ENABLE_MOCK_API=false
```

```bash
# .env.local (gitignored - personal overrides)
VITE_API_BASE_URL=http://localhost:5000
# Override any value locally without affecting others
```

### Important: Only VITE_ Prefixed Variables Are Exposed

Vite only exposes variables prefixed with `VITE_` to client-side code. This prevents accidentally leaking server-side secrets.

```bash
# .env
VITE_API_URL=https://api.example.com   # Exposed to client code
SECRET_API_KEY=sk_1234                  # NOT exposed (no VITE_ prefix)
DATABASE_URL=postgres://...             # NOT exposed (no VITE_ prefix)
```

---

## import.meta.env Typed Access

Vite injects environment variables through `import.meta.env`. Add TypeScript types for autocomplete and type safety.

### Type Declaration

```typescript
// env.d.ts (in project root or src/)
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_APP_NAME: string;
  readonly VITE_API_BASE_URL: string;
  readonly VITE_DEFAULT_LOCALE: string;
  readonly VITE_PAGINATION_SIZE: string;
  readonly VITE_ENABLE_DEBUG: string;
  readonly VITE_ENABLE_ANALYTICS: string;
  readonly VITE_ENABLE_MOCK_API: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

### Typed Config Module

Centralize environment access in a single config module instead of scattering `import.meta.env` across the codebase:

```typescript
// config/environment.ts

function requireEnv(key: string): string {
  const value = import.meta.env[key];
  if (value === undefined || value === '') {
    throw new Error(`Missing required environment variable: ${key}`);
  }
  return value;
}

function envBoolean(key: string, defaultValue: boolean = false): boolean {
  const value = import.meta.env[key];
  if (value === undefined) return defaultValue;
  return value === 'true';
}

function envNumber(key: string, defaultValue: number): number {
  const value = import.meta.env[key];
  if (value === undefined) return defaultValue;
  const parsed = parseInt(value, 10);
  return isNaN(parsed) ? defaultValue : parsed;
}

export const config = {
  appName: requireEnv('VITE_APP_NAME'),
  apiBaseUrl: requireEnv('VITE_API_BASE_URL'),
  defaultLocale: import.meta.env.VITE_DEFAULT_LOCALE ?? 'en',
  paginationSize: envNumber('VITE_PAGINATION_SIZE', 25),

  features: {
    debug: envBoolean('VITE_ENABLE_DEBUG'),
    analytics: envBoolean('VITE_ENABLE_ANALYTICS'),
    mockApi: envBoolean('VITE_ENABLE_MOCK_API'),
  },

  isDev: import.meta.env.DEV,
  isProd: import.meta.env.PROD,
  mode: import.meta.env.MODE,
} as const;
```

### Usage

```typescript
import { config } from '@/config/environment';

// Type-safe, centralized access
const api = createApiClient(config.apiBaseUrl);

if (config.features.analytics) {
  initializeAnalytics();
}

if (config.features.debug) {
  enableDevTools();
}
```

---

## Runtime Config Injection

Some configuration must change without rebuilding the application. Inject these values at deployment time.

### HTML Template Injection

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
  <script>
    // Injected by deployment pipeline - NOT bundled by Vite
    window.__RUNTIME_CONFIG__ = {
      apiBaseUrl: "__API_BASE_URL__",
      featureFlags: "__FEATURE_FLAGS__",
      sentryDsn: "__SENTRY_DSN__",
    };
  </script>
</head>
<body>
  <div id="app"></div>
  <script type="module" src="/src/main.ts"></script>
</body>
</html>
```

```bash
# Deployment script replaces placeholders
sed -i "s|__API_BASE_URL__|https://api.example.com|g" dist/index.html
sed -i "s|__FEATURE_FLAGS__|{\"beta\":true}|g" dist/index.html
sed -i "s|__SENTRY_DSN__|https://abc@sentry.io/123|g" dist/index.html
```

### Runtime Config Module

```typescript
// config/runtime.ts

interface RuntimeConfig {
  apiBaseUrl: string;
  featureFlags: Record<string, boolean>;
  sentryDsn: string;
}

function getRuntimeConfig(): RuntimeConfig {
  const raw = (window as any).__RUNTIME_CONFIG__;

  if (!raw || raw.apiBaseUrl.startsWith('__')) {
    // Placeholders not replaced - use development defaults
    return {
      apiBaseUrl: 'http://localhost:3001',
      featureFlags: { beta: true },
      sentryDsn: '',
    };
  }

  return {
    apiBaseUrl: raw.apiBaseUrl,
    featureFlags: typeof raw.featureFlags === 'string'
      ? JSON.parse(raw.featureFlags)
      : raw.featureFlags,
    sentryDsn: raw.sentryDsn,
  };
}

export const runtimeConfig = getRuntimeConfig();
```

---

## Environment-Specific API URLs

Configure API clients to use the correct URL per environment.

```typescript
// api/client.ts
import { config } from '@/config/environment';

export const apiClient = {
  baseURL: config.apiBaseUrl,

  async get<T>(path: string): Promise<T> {
    const response = await fetch(`${this.baseURL}${path}`);
    if (!response.ok) {
      throw new ApiError(response.status, await response.text());
    }
    return response.json();
  },

  async post<T>(path: string, body: unknown): Promise<T> {
    const response = await fetch(`${this.baseURL}${path}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    });
    if (!response.ok) {
      throw new ApiError(response.status, await response.text());
    }
    return response.json();
  },
};
```

---

## Feature Flags per Environment

Control which features are available in each environment.

### Build-Time Feature Flags

```bash
# .env.development
VITE_FEATURE_NEW_DASHBOARD=true
VITE_FEATURE_EXPORT_PDF=true
VITE_FEATURE_BETA_API=true

# .env.staging
VITE_FEATURE_NEW_DASHBOARD=true
VITE_FEATURE_EXPORT_PDF=true
VITE_FEATURE_BETA_API=false

# .env.production
VITE_FEATURE_NEW_DASHBOARD=false
VITE_FEATURE_EXPORT_PDF=true
VITE_FEATURE_BETA_API=false
```

```typescript
// config/features.ts
import { config } from '@/config/environment';

export const features = {
  newDashboard: config.features.newDashboard,
  exportPdf: config.features.exportPdf,
  betaApi: config.features.betaApi,
} as const;
```

```vue
<script setup lang="ts">
import { features } from '@/config/features';
</script>

<template>
  <NewDashboard v-if="features.newDashboard" />
  <LegacyDashboard v-else />

  <button v-if="features.exportPdf" @click="exportToPdf">
    Export PDF
  </button>
</template>
```

### Runtime Feature Flags

For flags that change without redeployment (A/B tests, gradual rollouts):

```typescript
// composables/useFeatureFlags.ts
import { ref, onMounted } from 'vue';
import { runtimeConfig } from '@/config/runtime';

export function useFeatureFlags() {
  const flags = ref<Record<string, boolean>>(runtimeConfig.featureFlags);

  // Optionally refresh flags from a remote endpoint
  async function refreshFlags() {
    try {
      const response = await fetch('/api/feature-flags');
      flags.value = await response.json();
    } catch {
      // Keep existing flags on failure
    }
  }

  return { flags, refreshFlags };
}
```

---

## Build-Time vs. Runtime Environment Variables

Choosing the wrong type leads to either unnecessary rebuilds or bloated bundles.

### Build-Time Variables (`import.meta.env`)

Values are **statically replaced** during the build. The bundler inlines the literal value.

```typescript
// Source code
const url = import.meta.env.VITE_API_BASE_URL;

// After build (the string is inlined)
const url = "https://api.example.com";
```

**Use for:**
- Values that affect dead code elimination (feature flags)
- Values that differ per build (not per deployment)
- Configuration that changes rarely

**Tradeoff:** Changing a value requires rebuilding the application.

### Runtime Variables (`window.__CONFIG__`)

Values are **injected at deployment time**. The JavaScript references a global variable.

```typescript
// Source code
const url = runtimeConfig.apiBaseUrl;

// After build (reference preserved, not inlined)
const url = window.__RUNTIME_CONFIG__.apiBaseUrl;
```

**Use for:**
- Values that change per deployment without rebuilding
- Secrets that should not be in the build artifact
- Configuration managed by operations teams
- A/B test flags and gradual rollouts

**Tradeoff:** Cannot participate in dead code elimination.

### Decision Matrix

| Variable Type | Build-Time | Runtime |
|---|---|---|
| API URLs | Good if one build per env | Better if one build, many envs |
| Feature flags (permanent) | Best (enables tree shaking) | Unnecessary |
| Feature flags (A/B test) | Cannot do | Required |
| Analytics keys | Either works | Better for security |
| App name/version | Best | Unnecessary |
| Debug mode | Best (enables dead code removal) | Unnecessary |

---

## Vite define for Compile-Time Constants

The `define` option in Vite replaces identifiers at compile time, enabling dead code elimination for constant expressions.

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import pkg from './package.json';

export default defineConfig(({ mode }) => ({
  define: {
    // Application metadata
    __APP_VERSION__: JSON.stringify(pkg.version),
    __BUILD_TIME__: JSON.stringify(new Date().toISOString()),
    __GIT_HASH__: JSON.stringify(process.env.GIT_HASH ?? 'dev'),

    // Compile-time feature flags (dead code eliminated when false)
    __FEATURE_ANALYTICS__: JSON.stringify(mode === 'production'),
    __FEATURE_DEV_TOOLS__: JSON.stringify(mode === 'development'),
  },
}));
```

```typescript
// env.d.ts - Type declarations for compile-time constants
declare const __APP_VERSION__: string;
declare const __BUILD_TIME__: string;
declare const __GIT_HASH__: string;
declare const __FEATURE_ANALYTICS__: boolean;
declare const __FEATURE_DEV_TOOLS__: boolean;
```

```typescript
// Usage
console.log(`App v${__APP_VERSION__} built at ${__BUILD_TIME__}`);

// This entire block is removed from production builds
if (__FEATURE_DEV_TOOLS__) {
  const { mountDevPanel } = await import('./dev/DevPanel');
  mountDevPanel();
}

// This is only included in production builds
if (__FEATURE_ANALYTICS__) {
  const { initAnalytics } = await import('./analytics');
  initAnalytics();
}
```

### define vs. import.meta.env

| Feature | `define` | `import.meta.env` |
|---|---|---|
| Source | `vite.config.ts` | `.env` files |
| Prefix requirement | None | Must start with `VITE_` |
| Type | Any JSON-serializable | Always `string` |
| Dead code elimination | Yes (when boolean) | Yes (for `DEV`/`PROD`) |
| Use case | Build metadata, feature flags | Environment-specific config |

---

## Best Practices

### DO

- Use `.env` files for environment-specific configuration with `VITE_` prefix
- Create a centralized config module instead of accessing `import.meta.env` directly
- Declare TypeScript types for all environment variables in `env.d.ts`
- Use `define` for compile-time constants that enable dead code elimination
- Use runtime config injection for values that must change without rebuilding
- Validate required environment variables at application startup
- Commit `.env.development`, `.env.staging`, and `.env.production` files

### DON'T

- Commit `.env.local` or any `.local` files (they are gitignored for a reason)
- Put secrets in `VITE_` prefixed variables (they are embedded in client-side JavaScript)
- Access `import.meta.env` throughout the codebase (centralize in a config module)
- Use runtime config when build-time config enables better optimization
- Assume environment variables are typed (they are always strings from `.env` files)
- Mix build-time and runtime config without clear documentation of which is which

---

## Guidelines

### Essential

- All environment-specific values in `.env` files, not hardcoded in source
- TypeScript type declarations for `ImportMetaEnv` in `env.d.ts`
- Centralized config module wrapping `import.meta.env` access
- `.env.local` files gitignored

### Recommended

- Typed helper functions for boolean and number environment variables
- Required variable validation at startup (fail fast on missing config)
- Separate `.env` files per environment (development, staging, production)
- Feature flags using `define` for compile-time dead code elimination

### Advanced

- Runtime config injection for deploy-time configuration changes
- Feature flag system supporting both build-time and runtime flags
- Automated environment variable documentation generated from `env.d.ts`
- CI validation that all required `VITE_` variables are set for each environment

---

## Benefits

- **Environment isolation** - same code behaves correctly in dev, staging, production
- **Type safety** - TypeScript catches typos in environment variable names
- **Security** - only `VITE_` prefixed variables exposed to client code
- **Optimization** - build-time flags enable dead code elimination
- **Flexibility** - runtime injection allows config changes without rebuilds
- **Centralized access** - single config module prevents scattered env references

---

## Related

- [vite-configuration.md](../vite-configuration.md) - Vite base configuration and plugins
- [configuration-files-vs-env-vars.md](../configuration-files-vs-env-vars.md) - When to use config files vs environment variables
- [vue-production-builds.md](./vue-production-builds.md) - Production build optimization and feature flags
