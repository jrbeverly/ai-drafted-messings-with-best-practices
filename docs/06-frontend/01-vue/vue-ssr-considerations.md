# Vue SSR Considerations

Best practices for server-side rendering decisions with Vue 3. Covers SSR vs CSR vs SSG trade-offs, Nuxt 3 fundamentals, hydration, universal code patterns, and why Hugo + Vue SPA is often the better choice for solo developers.

`keywords: ssr, csr, ssg, nuxt, hydration, server-rendering, seo, universal, isomorphic, vue3, typescript`

## Principle

Server-side rendering is a deployment strategy, not a default. Every rendering mode (SSR, CSR, SSG) has specific costs in complexity, infrastructure, and maintenance burden. Choose the rendering mode that matches your actual requirements. For most solo-developer projects, a static site generator for public content combined with a Vue SPA for authenticated application logic delivers better outcomes than full SSR.

## Rendering Modes Compared

Three primary rendering strategies exist for Vue applications. Each makes different trade-offs between initial load performance, SEO capability, infrastructure cost, and development complexity.

### Client-Side Rendering (CSR)

The browser downloads a minimal HTML shell, then JavaScript builds the entire page in the browser.

```
Browser requests page
  -> Server sends empty HTML + JS bundle
  -> Browser downloads and parses JS
  -> Vue mounts and renders DOM
  -> Page becomes interactive
```

```html
<!-- index.html served for all routes -->
<!DOCTYPE html>
<html>
  <head>
    <title>My App</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

```ts
// src/main.ts
import { createApp } from "vue";
import App from "./App.vue";
import { createRouter, createWebHistory } from "vue-router";

const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: "/", component: () => import("./views/HomePage.vue") },
    {
      path: "/dashboard",
      component: () => import("./views/DashboardPage.vue"),
    },
  ],
});

const app = createApp(App);
app.use(router);
app.mount("#app");
```

When CSR is appropriate:

- Authenticated applications where SEO is irrelevant (dashboards, admin panels, internal tools)
- Applications behind login walls
- Single-page applications with complex client-side state
- Zero-cost-when-idle deployments (static files on S3 + CloudFront)

### Server-Side Rendering (SSR)

The server runs Vue on every request, generates full HTML, sends it to the browser, then Vue hydrates the page to make it interactive.

```
Browser requests page
  -> Server runs Vue, generates HTML
  -> Server sends full HTML + JS bundle
  -> Browser renders HTML immediately (First Contentful Paint)
  -> Browser downloads and parses JS
  -> Vue hydrates existing DOM (page becomes interactive)
```

```ts
// server.ts (simplified Node.js SSR entry)
import { createSSRApp } from "vue";
import { renderToString } from "vue/server-renderer";
import App from "./App.vue";

export async function render(url: string): Promise<string> {
  const app = createSSRApp(App);

  // Router, store, etc. must be created per-request
  const router = createRouter(/* ... */);
  app.use(router);

  await router.push(url);
  await router.isReady();

  const html = await renderToString(app);
  return html;
}
```

When SSR is appropriate:

- Public content pages where SEO is critical (marketing sites, blogs, product pages)
- Pages where first contentful paint directly impacts conversion
- Content that changes frequently enough that SSG rebuilds are impractical
- Social media previews requiring dynamic Open Graph tags

### Static Site Generation (SSG)

Pages are pre-rendered at build time into static HTML files. No server-side computation at request time.

```
Build step pre-renders all routes
  -> Deploy static HTML + JS to CDN
  -> Browser requests page
  -> CDN serves pre-built HTML
  -> Browser renders HTML immediately
  -> Vue hydrates existing DOM
