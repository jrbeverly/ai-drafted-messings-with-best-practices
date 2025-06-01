# Configuration Files vs Environment Variables

`keywords: configuration, env-vars, environment-variables, import-meta-env, vite-env, zod-validation, runtime-config, build-time-config, secrets, twelve-factor, typed-config`

> **Principle:** Configuration determines behavior without changing code. Use environment variables for values that change per deployment, configuration files for values that change per project, and never store secrets in either location on the client side.

---

## When to Use Config Files vs Environment Variables

| Use Case | Config Files | Env Variables |
|----------|-------------|---------------|
| Build tool settings (Vite, ESLint, TypeScript) | Yes | No |
| Feature flags per environment | No | Yes |
| API base URLs | No | Yes |
| Design tokens, theme constants | Yes | No |
| Third-party SDK keys (public, client-side) | No | Yes |
| Secrets (API keys, passwords) | Never on client | Server-side only |
| Default application settings | Yes | No |
| Per-deployment overrides | No | Yes |

**Rule of thumb:** If the value changes between environments (dev/staging/prod), it is an environment variable. If the value is the same everywhere, it is a configuration file.

---

## Typed Configuration with TypeScript

For application settings that do not change per environment, use TypeScript files:

```ts
// src/config/app.config.ts
export const appConfig = {
  pagination: {
    defaultPageSize: 20,
    maxPageSize: 100,
  },
  debounce: {
    searchMs: 300,
    resizeMs: 150,
  },
  toast: {
    durationMs: 5000,
    maxVisible: 3,
  },
  routes: {
    loginPath: '/login',
    homePath: '/dashboard',
  },
} as const;

// Type is inferred and immutable
type AppConfig = typeof appConfig;
```

Benefits of TypeScript config files:
- Full type safety and autocompletion
- Compile-time validation
- Can import and compose with other modules
- Refactoring support from IDE

### JSON/YAML Configuration

For configuration consumed by multiple tools or languages:

```json
// config/features.json
{
  "enableDarkMode": true,
  "enableExperimentalSearch": false,
  "maxUploadSizeMb": 10
}
```

Import JSON directly in TypeScript (with `resolveJsonModule: true` in tsconfig):

```ts
import features from '@/config/features.json';

if (features.enableDarkMode) {
  // ...
}
```

---

## Vite Environment Variables (import.meta.env)

Vite replaces `import.meta.env` references at build time with actual values from `.env` files.

### How It Works

```
.env files → Vite build → String replacement in bundle → Static values in output
```

This means:
- Values are **embedded in the JavaScript bundle** at build time
- You cannot change them after building without rebuilding
- They are **visible to anyone** who inspects the bundle
- Only `VITE_`-prefixed variables are included

### Defining Variables

```bash
# .env (all environments)
VITE_APP_TITLE=My Application

# .env.development
VITE_API_BASE_URL=http://localhost:5000/api

# .env.staging
VITE_API_BASE_URL=https://staging-api.example.com/api

# .env.production
VITE_API_BASE_URL=https://api.example.com/api
```

### Type-Safe Access

```ts
// src/env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_APP_TITLE: string;
  readonly VITE_API_BASE_URL: string;
  readonly VITE_ENABLE_ANALYTICS: string;  // Always strings from .env
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

### Using in Code

```ts
// src/config/env.ts
// Centralize env var access for validation and transformation

export const env = {
  appTitle: import.meta.env.VITE_APP_TITLE,
  apiBaseUrl: import.meta.env.VITE_API_BASE_URL,
  enableAnalytics: import.meta.env.VITE_ENABLE_ANALYTICS === 'true',
  isDev: import.meta.env.DEV,
  isProd: import.meta.env.PROD,
  mode: import.meta.env.MODE,
} as const;
```

Centralizing env access in one file makes it easy to validate, transform, and mock in tests.

---

## Runtime vs Build-Time Configuration

### Build-Time (Vite env vars)

Values baked into the bundle during `vite build`:

```ts
// Replaced at build time — cannot change after build
const apiUrl = import.meta.env.VITE_API_BASE_URL;
```

**Use for:** API URLs, feature flags, analytics IDs, app version.

**Limitation:** Requires a new build for each environment. This is usually fine for CI/CD pipelines that build per environment.

### Runtime Configuration

Values loaded when the application starts in the browser:

```ts
// Option 1: Fetch config from server
const response = await fetch('/config.json');
const config = await response.json();

