# Bundling Third-Party Dependencies

`keywords: bundling, third-party, vendor, tree-shaking, CDN, npm, bundle-size, dynamic-import`

## Principle

**Self-host and bundle all third-party dependencies through your build pipeline rather than loading from external CDNs.** This gives you full control over availability, privacy, performance, and versioning while enabling tree shaking and bundle optimization.

---

## Why Self-Host Instead of CDN

External CDN links (`<script src="https://cdn.example.com/lib.js">`) were once considered best practice for caching benefits. Modern HTTP/2 and bundling tools have reversed this calculus.

### Reliability

CDN outages take your application down with no recourse. Third-party CDNs are a single point of failure outside your control.

```html
<!-- BAD: External dependency you cannot control -->
<script src="https://cdn.jsdelivr.net/npm/lodash@4.17.21/lodash.min.js"></script>

<!-- If jsdelivr goes down, your app breaks -->
```

When you bundle dependencies, your application is self-contained. If your hosting works, your application works.

### Privacy

Every external CDN request leaks user data (IP address, referrer, timing) to a third party. This creates GDPR/privacy compliance risks and exposes your users unnecessarily.

### Performance with HTTP/2

HTTP/2 multiplexes requests over a single connection. Loading many small files from your own origin is now efficient, eliminating the old argument that CDN-hosted shared libraries save connections. Bundled code from your origin benefits from a single TLS handshake and connection reuse.

---

## Importing from npm Instead of Script Tags

All third-party code should be installed as npm dependencies and imported through your module system.

```bash
# Install as a dependency
npm install date-fns
```

```typescript
// Import only what you need - enables tree shaking
import { format, parseISO } from 'date-fns';

const formatted = format(parseISO('2026-02-14'), 'MMMM do, yyyy');
```

This approach allows Vite to:
- Tree shake unused exports
- Apply minification
- Generate content-hashed filenames for cache busting
- Split into optimal chunks

---

## Vendor Chunk Configuration in Vite

By default, Vite bundles all node_modules into a single vendor chunk. For larger applications, splitting vendor code into logical chunks improves caching since vendor code changes less frequently than application code.

```typescript
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Core framework - changes rarely
          'vendor-vue': ['vue', 'vue-router', 'pinia'],

          // UI library - changes occasionally
          'vendor-ui': ['vuetify'],

          // Utility libraries - changes rarely
          'vendor-utils': ['date-fns', 'zod'],
        },
      },
    },
  },
});
```

A function-based approach gives finer control for larger dependency graphs:

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules')) {
            if (id.includes('vue') || id.includes('pinia') || id.includes('vue-router')) {
              return 'vendor-vue';
            }
            if (id.includes('vuetify')) {
              return 'vendor-ui';
            }
            // Everything else in a general vendor chunk
            return 'vendor-libs';
          }
        },
      },
    },
  },
});
```

---

## Subresource Integrity for Bundled Deps

Even when self-hosting, subresource integrity (SRI) hashes verify that files have not been tampered with between build and delivery. This is especially relevant when assets are served through a CDN layer like CloudFront.

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import sri from 'rollup-plugin-sri';

export default defineConfig({
  plugins: [
    sri({
      algorithms: ['sha384'],
    }),
  ],
});
```

The generated HTML will include integrity attributes:

```html
<script type="module" src="/assets/index-a1b2c3d4.js"
        integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/ux..." crossorigin="anonymous"></script>
```

---

## Tree Shaking Third-Party Libraries

Tree shaking removes unused code from your bundle. Its effectiveness depends on how the library is authored.

### Libraries That Tree Shake Well

Libraries with ES module exports allow bundlers to eliminate unused code:

```typescript
// date-fns uses per-function exports - only 'format' is bundled
import { format } from 'date-fns';

// lodash-es (not lodash) supports tree shaking
import { debounce } from 'lodash-es';
```

### Libraries That Do NOT Tree Shake

Libraries with side effects or CommonJS-only builds bundle everything:

```typescript
// BAD: Imports entire lodash (~70KB)
import _ from 'lodash';
const result = _.debounce(fn, 300);

// GOOD: Import specific function (~1KB)
import debounce from 'lodash-es/debounce';
const result = debounce(fn, 300);
```

### Verifying Tree Shaking

Use the Rollup visualizer to see what actually ends up in your bundle:

```typescript
// vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    visualizer({
      filename: 'dist/stats.html',
      open: true,
      gzipSize: true,
    }),
  ],
});
```

---

## Analyzing Bundle Impact