```

```ts
// nuxt.config.ts (Nuxt 3 SSG mode)
export default defineNuxtConfig({
  // Pre-render all routes at build time
  routeRules: {
    "/": { prerender: true },
    "/about": { prerender: true },
    "/blog/**": { prerender: true },
  },
});
```

When SSG is appropriate:

- Content that changes infrequently (docs, blogs, marketing pages)
- Sites where build-time data fetching is acceptable
- Maximum CDN cacheability needed
- Zero-cost-when-idle hosting (static files on S3/CloudFront, Netlify, Vercel)

### Comparison Matrix

| Factor                 | CSR                   | SSR                          | SSG                  |
| ---------------------- | --------------------- | ---------------------------- | -------------------- |
| First Contentful Paint | Slow (waits for JS)   | Fast (HTML from server)      | Fast (HTML from CDN) |
| Time to Interactive    | Fast (after JS loads) | Delayed (hydration)          | Delayed (hydration)  |
| SEO                    | Poor (empty HTML)     | Good (full HTML)             | Good (full HTML)     |
| Infrastructure Cost    | Lowest (static files) | Highest (server per request) | Low (static files)   |
| Build Complexity       | Low                   | High                         | Medium               |
| Dynamic Content        | Full support          | Full support                 | Build-time only      |
| Hosting                | S3/CDN                | Node.js server or Lambda     | S3/CDN               |
| TTFB                   | Fast (CDN)            | Variable (server render)     | Fast (CDN)           |
| Scales to Zero         | Yes                   | Needs Lambda or server       | Yes                  |

## Nuxt 3 Overview

Nuxt 3 is the standard framework for Vue SSR. It provides file-based routing, auto-imports, server engine (Nitro), and hybrid rendering modes.

### Project Structure

```
nuxt-app/
├── app.vue                  # Root component
├── nuxt.config.ts           # Nuxt configuration
├── pages/                   # File-based routing
│   ├── index.vue            # -> /
│   ├── about.vue            # -> /about
│   └── blog/
│       ├── index.vue        # -> /blog
│       └── [slug].vue       # -> /blog/:slug
├── components/              # Auto-imported components
│   ├── AppHeader.vue
│   └── BlogPostCard.vue
├── composables/             # Auto-imported composables
│   └── useAuth.ts
├── server/                  # Server-only code (Nitro)
│   ├── api/
│   │   └── users.get.ts     # -> /api/users (GET)
│   └── middleware/
│       └── auth.ts
├── layouts/                 # Page layouts
│   └── default.vue
└── public/                  # Static assets
    └── favicon.ico
```

### Hybrid Rendering

Nuxt 3 supports per-route rendering rules. This allows mixing SSR, SSG, and CSR within a single application.

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // Static pages: pre-rendered at build time
    "/": { prerender: true },
    "/about": { prerender: true },
    "/blog/**": { prerender: true },

    // SSR pages: rendered on each request
    "/dashboard/**": { ssr: true },

    // CSR pages: client-side only
    "/admin/**": { ssr: false },

    // ISR: regenerate every 60 seconds
    "/products/**": { isr: 60 },

    // Cache headers
    "/api/**": {
      headers: { "cache-control": "max-age=60" },
    },
  },
});
```

## Hydration

Hydration is the process where Vue attaches event listeners and reactive state to server-rendered HTML in the browser. The server produces static HTML; hydration makes it interactive.

### How Hydration Works

```
1. Server renders HTML string from Vue components
2. HTML sent to browser (user sees content immediately)
3. Browser loads Vue JavaScript bundle
4. Vue walks the existing DOM tree
5. Vue matches DOM nodes to virtual DOM expectations
6. Vue attaches event listeners and reactivity
7. Page is now fully interactive
```

### Hydration Mismatches

A hydration mismatch occurs when the HTML rendered on the server does not match what Vue expects to render on the client. Vue will warn in development and may re-render the entire subtree in production.

Common causes of hydration mismatches:

```vue
<!-- MISMATCH: Date differs between server and client -->
<script setup lang="ts">
const now = new Date().toLocaleString();
</script>

<template>
  <p>Current time: {{ now }}</p>
</template>
```