// Option 2: Inject via window global (server-rendered HTML)
declare global {
  interface Window {
    __APP_CONFIG__: {
      apiBaseUrl: string;
      featureFlags: Record<string, boolean>;
    };
  }
}
const config = window.__APP_CONFIG__;
```

**Use for:** Values that must change without rebuilding (multi-tenant apps, CDN-deployed SPAs, A/B testing).

### Server-Rendered Injection

```html
<!-- index.html (served by backend) -->
<script>
  window.__APP_CONFIG__ = {
    apiBaseUrl: "{{API_BASE_URL}}",
    tenantId: "{{TENANT_ID}}",
  };
</script>
```

The backend replaces `{{API_BASE_URL}}` with the actual value at request time. This allows one build artifact to run in multiple environments.

### Choosing Between Them

```
Need different values per environment?
  ├── Yes → Can you rebuild per environment?
  │     ├── Yes → Build-time (VITE_ env vars) ✓ Simpler
  │     └── No  → Runtime config (fetch or window injection)
  └── No → Config file (TypeScript, JSON)
```

---

## Config Validation with Zod

Validate configuration at application startup to fail fast on misconfiguration:

```ts
// src/config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  VITE_APP_TITLE: z.string().min(1, 'App title is required'),
  VITE_API_BASE_URL: z.string().url('API base URL must be a valid URL'),
  VITE_ENABLE_ANALYTICS: z
    .enum(['true', 'false'])
    .transform((val) => val === 'true'),
  MODE: z.enum(['development', 'staging', 'production']),
});

// Validate at import time — app fails to start if invalid
const parsed = envSchema.parse({
  VITE_APP_TITLE: import.meta.env.VITE_APP_TITLE,
  VITE_API_BASE_URL: import.meta.env.VITE_API_BASE_URL,
  VITE_ENABLE_ANALYTICS: import.meta.env.VITE_ENABLE_ANALYTICS,
  MODE: import.meta.env.MODE,
});

export const env = {
  appTitle: parsed.VITE_APP_TITLE,
  apiBaseUrl: parsed.VITE_API_BASE_URL,
  enableAnalytics: parsed.VITE_ENABLE_ANALYTICS,
  mode: parsed.MODE,
  isDev: import.meta.env.DEV,
  isProd: import.meta.env.PROD,
} as const;

export type Env = typeof env;
```

### Runtime Config Validation

```ts
// src/config/runtime.ts
import { z } from 'zod';

const runtimeConfigSchema = z.object({
  apiBaseUrl: z.string().url(),
  featureFlags: z.record(z.boolean()).default({}),
  maxUploadSizeMb: z.number().positive().default(10),
});

export type RuntimeConfig = z.infer<typeof runtimeConfigSchema>;

export async function loadRuntimeConfig(): Promise<RuntimeConfig> {
  const response = await fetch('/config.json');

  if (!response.ok) {
    throw new Error(`Failed to load runtime config: ${response.status}`);
  }

  const raw = await response.json();
  return runtimeConfigSchema.parse(raw);
}
```

Zod validation provides:
- Type-safe parsed output
- Clear error messages on misconfiguration
- Default values for optional settings
- Transformation (string to boolean, string to number)

---

## Environment-Specific Configuration

### Layered Configuration Pattern

```
Base config (always applied)
  └── Environment override (dev/staging/prod)
       └── Local override (developer-specific, gitignored)
```

### Implementation

```ts
// src/config/index.ts
import { env } from './env';
import { appConfig } from './app.config';

// Environment-specific overrides
const environmentOverrides: Record<string, Partial<typeof appConfig>> = {
  development: {
    debounce: {
      searchMs: 0,  // No debounce in dev for faster feedback
      resizeMs: 0,
    },
  },
  production: {
    pagination: {
      defaultPageSize: 25,
      maxPageSize: 200,
    },
  },
};

export const config = {
  ...appConfig,
  ...environmentOverrides[env.mode],
};
```

### .env File Layering

```
.env                    # Base values (committed)
.env.development        # Dev overrides (committed)
.env.staging            # Staging overrides (committed)
.env.production         # Prod overrides (committed)
.env.local              # Personal overrides (gitignored)
.env.development.local  # Personal dev overrides (gitignored)
```

Vite loads these in order, with later files overriding earlier ones. `.local` files are for developer-specific values and must be in `.gitignore`.

---

## Secrets Handling

### The Rule: Never Store Secrets in Frontend Code

Frontend code runs in the user's browser. Everything in the bundle is visible. There is no way to hide secrets in client-side code.

```ts
// NEVER DO THIS
const VITE_SECRET_API_KEY = import.meta.env.VITE_SECRET_API_KEY;
// This is embedded in the JavaScript bundle and visible to anyone

