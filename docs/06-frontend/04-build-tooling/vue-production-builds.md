# Vue Production Builds

`keywords: production, build, optimization, Vite, tree-shaking, chunks, source-maps, pre-rendering, feature-flags`

## Principle

**Production builds must be optimized for end-user performance, not developer convenience.** Strip development-only code, split output into cacheable chunks, minimize total bundle size, and verify build output before deployment.

---

## Vite Production Build Optimization

Vite uses Rollup under the hood for production builds, providing minification, tree shaking, and code splitting out of the box. Configure it to match your deployment requirements.

### Base Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';

export default defineConfig({
  plugins: [vue()],
  build: {
    // Target modern browsers for smaller output
    target: 'es2022',

    // Output directory
    outDir: 'dist',

    // Generate source maps as separate files (not inline)
    sourcemap: true,

    // Warn when a chunk exceeds this size (in KB)
    chunkSizeWarningLimit: 250,

    // Minification (esbuild is faster, terser produces smaller output)
    minify: 'esbuild',

    // CSS code splitting - each async chunk gets its own CSS file
    cssCodeSplit: true,

    // Clean output directory before build
    emptyOutDir: true,
  },
});
```

---

## Environment-Specific Builds

Use Vite's mode system to produce different builds for each environment.

```bash
# Development (default)
vite build --mode development

# Staging
vite build --mode staging

# Production
vite build --mode production
```

```typescript
// vite.config.ts
import { defineConfig, type UserConfig } from 'vite';

export default defineConfig(({ mode }) => {
  const config: UserConfig = {
    plugins: [vue()],
    build: {
      target: 'es2022',
      sourcemap: mode !== 'development',
    },
  };

  if (mode === 'production') {
    config.build = {
      ...config.build,
      // More aggressive minification for production
      minify: 'terser',
      terserOptions: {
        compress: {
          drop_console: true,  // Remove console.log statements
          drop_debugger: true, // Remove debugger statements
        },
      },
    };
  }

  return config;
});
```

---

## Removing Dev-Only Code

Development utilities, debug logging, and dev tools should not exist in production bundles.

### Using import.meta.env.DEV

Vite statically replaces `import.meta.env.DEV` and `import.meta.env.PROD` at build time. Dead code elimination removes unreachable branches.

```typescript
// This entire block is removed from production builds
if (import.meta.env.DEV) {
  console.log('Debug: user data loaded', userData);
  window.__DEBUG_STATE__ = store;
}

// Only this code remains in production
if (import.meta.env.PROD) {
  initializeErrorReporting();
}
```

### Dev-Only Plugins and Components

```typescript
// main.ts
import { createApp } from 'vue';
import App from './App.vue';

const app = createApp(App);

if (import.meta.env.DEV) {
  // Dev tools only loaded during development
  const { setupDevTools } = await import('./dev/devtools');
  setupDevTools(app);
}

app.mount('#app');
```

### Conditional Imports

```typescript
// router/index.ts
const routes: RouteRecordRaw[] = [
  // Production routes
  { path: '/', component: () => import('@/pages/HomePage.vue') },
  { path: '/dashboard', component: () => import('@/pages/DashboardPage.vue') },
];

if (import.meta.env.DEV) {
  // Dev-only routes for testing and debugging
  routes.push(
    { path: '/__dev/components', component: () => import('@/dev/ComponentGallery.vue') },
    { path: '/__dev/api-test', component: () => import('@/dev/ApiTestPage.vue') },
  );
}
```

---

## Conditional Feature Flags

Use compile-time constants to enable or disable features per environment.

```typescript
// vite.config.ts
export default defineConfig(({ mode }) => ({
  define: {
    __FEATURE_ANALYTICS__: JSON.stringify(mode === 'production'),
    __FEATURE_DEBUG_PANEL__: JSON.stringify(mode === 'development'),
    __FEATURE_BETA_DASHBOARD__: JSON.stringify(mode !== 'production'),
  },
}));
```

```typescript
// Declare types for compile-time constants
// env.d.ts
declare const __FEATURE_ANALYTICS__: boolean;
declare const __FEATURE_DEBUG_PANEL__: boolean;
declare const __FEATURE_BETA_DASHBOARD__: boolean;
```

```typescript
// Usage - dead code eliminated when flag is false
if (__FEATURE_ANALYTICS__) {
  initializeAnalytics();
}