```vue
<!-- FIX: Use ClientOnly wrapper or defer to client -->
<script setup lang="ts">
import { ref, onMounted } from "vue";

const now = ref<string>("");

onMounted(() => {
  now.value = new Date().toLocaleString();
});
</script>

<template>
  <p>Current time: {{ now || "Loading..." }}</p>
</template>
```

```vue
<!-- MISMATCH: Random values differ between server and client -->
<script setup lang="ts">
const randomId = Math.random().toString(36).substring(2);
</script>

<template>
  <div :id="randomId">Content</div>
</template>
```

```vue
<!-- FIX: Use useId() for SSR-stable unique identifiers (Vue 3.5+) -->
<script setup lang="ts">
import { useId } from "vue";

const id = useId();
</script>

<template>
  <div :id="id">Content</div>
</template>
```

Other common mismatch sources:

- Browser extensions modifying DOM before hydration
- Invalid HTML nesting (browser auto-corrects, server does not)
- Locale-dependent formatting differences between server and client
- Third-party scripts injecting DOM elements

## Data Fetching with onServerPrefetch

`onServerPrefetch` runs a function on the server before the component is rendered. The resolved data is serialized into the HTML payload so the client does not re-fetch it.

```vue
<script setup lang="ts">
import { ref, onServerPrefetch, onMounted } from "vue";

interface Article {
  id: string;
  title: string;
  content: string;
}

const article = ref<Article | null>(null);
const error = ref<string | null>(null);

async function fetchArticle(slug: string): Promise<Article> {
  const response = await fetch(`https://api.example.com/articles/${slug}`);
  if (!response.ok) {
    throw new Error(`Failed to fetch article: ${response.status}`);
  }
  return response.json();
}

const props = defineProps<{ slug: string }>();

// Runs on the server: data is included in the HTML payload
onServerPrefetch(async () => {
  try {
    article.value = await fetchArticle(props.slug);
  } catch (e) {
    error.value = e instanceof Error ? e.message : "Unknown error";
  }
});

// Runs on the client: only fetch if server did not provide data
onMounted(async () => {
  if (!article.value) {
    try {
      article.value = await fetchArticle(props.slug);
    } catch (e) {
      error.value = e instanceof Error ? e.message : "Unknown error";
    }
  }
});
</script>

<template>
  <div v-if="error" class="error">{{ error }}</div>
  <article v-else-if="article">
    <h1>{{ article.title }}</h1>
    <div v-html="article.content" />
  </article>
  <div v-else>Loading...</div>
</template>
```

In Nuxt 3, prefer the built-in `useAsyncData` or `useFetch` composables, which handle server/client data transfer automatically:

```vue
<!-- Nuxt 3: useAsyncData handles SSR data serialization -->
<script setup lang="ts">
interface Article {
  id: string;
  title: string;
  content: string;
}

const route = useRoute();

const { data: article, error } = await useAsyncData<Article>(
  `article-${route.params.slug}`,
  () => $fetch(`/api/articles/${route.params.slug}`),
);
</script>

<template>
  <div v-if="error" class="error">{{ error.message }}</div>
  <article v-else-if="article">
    <h1>{{ article.title }}</h1>
    <div v-html="article.content" />
  </article>
</template>
```

## useSSRContext

`useSSRContext` provides access to a shared context object during server-side rendering. Useful for passing data from components to the server entry point (e.g., setting status codes or collecting meta tags).

```vue
<script setup lang="ts">
import { useSSRContext } from "vue";

interface SSRContext {
  statusCode?: number;
  headers?: Record<string, string>;
  teleports?: Record<string, string>;
}

const props = defineProps<{ articleId: string }>();

const ssrContext = import.meta.env.SSR
  ? useSSRContext<SSRContext>()
  : undefined;