// NEVER DO THIS EITHER
const config = {
  databasePassword: 'my-secret-password',
};
// Visible in source code, version control, and bundle
```

### What Belongs Where

| Value | Location | Example |
|-------|----------|---------|
| Public API keys (client-side safe) | `.env` with `VITE_` prefix | Google Maps API key, Stripe publishable key |
| Secret API keys | Server-side only | Stripe secret key, database passwords |
| OAuth client IDs | `.env` with `VITE_` prefix | They are public by design |
| OAuth client secrets | Server-side only | Never expose to client |
| Feature flags | `.env` with `VITE_` prefix | Non-sensitive behavioral toggles |

### Proxy Pattern for Protected APIs

When the frontend needs to call an API that requires a secret key:

```
Browser → Your Backend (has secret) → External API
         ↑ Frontend only talks to your backend
```

```ts
// Frontend: calls your backend, no secrets needed
const response = await fetch('/api/weather?city=london');

// Backend: adds the secret API key
app.get('/api/weather', async (req, res) => {
  const apiKey = process.env.WEATHER_API_KEY; // Server-side env var
  const data = await fetch(
    `https://api.weather.com/v1?key=${apiKey}&city=${req.query.city}`
  );
  res.json(await data.json());
});
```

---

## Twelve-Factor App Config Principle Applied to Frontend

The twelve-factor app methodology states: "Store config in the environment."

For frontend applications, this translates to:

1. **Anything that varies between deploys is config** — API URLs, feature flags, analytics IDs
2. **Config is not code** — do not hardcode environment-specific values
3. **Strict separation** — the same build artifact should be deployable to any environment (aspiration for runtime config, practical with build-time for CI/CD)
4. **No config groups** — do not group config by environment name in code

```ts
// BAD: Config grouped by environment name in code
const config = {
  development: { apiUrl: 'http://localhost:5000' },
  production: { apiUrl: 'https://api.example.com' },
};

// GOOD: Config from environment, environment-agnostic code
const apiUrl = import.meta.env.VITE_API_BASE_URL;
```

The twelve-factor approach means the application code has no knowledge of specific environments. Configuration is injected from outside.

---

## Best Practices

### DO

- Centralize env var access in a single module (`src/config/env.ts`)
- Validate all configuration at startup with Zod or similar
- Type env vars in `src/env.d.ts` for IDE support
- Use `.env.local` for developer-specific overrides (gitignored)
- Proxy protected API calls through your backend
- Use build-time config (Vite env vars) when CI/CD builds per environment
- Use runtime config when one build must serve multiple environments

### DON'T

- Don't store secrets in `VITE_` variables — they are embedded in the bundle
- Don't hardcode environment-specific values in application code
- Don't use `process.env` in frontend code — use `import.meta.env` with Vite
- Don't scatter `import.meta.env` calls throughout the codebase — centralize them
- Don't skip validation — fail fast on misconfiguration rather than fail mysteriously at runtime
- Don't commit `.env.local` files to version control

---

## Guidelines

### Essential

- All `VITE_` env vars typed in `src/env.d.ts`
- Centralized config module (`src/config/env.ts`) instead of scattered `import.meta.env` calls
- `.env.local` files in `.gitignore`
- No secrets in client-side code or `VITE_` variables

### Recommended

- Zod validation for all env vars at application startup
- Separate build-time config (`import.meta.env`) from app config (TypeScript files)
- Document all required env vars in a `.env.example` file
- Runtime config pattern for CDN-deployed or multi-tenant applications

### Advanced

- Server-side HTML injection (`window.__APP_CONFIG__`) for zero-rebuild deployments
- Zod-validated runtime config loaded via `fetch('/config.json')`
- Feature flag service integration for dynamic configuration
- Config schema generation for documentation

---

## Benefits

- Clear separation between code and configuration
- Type-safe configuration with compile-time and runtime validation
- Fail-fast behavior catches misconfiguration before users see errors
- Security by design — secrets never reach the client bundle
- Environment-agnostic application code

---

## Related

- [vite-configuration.md](vite-configuration.md) — Vite `.env` file loading and `import.meta.env` behavior
- [vue-environment-configs.md](vue-environment-configs.md) — Vue-specific environment configuration patterns
