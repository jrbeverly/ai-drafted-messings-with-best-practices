# Vite Configuration

`keywords: vite, vite.config.ts, vue-plugin, env-variables, proxy, build-options, plugins, css, alias, optimizeDeps, ssr, dev-server`

> **Principle:** Vite is the default build tool for Vue 3 projects. Configure it explicitly, keep defaults where sensible, and document deviations so every team member and CI pipeline produces identical builds.

---

## Vite Config Basics

Every Vue project starts with a `vite.config.ts` at the project root. Use TypeScript for type-checked configuration and IDE autocompletion.

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import { resolve } from 'path';

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
    },
  },
});
```

Use `defineConfig` for type inference. Never use a plain object export — you lose autocompletion and type safety.

### Conditional Configuration

For environment-specific behavior, use the function form:

```ts
export default defineConfig(({ command, mode }) => {
  const isDev = command === 'serve';

  return {
    plugins: [vue()],
    build: {
      sourcemap: isDev,
    },
  };
});
```

`command` is `'serve'` during development and `'build'` for production. `mode` defaults to `'development'` or `'production'` but can be overridden with `--mode`.

---

## Vue Plugin Setup

The `@vitejs/plugin-vue` plugin handles `.vue` single-file components, `<script setup>`, and template compilation.

```ts
import vue from '@vitejs/plugin-vue';

export default defineConfig({
  plugins: [
    vue({
      script: {
        defineModel: true,
        propsDestructure: true,
      },
    }),
  ],
});
```

For JSX support, add `@vitejs/plugin-vue-jsx`:

```ts
import vue from '@vitejs/plugin-vue';
import vueJsx from '@vitejs/plugin-vue-jsx';

export default defineConfig({
  plugins: [vue(), vueJsx()],
});
```

---

## Environment Variables

Vite uses `.env` files for environment-specific values. Only variables prefixed with `VITE_` are exposed to client code.

### File Hierarchy

```
.env                # Loaded in all cases
.env.local          # Loaded in all cases, ignored by git
.env.[mode]         # Loaded only in specified mode
.env.[mode].local   # Loaded only in specified mode, ignored by git
```

Priority (highest to lowest): `.env.[mode].local` > `.env.[mode]` > `.env.local` > `.env`

### Defining Variables

```bash
# .env
VITE_API_BASE_URL=https://api.example.com
VITE_APP_TITLE=My Application

# Not exposed to client (no VITE_ prefix)
DATABASE_URL=postgres://localhost/mydb
```

### Accessing in Code

```ts
// Client code — only VITE_ prefixed variables
const apiUrl = import.meta.env.VITE_API_BASE_URL;
const mode = import.meta.env.MODE;       // 'development' | 'production'
const isDev = import.meta.env.DEV;       // boolean
const isProd = import.meta.env.PROD;     // boolean
const baseUrl = import.meta.env.BASE_URL; // from config.base
```

### Type Declarations

Declare types for custom env variables in `src/env.d.ts`:

```ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string;
  readonly VITE_APP_TITLE: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

---

## Proxy Configuration for Development

Proxy API requests to a backend during development to avoid CORS issues.

```ts
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:5000',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
      '/ws': {
        target: 'ws://localhost:5000',
        ws: true,
      },
    },
  },
});
```

This forwards `/api/users` to `http://localhost:5000/users` during development. The proxy only applies to the dev server — production builds require a real API URL or reverse proxy.

---

## Build Options

### Target Browsers

```ts
export default defineConfig({
  build: {
    target: 'es2020',           // Target modern browsers
    outDir: 'dist',             // Output directory
    assetsDir: 'assets',        // Assets subdirectory within outDir
    sourcemap: true,            // Generate source maps (or 'hidden' for error tracking)
    minify: 'esbuild',         // 'esbuild' (default, fast) or 'terser' (smaller, slower)
    cssMinify: true,            // Minify CSS
  },
});
```

### Chunk Splitting

Control how code is split into chunks for optimal loading:

```ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vue-vendor': ['vue', 'vue-router', 'pinia'],
          'ui-vendor': ['vuetify'],
          'utils': ['lodash-es', 'date-fns'],
        },
      },
    },
    chunkSizeWarningLimit: 500,  // Warn if chunk exceeds 500KB
  },
});
```

For dynamic chunk splitting based on module analysis:

```ts
manualChunks(id) {
  if (id.includes('node_modules')) {
    if (id.includes('vue') || id.includes('pinia')) {
      return 'vue-vendor';
    }
    return 'vendor';
  }
},
```

---

## Vite Plugins Ecosystem

Common plugins and their purposes:

```ts
import vue from '@vitejs/plugin-vue';
import vueDevTools from 'vite-plugin-vue-devtools';
import Components from 'unplugin-vue-components/vite';
import AutoImport from 'unplugin-auto-import/vite';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    vue(),

    // Vue DevTools integration
    vueDevTools(),

    // Auto-import Vue APIs (ref, computed, watch, etc.)
    AutoImport({
      imports: ['vue', 'vue-router', 'pinia'],
      dts: 'src/auto-imports.d.ts',
    }),

    // Auto-register components from directories
    Components({
      dirs: ['src/components'],
      dts: 'src/components.d.ts',
    }),

    // PWA support
    VitePWA({
      registerType: 'autoUpdate',
    }),
  ],
});
```

Only add plugins you actively use. Each plugin adds build complexity and potential failure points.

---