async function loadArticle() {
  const response = await fetch(`/api/articles/${props.articleId}`);

  if (!response.ok) {
    // Set 404 status code during server rendering
    if (ssrContext) {
      ssrContext.statusCode = 404;
    }
    return null;
  }

  return response.json();
}
</script>
```

```ts
// server entry: reads context after rendering
import { createSSRApp } from "vue";
import { renderToString } from "vue/server-renderer";
import App from "./App.vue";

export async function render(url: string) {
  const app = createSSRApp(App);
  const ctx: { statusCode?: number } = {};

  const html = await renderToString(app, ctx);

  return {
    html,
    statusCode: ctx.statusCode ?? 200,
  };
}
```

## Client-Only Components

Some components rely on browser APIs and cannot run on the server. Use `<ClientOnly>` (Nuxt) or conditional rendering to restrict these components to the client.

### Nuxt ClientOnly

```vue
<!-- Nuxt 3: ClientOnly built-in component -->
<template>
  <div>
    <h1>Analytics Dashboard</h1>

    <!-- This chart library accesses canvas, which is browser-only -->
    <ClientOnly>
      <ChartComponent :data="chartData" />

      <template #fallback>
        <div class="chart-placeholder">
          <p>Loading chart...</p>
        </div>
      </template>
    </ClientOnly>
  </div>
</template>
```

### Manual Client-Only Pattern (without Nuxt)

```vue
<script setup lang="ts">
import { ref, onMounted, defineAsyncComponent } from "vue";

// Lazy-load the component only on the client
const MapComponent = defineAsyncComponent(() => import("./MapComponent.vue"));

const isMounted = ref(false);

onMounted(() => {
  isMounted.value = true;
});
</script>

<template>
  <div>
    <h1>Store Locations</h1>

    <!-- Only render after client-side mount -->
    <component v-if="isMounted" :is="MapComponent" :locations="stores" />
    <div v-else class="map-placeholder">
      <p>Loading map...</p>
    </div>
  </div>
</template>
```

### Nuxt .client.vue Convention

In Nuxt 3, suffix a component filename with `.client.vue` to restrict it to client-only rendering automatically:

```
components/
├── AppHeader.vue              # Universal (server + client)
├── AnalyticsTracker.client.vue # Client-only (auto-wrapped in ClientOnly)
└── SeoMetaTags.server.vue      # Server-only (stripped from client bundle)
```

## Browser API Guards

Server-side code runs in Node.js where `window`, `document`, `localStorage`, and other browser APIs do not exist. All access to browser globals must be guarded.

### Guard Patterns

```ts
// composables/useWindowSize.ts
import { ref, onMounted, onUnmounted } from "vue";

export function useWindowSize() {
  // Safe defaults for server rendering
  const width = ref(0);
  const height = ref(0);

  function update() {
    // Guard: only access window in browser
    if (typeof window !== "undefined") {
      width.value = window.innerWidth;
      height.value = window.innerHeight;
    }
  }

  // onMounted only runs in the browser
  onMounted(() => {
    update();
    window.addEventListener("resize", update);
  });

  onUnmounted(() => {
    if (typeof window !== "undefined") {
      window.removeEventListener("resize", update);
    }
  });

  return { width, height };
}
```

```ts
// composables/useLocalStorage.ts (SSR-safe version)
import { ref, watch } from "vue";
import type { Ref } from "vue";