if (__FEATURE_DEBUG_PANEL__) {
  mountDebugPanel();
}
```

---

## Tree Shaking Effectiveness

Tree shaking removes unused exports from the final bundle. Its effectiveness depends on code structure.

### What Tree Shakes Well

```typescript
// Named exports from ES modules - unused exports removed
export function formatDate(date: Date): string { /* ... */ }
export function formatCurrency(amount: number): string { /* ... */ }
// If only formatDate is imported, formatCurrency is eliminated
```

### What Prevents Tree Shaking

```typescript
// Side effects at module level prevent tree shaking
import './polyfills'; // Always included (side effect import)

// Default exports with large objects
export default {
  formatDate, formatCurrency, formatPhone, formatAddress,
  // All included even if only one is used
};

// Dynamic property access
const utils = { formatDate, formatCurrency };
const fn = utils[dynamicKey]; // Bundler cannot determine which is used
```

### Marking Packages as Side-Effect-Free

```json
// package.json
{
  "sideEffects": [
    "*.css",
    "*.vue"
  ]
}
```

This tells the bundler that all other files can be safely tree-shaken if their exports are unused.

---

## Chunk Strategy

### Manual Chunks for Stable Caching

Split output into chunks with different change frequencies for optimal caching:

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Framework core - changes on major version updates
          'vue-core': ['vue', 'vue-router', 'pinia'],

          // UI framework - changes occasionally
          'ui-lib': ['vuetify'],

          // Utilities - rarely change
          'utils': ['date-fns', 'zod'],
        },
      },
    },
  },
});
```

### Dynamic Imports for Route-Level Code Splitting

```typescript
// router/index.ts
const routes: RouteRecordRaw[] = [
  {
    path: '/',
    // Each route is a separate chunk, loaded on navigation
    component: () => import('@/pages/HomePage.vue'),
  },
  {
    path: '/settings',
    component: () => import('@/pages/SettingsPage.vue'),
  },
  {
    path: '/reports',
    // Heavy page with charts - loaded only when visited
    component: () => import('@/pages/ReportsPage.vue'),
  },
];
```

### Naming Chunks for Debugging

```typescript
// Named chunks appear with readable names in network tab
const ReportsPage = () => import(/* webpackChunkName: "reports" */ '@/pages/ReportsPage.vue');

// Vite also supports naming via rollupOptions
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        chunkFileNames: 'assets/[name]-[hash].js',
        entryFileNames: 'assets/[name]-[hash].js',
        assetFileNames: 'assets/[name]-[hash][extname]',
      },
    },
  },
});
```

---

## Analyzing Build Output

Always inspect what goes into your production bundle.

### Rollup Plugin Visualizer

```typescript
// vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    vue(),
    visualizer({
      filename: 'dist/bundle-stats.html',
      open: false,
      gzipSize: true,
      brotliSize: true,
      template: 'treemap', // or 'sunburst', 'network'
    }),
  ],
});
```

### Build Size Reporting

```bash
# After building, check output sizes
npx vite build

# Output shows:
# dist/assets/index-a1b2c3.js    145.23 kB │ gzip: 47.12 kB
# dist/assets/vue-core-d4e5f6.js  82.45 kB │ gzip: 31.89 kB
# dist/assets/vendor-ui-g7h8i9.js 210.67 kB │ gzip: 68.34 kB
```

### Automated Size Budget in CI

```yaml
# .gitea/workflows/build.yml
- name: Build
  run: npm run build

- name: Check bundle size
  run: |
    MAX_SIZE=500000  # 500KB gzipped total budget
    TOTAL=$(find dist/assets -name '*.js' -exec gzip -c {} \; | wc -c)
    if [ "$TOTAL" -gt "$MAX_SIZE" ]; then
      echo "Bundle size ($TOTAL bytes gzipped) exceeds budget ($MAX_SIZE bytes)"
      exit 1
    fi
```

---

## Source Maps in Production

Source maps enable debugging production errors without exposing source code to end users.

### Separate Source Map Files

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    // Generate .js.map files alongside .js files
    sourcemap: true,

    // Alternative: 'hidden' generates maps but does not add the
    // //# sourceMappingURL comment to the JS file.
    // Users cannot find maps, but error reporting tools can.
    sourcemap: 'hidden',
  },
});
```

### Deployment Strategy

```bash
# Deploy application files to public CDN/S3
aws s3 sync dist/ s3://app-bucket/ --exclude "*.map"