## CSS Configuration

### Preprocessors

Vite supports CSS preprocessors out of the box — just install the preprocessor package:

```bash
pnpm add -D sass
```

```ts
export default defineConfig({
  css: {
    preprocessorOptions: {
      scss: {
        additionalData: `@use "@/styles/variables" as *;`,
      },
    },
  },
});
```

### CSS Modules

CSS Modules are supported for any file ending in `.module.css` (or `.module.scss`):

```ts
export default defineConfig({
  css: {
    modules: {
      localsConvention: 'camelCase',  // Convert kebab-case to camelCase
      scopeBehaviour: 'local',         // Default scoping
    },
  },
});
```

```vue
<script setup lang="ts">
import styles from './MyComponent.module.scss';
</script>

<template>
  <div :class="styles.container">
    <h1 :class="styles.headerTitle">Hello</h1>
  </div>
</template>
```

### PostCSS

Configure PostCSS via `postcss.config.js` at the project root:

```js
// postcss.config.js
export default {
  plugins: {
    autoprefixer: {},
    ...(process.env.NODE_ENV === 'production' ? { cssnano: {} } : {}),
  },
};
```

---

## Resolve Aliases for Path Mapping

Aliases eliminate long relative imports like `../../../components/Button.vue`:

```ts
import { resolve } from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
      '@components': resolve(__dirname, 'src/components'),
      '@composables': resolve(__dirname, 'src/composables'),
      '@types': resolve(__dirname, 'src/types'),
      '@assets': resolve(__dirname, 'src/assets'),
    },
  },
});
```

Keep aliases synchronized with `tsconfig.json` paths — see [typescript-path-aliases.md](typescript-path-aliases.md).

---

## Dependency Pre-Bundling (optimizeDeps)

Vite pre-bundles dependencies with esbuild for faster dev server startup. Customize when needed:

```ts
export default defineConfig({
  optimizeDeps: {
    include: [
      'vue',
      'vue-router',
      'pinia',
      'axios',
    ],
    exclude: ['your-local-linked-package'],
    esbuildOptions: {
      target: 'es2020',
    },
  },
});
```

Use `include` to force pre-bundling of dependencies that Vite might miss (dynamically imported or deeply nested). Use `exclude` for packages you are developing locally with `npm link`.

---

## SSR Configuration Basics

For server-side rendering with Vue:

```ts
export default defineConfig({
  ssr: {
    noExternal: ['vuetify'],       // Bundle these for SSR
    external: ['express'],          // Keep as external in SSR
  },
  build: {
    ssr: true,                      // When building SSR bundle
    rollupOptions: {
      input: 'src/entry-server.ts',
    },
  },
});
```

Most Vue projects use Nuxt for SSR. Only configure Vite SSR directly for custom server setups.

---

## Dev Server Options

```ts
export default defineConfig({
  server: {
    port: 3000,                     // Dev server port
    strictPort: true,               // Fail if port is taken (don't auto-increment)
    host: true,                     // Listen on all addresses (for Docker/remote access)
    open: true,                     // Open browser on server start
    cors: true,                     // Enable CORS
    hmr: {
      overlay: true,                // Show error overlay
    },
    watch: {
      usePolling: true,             // Needed in some Docker/VM setups
    },
  },
  preview: {
    port: 4173,                     // Preview server port (for vite preview)
    strictPort: true,
  },
});
```

---

## Best Practices

### DO

- Use `defineConfig` for type-safe configuration
- Prefix all client-exposed env vars with `VITE_`
- Declare env var types in `src/env.d.ts`
- Keep resolve aliases in sync with `tsconfig.json` paths
- Set `strictPort: true` to catch port conflicts early
- Use `sourcemap: 'hidden'` in production for error tracking without exposing source
- Split vendor chunks to improve caching

### DON'T

- Don't expose secrets via `VITE_` prefixed variables — they are embedded in the bundle
- Don't rely on the dev proxy in production — configure your reverse proxy or API gateway
- Don't add plugins speculatively — each adds build-time overhead
- Don't use `terser` minification unless you need its specific features — `esbuild` is faster
- Don't put build configuration in `.env` files — use `vite.config.ts` for build settings
- Don't use CJS (`require()`) in `vite.config.ts` — Vite uses ESM natively

---

## Guidelines

### Essential

- Every project has a `vite.config.ts` with `defineConfig`
- Vue plugin is always the first plugin in the array
- Environment variables are typed in `src/env.d.ts`
- Path aliases match between Vite config and `tsconfig.json`

### Recommended

- Configure dev server proxy for backend API integration
- Set up manual chunk splitting for vendor libraries
- Use `strictPort: true` in development
- Configure CSS preprocessor global imports via `additionalData`

### Advanced

- Custom plugin development for project-specific transforms
- SSR configuration for server-rendered applications
- Fine-tuned `optimizeDeps` for complex dependency graphs
- Conditional plugin loading based on build mode

---

## Benefits

- Native ESM dev server with instant hot module replacement
- Pre-configured Vue SFC support with no extra setup
- Type-safe configuration with full IDE support
- Built-in env variable management with client/server separation
- Optimized production builds via Rollup
- Extensible plugin ecosystem

---

## Related

- [vue-production-builds.md](vue-production-builds.md) — Production build optimization and deployment strategies
- [eslint-prettier-setup.md](eslint-prettier-setup.md) — Code quality tooling that integrates with Vite dev workflow