export function useLocalStorage<T>(key: string, defaultValue: T): Ref<T> {
  const isClient = typeof window !== "undefined";

  // Read from localStorage only on client
  const stored = isClient ? localStorage.getItem(key) : null;
  const value = ref<T>(stored ? JSON.parse(stored) : defaultValue) as Ref<T>;

  // Write to localStorage only on client
  if (isClient) {
    watch(
      value,
      (newValue) => {
        localStorage.setItem(key, JSON.stringify(newValue));
      },
      { deep: true },
    );
  }

  return value;
}
```

### Nuxt Environment Helpers

```ts
// Nuxt 3 provides built-in environment detection
export function useClientOnly() {
  const nuxtApp = useNuxtApp();

  // import.meta.client is true only in the browser
  if (import.meta.client) {
    // Safe to access window, document, etc.
    console.log("Running in browser:", window.location.href);
  }

  // import.meta.server is true only during SSR
  if (import.meta.server) {
    // Safe to access Node.js APIs
    console.log("Running on server");
  }
}
```

### import.meta.env.SSR

Vue provides `import.meta.env.SSR` as a build-time flag. Bundlers tree-shake the dead branch, so browser APIs inside the `else` block are never included in server builds.

```ts
// This is tree-shaken at build time
if (import.meta.env.SSR) {
  // Server-only code path
} else {
  // Client-only code path: safe to use window, document, etc.
  document.title = "My Page";
}
```

## Universal Code Patterns

Universal (isomorphic) code runs identically on server and client. Follow these patterns to write code that works in both environments.

### Creating App Instances Per Request

On the server, every request must get a fresh app instance. Shared state between requests causes data leakage between users.

```ts
// app.ts - factory function, NOT a singleton
import { createSSRApp } from "vue";
import { createPinia } from "pinia";
import App from "./App.vue";
import { createAppRouter } from "./router";

export function createApp() {
  const app = createSSRApp(App);
  const pinia = createPinia();
  const router = createAppRouter();

  app.use(pinia);
  app.use(router);

  return { app, pinia, router };
}
```

```ts
// entry-server.ts
import { renderToString } from "vue/server-renderer";
import { createApp } from "./app";

export async function render(url: string) {
  // Fresh instance per request - no shared state
  const { app, router, pinia } = createApp();

  await router.push(url);
  await router.isReady();

  const html = await renderToString(app);

  // Serialize store state for client hydration
  const initialState = JSON.stringify(pinia.state.value);

  return { html, initialState };
}
```

```ts
// entry-client.ts
import { createApp } from "./app";

const { app, router, pinia } = createApp();

// Hydrate Pinia state from server-serialized data
if (window.__INITIAL_STATE__) {
  pinia.state.value = JSON.parse(window.__INITIAL_STATE__);
}

router.isReady().then(() => {
  app.mount("#app");
});
```

### Avoiding Module-Level Side Effects

```ts
// BAD: Module-level side effect. Runs once on server, shared across all requests.
const cache = new Map<string, unknown>();
let requestCount = 0;

export function useCache() {
  requestCount++; // Shared across all server requests!
  return { cache, requestCount };
}
```

```ts
// GOOD: State created per composable invocation
export function useCache() {
  const cache = ref(new Map<string, unknown>());
  const requestCount = ref(0);

  function increment() {
    requestCount.value++;
  }

  return { cache, requestCount, increment };
}
```

## SSR-Safe Composables

Composables used in SSR applications must handle both server and client environments correctly.

```ts
// composables/useMediaQuery.ts (SSR-safe)
import { ref, onMounted, onUnmounted, watchEffect } from "vue";

export function useMediaQuery(query: string) {
  // Default to false on server (no matchMedia available)
  const matches = ref(false);
  let mediaQuery: MediaQueryList | null = null;

  function update() {
    if (mediaQuery) {
      matches.value = mediaQuery.matches;
    }
  }

  onMounted(() => {
    mediaQuery = window.matchMedia(query);
    update();
    mediaQuery.addEventListener("change", update);
  });

  onUnmounted(() => {
    mediaQuery?.removeEventListener("change", update);
  });

  return { matches };
}
```

```ts
// composables/useClipboard.ts (SSR-safe)
import { ref } from "vue";