# Upload source maps to error reporting service only
sentry-cli releases files upload-sourcemaps dist/ --url-prefix '~/assets/'

# Or store maps in a private location for debugging
aws s3 sync dist/ s3://private-maps-bucket/ --include "*.map" --exclude "*"
```

### Hidden Source Maps

The `'hidden'` option generates source maps without the reference comment in the JavaScript files. Error monitoring services (Sentry, Datadog) can still use them, but browser DevTools will not automatically load them:

```typescript
export default defineConfig({
  build: {
    sourcemap: 'hidden',
  },
});
```

---

## Build Performance Optimization

Speed up the build process itself for faster CI pipelines and local development.

### Faster Minification

```typescript
export default defineConfig({
  build: {
    // esbuild is ~10-100x faster than terser
    // Use esbuild unless you need terser's advanced compression
    minify: 'esbuild',
  },
});
```

### Dependency Pre-Bundling

```typescript
export default defineConfig({
  optimizeDeps: {
    // Pre-bundle these dependencies for faster dev server startup
    include: ['vue', 'vue-router', 'pinia', 'vuetify'],
  },
});
```

### Build Caching

```bash
# In CI, cache node_modules and Vite's dependency cache
# .gitea/workflows/build.yml
- uses: actions/cache@v4
  with:
    path: |
      node_modules
      node_modules/.vite
    key: deps-${{ hashFiles('package-lock.json') }}
```

---

## Pre-Rendering Critical Routes

For pages that do not require authentication, pre-render the HTML at build time for instant first paint.

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';

// Using vite-ssg for static site generation of specific routes
export default defineConfig({
  plugins: [vue()],
  ssgOptions: {
    // Only pre-render public pages
    includedRoutes: [
      '/',
      '/pricing',
      '/about',
      '/docs',
    ],
    // Skip authenticated routes
    // Everything else renders client-side as normal SPA
  },
});
```

For simpler pre-rendering without SSG:

```typescript
// vite.config.ts
import { createHtmlPlugin } from 'vite-plugin-html';

export default defineConfig({
  plugins: [
    vue(),
    createHtmlPlugin({
      minify: true,
      inject: {
        data: {
          title: 'My Application',
          // Inject critical CSS or meta tags at build time
        },
      },
    }),
  ],
});
```

---

## Best Practices

### DO

- Set `build.target` to `es2022` or higher for modern browsers
- Use `import.meta.env.DEV` guards for development-only code
- Split vendor chunks by change frequency for stable caching
- Use dynamic imports for route-level code splitting
- Generate hidden source maps and upload to error monitoring services
- Analyze bundle output after adding new dependencies
- Run production builds in CI to catch build errors early

### DON'T

- Ship console.log or debugger statements to production
- Inline source maps in production builds (use separate files)
- Create a single monolithic bundle with no code splitting
- Skip bundle size analysis when adding large dependencies
- Use terser when esbuild is sufficient (slower builds for marginal gains)
- Pre-render pages that require authentication or dynamic data

---

## Guidelines

### Essential

- Production builds use `esbuild` or `terser` minification with console/debugger removal
- Route-level code splitting via dynamic imports in the router
- Source maps generated as separate files, not inlined

### Recommended

- Vendor chunk splitting with `manualChunks` for stable long-term caching
- Bundle size budgets enforced in CI pipeline
- Build output analyzed with rollup-plugin-visualizer on dependency changes
- Feature flags using `define` for compile-time dead code elimination

### Advanced

- Hidden source maps uploaded to error monitoring service only
- Pre-rendering of public marketing and documentation pages
- Automated bundle size regression tracking across pull requests
- Build performance profiling and dependency pre-bundling optimization

---

## Benefits

- **Smaller bundles** - tree shaking and dead code elimination remove unused code
- **Faster page loads** - code splitting loads only what the current route needs
- **Better caching** - stable vendor chunks stay cached across deployments
- **Production safety** - dev-only code guaranteed absent from production
- **Debuggable errors** - source maps available to monitoring without exposing source

---

## Related

- [vite-configuration.md](../vite-configuration.md) - Vite dev server and base build configuration
- [vue-environment-configs.md](./vue-environment-configs.md) - Environment variables and multi-environment builds