Before adding any dependency, check its size impact.

### Using bundlephobia

Check [bundlephobia.com](https://bundlephobia.com) before installing:

```bash
# Check size before adding
# Visit: https://bundlephobia.com/package/moment
# moment: 72.1KB minified + gzipped (heavy)

# Visit: https://bundlephobia.com/package/date-fns
# date-fns: 6.6KB minified + gzipped with tree shaking (light)
```

### Analyzing Your Existing Bundle

```bash
# Generate a visual bundle analysis
npx vite-bundle-visualizer

# Or use source-map-explorer on built output
npx source-map-explorer dist/assets/*.js
```

### Setting Bundle Size Budgets

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    // Warn when a chunk exceeds 250KB
    chunkSizeWarningLimit: 250,
  },
});
```

---

## Dynamic Import for Large Dependencies

Load heavy libraries only when needed using dynamic imports. This keeps the initial bundle small and defers cost to the point of use.

```typescript
// Heavy charting library loaded only when the chart component mounts
const ChartView = defineAsyncComponent(() => import('./ChartView.vue'));

// Inside a component - load on demand
async function generatePDF() {
  const { jsPDF } = await import('jspdf');
  const doc = new jsPDF();
  doc.text('Hello', 10, 10);
  doc.save('output.pdf');
}

// Conditional loading based on feature usage
async function initializeEditor() {
  if (userHasEditorAccess) {
    const { Editor } = await import('@tiptap/vue-3');
    return Editor;
  }
  return null;
}
```

Vite automatically code-splits dynamic imports into separate chunks that are fetched on demand.

---

## Replacing Heavy Dependencies with Lighter Alternatives

Periodically audit your dependencies and replace oversized libraries with lighter alternatives that cover your actual usage.

| Heavy Library | Size (min+gz) | Lighter Alternative | Size (min+gz) |
|---|---|---|---|
| `moment` | ~72KB | `date-fns` | ~6KB (tree-shaken) |
| `lodash` | ~70KB | `lodash-es` (tree-shaken) | ~2-5KB typical |
| `axios` | ~13KB | `fetch` (built-in) | 0KB |
| `numeral` | ~11KB | `Intl.NumberFormat` (built-in) | 0KB |
| `uuid` | ~3KB | `crypto.randomUUID()` (built-in) | 0KB |
| `classnames` | ~1KB | Template literals | 0KB |

```typescript
// BEFORE: axios for simple GET requests
import axios from 'axios';
const { data } = await axios.get('/api/users');

// AFTER: Built-in fetch (zero dependency cost)
const data = await fetch('/api/users').then(r => r.json());
```

```typescript
// BEFORE: uuid package
import { v4 as uuidv4 } from 'uuid';
const id = uuidv4();

// AFTER: Built-in crypto API
const id = crypto.randomUUID();
```

---

## Best Practices

### DO

- Install all dependencies through npm and import via ES modules
- Configure vendor chunk splitting for stable long-term caching
- Check bundlephobia before adding any new dependency
- Use dynamic imports for libraries over 50KB that are not needed at startup
- Prefer libraries that publish ES module builds for tree shaking
- Audit bundle composition quarterly with visualizer tools
- Use built-in browser APIs (fetch, Intl, crypto) when they cover your needs

### DON'T

- Load scripts from external CDNs in production
- Import entire utility libraries when you use a few functions
- Add dependencies without checking their bundle size impact
- Assume tree shaking works without verifying via bundle analysis
- Use CommonJS-only libraries when ES module alternatives exist
- Keep unused dependencies in package.json

---

## Guidelines

### Essential

- All third-party code bundled through Vite, no external CDN script tags
- Vendor chunks separated from application code for cache efficiency
- Bundle size checked before adding new dependencies

### Recommended

- Tree shaking verified for top-10 largest dependencies
- Dynamic imports used for heavy libraries not needed at page load
- Bundle visualizer run as part of release process

### Advanced

- Subresource integrity enabled for production assets
- Automated bundle size budgets enforced in CI
- Quarterly dependency audit replacing heavy libraries with lighter alternatives

---

## Benefits

- **Full availability control** - no external CDN dependency
- **User privacy preserved** - no third-party request leakage
- **Smaller bundles** - tree shaking and selective imports
- **Better caching** - content-hashed vendor chunks change rarely
- **Faster initial load** - dynamic imports defer non-critical code

---

## Related

- [vite-configuration.md](../vite-configuration.md) - Vite build and dev server setup
- [package-json-structure.md](../package-json-structure.md) - Dependency management and scripts