export function useClipboard() {
  const copied = ref(false);
  const isSupported = ref(false);

  // Check support only on client
  if (typeof navigator !== "undefined" && navigator.clipboard) {
    isSupported.value = true;
  }

  async function copy(text: string): Promise<void> {
    if (!isSupported.value) {
      console.warn("Clipboard API not supported");
      return;
    }

    try {
      await navigator.clipboard.writeText(text);
      copied.value = true;
      setTimeout(() => {
        copied.value = false;
      }, 2000);
    } catch (e) {
      console.error("Failed to copy text:", e);
    }
  }

  return { copy, copied, isSupported };
}
```

### Composable SSR Safety Checklist

```ts
// Template for SSR-safe composable
import { ref, onMounted, onUnmounted } from "vue";

export function useSsrSafeFeature() {
  // 1. Provide safe defaults for server rendering
  const value = ref<string>("default");

  // 2. Put browser API access inside onMounted
  onMounted(() => {
    // window, document, navigator are safe here
    value.value = window.navigator.userAgent;
  });

  // 3. Guard any non-lifecycle browser access
  function doClientThing() {
    if (typeof window === "undefined") return;
    window.scrollTo(0, 0);
  }

  // 4. Clean up on unmount
  onUnmounted(() => {
    // Cleanup only runs on client
  });

  return { value, doClientThing };
}
```

## State Serialization

When using SSR, server-side state (Pinia stores, fetched data) must be serialized into the HTML response and deserialized on the client to avoid duplicate data fetching.

```ts
// Server: serialize Pinia state into HTML
import { renderToString } from "vue/server-renderer";
import { createApp } from "./app";

export async function render(url: string) {
  const { app, pinia, router } = createApp();

  await router.push(url);
  await router.isReady();

  const html = await renderToString(app);

  // Serialize state with proper escaping
  const state = JSON.stringify(pinia.state.value)
    .replace(/</g, "\\u003c") // Prevent XSS via </script> injection
    .replace(/>/g, "\\u003e");

  const fullHtml = `
    <!DOCTYPE html>
    <html>
    <head><title>My App</title></head>
    <body>
      <div id="app">${html}</div>
      <script>window.__PINIA_STATE__ = ${state}</script>
      <script type="module" src="/src/entry-client.ts"></script>
    </body>
    </html>
  `;

  return fullHtml;
}
```

```ts
// Client: hydrate Pinia state from serialized data
import { createApp } from "./app";

const { app, pinia, router } = createApp();

// Restore server state before mounting
if (window.__PINIA_STATE__) {
  pinia.state.value = window.__PINIA_STATE__;
}

router.isReady().then(() => {
  app.mount("#app");
});
```

Nuxt 3 handles state serialization automatically through `useState`, `useAsyncData`, and Pinia integration. Manual serialization is only necessary for custom SSR setups.

## Meta Tags with SSR (useHead)

SSR enables dynamic meta tags that are visible to search engine crawlers and social media link previews. Use `@unhead/vue` (standalone) or the built-in Nuxt `useHead` composable.

```vue
<!-- Nuxt 3: useHead is auto-imported -->
<script setup lang="ts">
interface Article {
  title: string;
  description: string;
  image: string;
  publishedAt: string;
  author: string;
}

const route = useRoute();

const { data: article } = await useAsyncData<Article>(
  `article-${route.params.slug}`,
  () => $fetch(`/api/articles/${route.params.slug}`),
);

// Dynamic meta tags rendered on the server
useHead({
  title: () => article.value?.title ?? "Article Not Found",
  meta: [
    { name: "description", content: () => article.value?.description ?? "" },
    { property: "og:title", content: () => article.value?.title ?? "" },
    {
      property: "og:description",
      content: () => article.value?.description ?? "",
    },
    { property: "og:image", content: () => article.value?.image ?? "" },
    { property: "og:type", content: "article" },
    { name: "twitter:card", content: "summary_large_image" },
    { name: "twitter:title", content: () => article.value?.title ?? "" },
    {
      name: "twitter:description",
      content: () => article.value?.description ?? "",
    },
    { name: "twitter:image", content: () => article.value?.image ?? "" },
  ],
  link: [
    { rel: "canonical", href: `https://example.com/blog/${route.params.slug}` },
  ],
});
</script>

<template>
  <article v-if="article">
    <h1>{{ article.title }}</h1>
    <p>{{ article.description }}</p>
  </article>
</template>
```

### Standalone Vue (without Nuxt)

```ts
// main.ts
import { createHead } from "@unhead/vue";
import { createApp } from "vue";
import App from "./App.vue";

const app = createApp(App);
const head = createHead();
app.use(head);
app.mount("#app");
```

```vue
<script setup lang="ts">
import { useHead } from "@unhead/vue";

const props = defineProps<{ title: string; description: string }>();

useHead({
  title: () => `${props.title} | My App`,
  meta: [{ name: "description", content: () => props.description }],
});
</script>
```

## Hugo + Vue SPA: The solo developer Alternative

For solo developers, full SSR (Nuxt deployed on a Node.js server or Lambda) introduces infrastructure complexity, cold start latency, and operational burden that is rarely justified. A more effective architecture separates public-facing content from the authenticated application.

### The Architecture

```
Public content (marketing, blog, docs):
  Hugo (static site generator)
  -> Pre-built HTML at build time
  -> Deployed to S3 + CloudFront
  -> Zero infrastructure cost at idle
  -> Perfect SEO, instant page loads

Authenticated application (dashboard, admin):
  Vue SPA (client-side rendering)
  -> Static JS/CSS bundle
  -> Deployed to S3 + CloudFront
  -> API calls to Lambda backend
  -> Zero infrastructure cost at idle
  -> No SEO needed (behind login)
```

### Why This Beats Full SSR

```
Hugo + Vue SPA                         Full SSR (Nuxt on Lambda/Server)
───────────────────────────────────    ───────────────────────────────────
Static HTML for public pages            Node.js server or Lambda for every page
Zero runtime cost                       Per-request compute cost
No cold starts                          Lambda cold starts (1-3s)
No hydration mismatches on public       Hydration complexity everywhere
Hugo builds in <1 second                Nuxt builds are slower
Two simple deployments                  One complex deployment
Each part is independently simple       Everything is coupled
No Node.js server to maintain           Node.js runtime to manage
CDN serves everything                   Server renders, then CDN caches
```

### Implementation Example

```
project/
├── site/                    # Hugo static site (public content)
│   ├── hugo.toml
│   ├── content/
│   │   ├── _index.md        # Homepage
│   │   ├── about.md
│   │   └── blog/
│   │       ├── first-post.md
│   │       └── second-post.md
│   ├── layouts/
│   │   └── _default/
│   │       └── baseof.html
│   └── static/
│       └── images/
├── app/                     # Vue SPA (authenticated app)
│   └── PortalService/
│       ├── src/
│       │   ├── main.ts
│       │   ├── App.vue
│       │   ├── views/
│       │   │   ├── DashboardPage.vue
│       │   │   └── SettingsPage.vue
│       │   └── composables/
│       └── vite.config.ts
└── env/                     # Infrastructure
    └── portal/
        ├── cloudfront.tf     # Single CloudFront distribution
        ├── s3.tf             # S3 buckets for Hugo + SPA
        └── lambda.tf         # API Lambda functions
```

```hcl
# CloudFront routing: Hugo for public, SPA for app
resource "aws_cloudfront_distribution" "main" {
  # Public content from Hugo
  origin {
    domain_name = aws_s3_bucket.site.bucket_regional_domain_name
    origin_id   = "hugo-site"
  }

  # SPA from separate bucket
  origin {
    domain_name = aws_s3_bucket.app.bucket_regional_domain_name
    origin_id   = "vue-spa"
  }

  # Route /app/* to Vue SPA
  ordered_cache_behavior {
    path_pattern     = "/app/*"
    target_origin_id = "vue-spa"
    # SPA routing: always serve index.html for /app/* paths
    # Use CloudFront Functions or Lambda@Edge for SPA fallback
  }

  # Everything else goes to Hugo
  default_cache_behavior {
    target_origin_id = "hugo-site"
  }
}
```

### When Full SSR Is Actually Needed

Full SSR (Nuxt) is justified only when:

- Dynamic, personalized content must be SEO-indexable (e.g., user-generated content pages with unique URLs)
- Real-time data must appear in the initial HTML (stock prices, live scores)
- First contentful paint is a measurable business metric (e-commerce product pages where milliseconds impact conversion)
- You have dedicated infrastructure support (not a solo developer constraint)

For most solo-developer SaaS products, the public pages are static (marketing, docs, blog) and the application is behind authentication. Hugo + Vue SPA covers both cases with minimal complexity and zero idle cost.

## Best Practices

**DO:**

- Choose rendering mode based on actual SEO and performance requirements
- Use SSG or Hugo for public content that changes infrequently
- Use CSR (Vue SPA) for authenticated application pages
- Guard all browser API access with `typeof window !== 'undefined'`
- Create fresh app instances per server request to prevent state leakage
- Serialize state securely with HTML entity escaping to prevent XSS
- Use `onMounted` for all browser-specific initialization
- Provide meaningful fallback content in `<ClientOnly>` slots
- Use `useId()` (Vue 3.5+) for SSR-stable unique identifiers

**DON'T:**

- Default to SSR without verifying SEO is a real requirement
- Use `window`, `document`, or `localStorage` outside lifecycle hooks or guards
- Share module-level mutable state across server requests
- Generate random values or timestamps during server render without client synchronization
- Use SSR just for perceived performance (measure first, optimize second)
- Deploy Nuxt on always-on servers for a solo-developer project (use Lambda or SSG instead)
- Forget the `#fallback` template in `<ClientOnly>` (causes layout shift)
- Serialize state without escaping `</script>` sequences (XSS vulnerability)

## Guidelines

**Essential:**

- Guard all browser global access: `typeof window !== 'undefined'`
- Create per-request app instances in SSR server entry
- Handle hydration mismatches by deferring dynamic values to `onMounted`
- Escape serialized state to prevent script injection

**Recommended:**

- Use Nuxt 3 hybrid rendering with per-route `routeRules` if you need mixed rendering modes
- Prefer `useAsyncData` (Nuxt) or `onServerPrefetch` (vanilla Vue) for SSR data fetching
- Separate public content (Hugo/SSG) from authenticated app (SPA) for solo-developer projects
- Use `@unhead/vue` or Nuxt `useHead` for dynamic meta tags

**Advanced:**

- Implement ISR (incremental static regeneration) for content that updates periodically
- Use `useSSRContext` to communicate status codes and headers from components to the server
- Stream SSR responses with `renderToWebStream` for large pages
- Use `.client.vue` and `.server.vue` suffixes in Nuxt for environment-specific component variants

## Benefits

Informed rendering choice. Matching rendering mode to actual requirements avoids unnecessary complexity.

SEO where it matters. Server-rendered or pre-built HTML gives crawlers full content without JavaScript execution.

Zero idle cost. Static deployments (CSR, SSG, Hugo) scale to zero with no runtime infrastructure.

Operational simplicity. Hugo + Vue SPA eliminates Node.js server management for solo developers.

Fast first paint. SSR and SSG deliver visible content before JavaScript loads.

Progressive enhancement. `<ClientOnly>` and browser guards enable graceful degradation on the server.

## Related

- [vue-performance-optimization.md](./vue-performance-optimization.md) - Client-side performance patterns and lazy loading
- [vue-component-structure.md](./vue-component-structure.md) - SFC structure and component design
- [vue-error-handling.md](./vue-error-handling.md) - Error boundaries and fallback strategies
