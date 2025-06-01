# Vue Router Patterns

Vue Router 4 patterns for Vue 3 applications. Type-safe routing, lazy loading, navigation guards, and route organization.

**Keywords:** vue-router, routing, navigation-guards, lazy-loading, route-meta, typed-routes, code-splitting, scroll-behavior

## Principle

Vue Router is the official router for Vue.js. Define routes declaratively, guard navigation with middleware, and split code by route for optimal performance. Keep route definitions centralized and type-safe.

## Router Setup with TypeScript

Install and configure Vue Router 4:

```bash
npm install vue-router@4
```

Create the router instance:

```ts
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import type { RouteRecordRaw } from 'vue-router'
import { routes } from './routes'

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) {
      return savedPosition
    }
    if (to.hash) {
      return { el: to.hash, behavior: 'smooth' }
    }
    return { top: 0 }
  }
})

export default router
```

Register in main entry:

```ts
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

createApp(App)
  .use(router)
  .mount('#app')
```

## Route Definitions

### Static Routes

Define simple routes with components:

```ts
// router/routes.ts
import type { RouteRecordRaw } from 'vue-router'
import HomeView from '@/views/HomeView.vue'

export const routes: RouteRecordRaw[] = [
  {
    path: '/',
    name: 'home',
    component: HomeView
  },
  {
    path: '/about',
    name: 'about',
    component: () => import('@/views/AboutView.vue')
  }
]
```

### Dynamic Routes

Routes with parameters:

```ts
const routes: RouteRecordRaw[] = [
  {
    path: '/users/:userId',
    name: 'user-detail',
    component: () => import('@/views/UserDetailView.vue'),
    props: true // pass route params as props
  },
  {
    path: '/posts/:year(\\d{4})/:month(\\d{2})',
    name: 'post-archive',
    component: () => import('@/views/PostArchiveView.vue')
  },
  {
    // Optional param
    path: '/search/:query?',
    name: 'search',
    component: () => import('@/views/SearchView.vue')
  }
]
```

### Nested Routes

Child routes rendered inside parent layout:

```ts
const routes: RouteRecordRaw[] = [
  {
    path: '/dashboard',
    name: 'dashboard',
    component: () => import('@/layouts/DashboardLayout.vue'),
    children: [
      {
        path: '',
        name: 'dashboard-home',
        component: () => import('@/views/dashboard/DashboardHomeView.vue')
      },
      {
        path: 'settings',
        name: 'dashboard-settings',
        component: () => import('@/views/dashboard/SettingsView.vue')
      },
      {
        path: 'users/:userId',
        name: 'dashboard-user',
        component: () => import('@/views/dashboard/UserView.vue'),
        children: [
          {
            path: 'profile',
            name: 'dashboard-user-profile',
            component: () => import('@/views/dashboard/UserProfileView.vue')
          },
          {
            path: 'activity',
            name: 'dashboard-user-activity',
            component: () => import('@/views/dashboard/UserActivityView.vue')
          }
        ]
      }
    ]
  }
]
```

Parent layout with `<router-view>`:

```vue
<!-- layouts/DashboardLayout.vue -->
<script setup lang="ts">
import SideNav from '@/components/SideNav.vue'
</script>

<template>
  <div class="dashboard-layout">
    <SideNav />
    <main class="dashboard-content">
      <router-view />
    </main>
  </div>
</template>
```

### Named Routes and Named Views

Multiple `<router-view>` outlets in one layout:

```ts
const routes: RouteRecordRaw[] = [
  {
    path: '/workspace',
    name: 'workspace',
    components: {
      default: () => import('@/views/WorkspaceView.vue'),
      sidebar: () => import('@/views/WorkspaceSidebar.vue'),
      header: () => import('@/views/WorkspaceHeader.vue')
    }
  }
]
```

```vue
<!-- App.vue -->
<template>
  <router-view name="header" />
  <div class="main">
    <router-view name="sidebar" />
    <router-view />
  </div>
</template>
```

## Lazy Loading and Code Splitting

Every route should use dynamic imports for automatic code splitting:

```ts
const routes: RouteRecordRaw[] = [
  {
    path: '/admin',
    name: 'admin',
    // Each dynamic import creates a separate chunk
    component: () => import('@/views/AdminView.vue')
  },
  {
    path: '/reports',
    name: 'reports',
    // Group related routes into the same chunk
    component: () => import(/* webpackChunkName: "reports" */ '@/views/ReportsView.vue')
  },
  {
    path: '/reports/monthly',
    name: 'reports-monthly',
    component: () => import(/* webpackChunkName: "reports" */ '@/views/MonthlyReportView.vue')
  }
]
```

Preload critical routes with router `prefetch`:

```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()

onMounted(() => {
  // Prefetch the dashboard route component after initial load
  const dashboardRoute = router.getRoutes().find(r => r.name === 'dashboard')
  if (dashboardRoute?.components?.default && typeof dashboardRoute.components.default === 'function') {
    (dashboardRoute.components.default as () => Promise<unknown>)()
  }
})
</script>
```

## Route Meta Fields

Attach custom data to routes for guards, breadcrumbs, and layout control:

```ts
// router/types.ts
import 'vue-router'

declare module 'vue-router' {
  interface RouteMeta {
    requiresAuth?: boolean
    roles?: string[]
    title?: string
    breadcrumb?: string
    layout?: 'default' | 'dashboard' | 'minimal'
    transition?: string
  }
}
```

```ts
// router/routes.ts
const routes: RouteRecordRaw[] = [
  {
    path: '/login',
    name: 'login',
    component: () => import('@/views/LoginView.vue'),
    meta: {
      title: 'Sign In',
      layout: 'minimal',
      requiresAuth: false
    }
  },
  {
    path: '/dashboard',
    name: 'dashboard',
    component: () => import('@/views/DashboardView.vue'),
    meta: {
      title: 'Dashboard',
      layout: 'dashboard',
      requiresAuth: true,
      breadcrumb: 'Dashboard'
    }
  },
  {
    path: '/admin/users',
    name: 'admin-users',
    component: () => import('@/views/AdminUsersView.vue'),
    meta: {
      title: 'User Management',
      requiresAuth: true,
      roles: ['admin'],
      breadcrumb: 'Users'
    }
  }
]
```

## Navigation Guards

### Global Before Guard

Runs before every navigation. Use for authentication and authorization:

```ts
// router/guards.ts
import type { Router } from 'vue-router'
import { useAuthStore } from '@/stores/useAuthStore'

export function registerGuards(router: Router): void {
  router.beforeEach((to, from) => {
    const authStore = useAuthStore()

    // Update document title
    if (to.meta.title) {
      document.title = `${to.meta.title} | My App`
    }

    // Check authentication
    if (to.meta.requiresAuth && !authStore.isAuthenticated) {
      return {
        name: 'login',
        query: { redirect: to.fullPath }
      }
    }

    // Check role-based access
    if (to.meta.roles && to.meta.roles.length > 0) {
      const hasRole = to.meta.roles.some(role =>
        authStore.user?.roles.includes(role)
      )
      if (!hasRole) {
        return { name: 'forbidden' }
      }
    }

    // Allow navigation
    return true
  })
}
```

Register guards after router creation:

```ts
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import { routes } from './routes'
import { registerGuards } from './guards'

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes
})

registerGuards(router)

export default router
```

### Per-Route Guards

Guard specific routes:

```ts
const routes: RouteRecordRaw[] = [
  {
    path: '/checkout',
    name: 'checkout',
    component: () => import('@/views/CheckoutView.vue'),
    beforeEnter: (to, from) => {
      const cartStore = useCartStore()
      if (cartStore.items.length === 0) {
        return { name: 'cart', query: { error: 'empty' } }
      }
      return true
    }
  },
  {
    path: '/onboarding',
    name: 'onboarding',
    component: () => import('@/views/OnboardingView.vue'),
    beforeEnter: (to, from) => {
      const userStore = useUserStore()
      if (userStore.hasCompletedOnboarding) {
        return { name: 'dashboard' }
      }
      return true
    }
  }
]
```

### In-Component Guards

Use `onBeforeRouteLeave` and `onBeforeRouteUpdate` inside components:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { onBeforeRouteLeave, onBeforeRouteUpdate, useRoute } from 'vue-router'

const route = useRoute()
const hasUnsavedChanges = ref(false)

// Warn before leaving with unsaved changes
onBeforeRouteLeave((to, from) => {
  if (hasUnsavedChanges.value) {
    const answer = window.confirm('You have unsaved changes. Leave anyway?')
    if (!answer) {
      return false
    }
  }
  return true
})

// React to param changes without unmounting the component
onBeforeRouteUpdate((to, from) => {
  if (to.params.userId !== from.params.userId) {
    // Load new user data when userId param changes
    loadUser(to.params.userId as string)
  }
})
</script>
```

### Global After Hook

Runs after navigation completes. Useful for analytics and loading indicators:

```ts
// router/guards.ts
export function registerAfterHooks(router: Router): void {
  router.afterEach((to, from, failure) => {
    if (!failure) {
      // Track page view
      analytics.trackPageView(to.fullPath)

      // Hide loading indicator
      loadingBar.finish()
    }
  })
}
```

## Typed Route Params with useRoute

Access route data with type safety:

```vue
<script setup lang="ts">
import { computed } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()

// Access params (always strings)
const userId = computed(() => route.params.userId as string)

// Parse numeric params
const page = computed(() => {
  const p = Number(route.params.page)
  return isNaN(p) ? 1 : p
})

// Access query parameters
const searchQuery = computed(() => {
  return (route.query.q as string) ?? ''
})

const sortOrder = computed(() => {
  return (route.query.sort as string) ?? 'desc'
})

// Access hash
const section = computed(() => route.hash.slice(1))

// Access meta
const pageTitle = computed(() => route.meta.title ?? 'Untitled')
</script>

<template>
  <div>
    <h1>{{ pageTitle }}</h1>
    <p>User ID: {{ userId }}</p>
    <p>Search: {{ searchQuery }}</p>
  </div>
</template>
```

## Programmatic Navigation with useRouter

Navigate from component logic:

```vue
<script setup lang="ts">
import { useRouter } from 'vue-router'

const router = useRouter()

// Navigate by path
function goHome() {
  router.push('/')
}

// Navigate by name with params
function goToUser(userId: string) {
  router.push({
    name: 'user-detail',
    params: { userId }
  })
}

// Navigate with query parameters
function search(query: string) {
  router.push({
    name: 'search',
    query: { q: query, page: '1' }
  })
}

// Replace current history entry (no back button)
function replaceWithDashboard() {
  router.replace({ name: 'dashboard' })
}

// Go back or forward
function goBack() {
  router.back()
}

function goForward() {
  router.forward()
}

// Go to specific history position
function goToPosition(n: number) {
  router.go(n)
}
</script>
```

## Navigation Failure Handling

Detect and handle failed navigations:

```vue
<script setup lang="ts">
import { useRouter, isNavigationFailure, NavigationFailureType } from 'vue-router'

const router = useRouter()

async function navigateToRoute(routeName: string) {
  try {
    const failure = await router.push({ name: routeName })

    if (failure) {
      if (isNavigationFailure(failure, NavigationFailureType.aborted)) {
        // Navigation was aborted (e.g., by a guard)
        console.warn('Navigation aborted:', failure.to.fullPath)
        showNotification('You do not have access to that page.')
      }
      if (isNavigationFailure(failure, NavigationFailureType.duplicated)) {
        // Already on the target route
        console.info('Already on this route.')
      }
    }
  } catch (error) {
    // Unexpected error during navigation
    console.error('Navigation error:', error)
  }
}
</script>
```

## Query Parameters Handling

Read and update query parameters reactively:

```vue
<script setup lang="ts">
import { computed, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'

const route = useRoute()
const router = useRouter()

// Read query params with defaults
const page = computed(() => Number(route.query.page) || 1)
const perPage = computed(() => Number(route.query.perPage) || 20)
const sortBy = computed(() => (route.query.sortBy as string) ?? 'createdAt')
const sortOrder = computed(() => (route.query.order as 'asc' | 'desc') ?? 'desc')
const filters = computed(() => {
  const raw = route.query.filter
  if (Array.isArray(raw)) return raw as string[]
  if (raw) return [raw as string]
  return []
})

// Update query params (preserves other params)
function updatePage(newPage: number) {
  router.push({
    query: { ...route.query, page: String(newPage) }
  })
}

function updateSort(field: string) {
  const newOrder = sortBy.value === field && sortOrder.value === 'asc' ? 'desc' : 'asc'
  router.push({
    query: { ...route.query, sortBy: field, order: newOrder, page: '1' }
  })
}

function clearFilters() {
  const { filter, ...rest } = route.query
  router.push({ query: rest })
}

// Watch for query changes
watch(
  () => route.query,
  (newQuery) => {
    // Refetch data when query changes
    fetchData({
      page: Number(newQuery.page) || 1,
      perPage: Number(newQuery.perPage) || 20,
      sortBy: (newQuery.sortBy as string) ?? 'createdAt',
      order: (newQuery.order as string) ?? 'desc'
    })
  },
  { immediate: true }
)
</script>
```

## Route Transitions

Animate route changes with Vue transitions:

```vue
<!-- App.vue -->
<script setup lang="ts">
import { useRoute } from 'vue-router'
import { computed } from 'vue'

const route = useRoute()
const transitionName = computed(() => route.meta.transition ?? 'fade')
</script>

<template>
  <router-view v-slot="{ Component, route: currentRoute }">
    <transition :name="transitionName" mode="out-in">
      <component :is="Component" :key="currentRoute.path" />
    </transition>
  </router-view>
</template>

<style>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.slide-enter-active,
.slide-leave-active {
  transition: transform 0.3s ease;
}
.slide-enter-from {
  transform: translateX(100%);
}
.slide-leave-to {
  transform: translateX(-100%);
}
</style>
```

Per-route transitions via meta:

```ts
const routes: RouteRecordRaw[] = [
  {
    path: '/dashboard',
    name: 'dashboard',
    component: () => import('@/views/DashboardView.vue'),
    meta: { transition: 'fade' }
  },
  {
    path: '/settings',
    name: 'settings',
    component: () => import('@/views/SettingsView.vue'),
    meta: { transition: 'slide' }
  }
]
```

## Scroll Behavior Configuration

Control scroll position on navigation:

```ts
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import type { RouterScrollBehavior } from 'vue-router'

const scrollBehavior: RouterScrollBehavior = (to, from, savedPosition) => {
  // Restore saved position on back/forward
  if (savedPosition) {
    return savedPosition
  }

  // Scroll to anchor
  if (to.hash) {
    return {
      el: to.hash,
      behavior: 'smooth',
      top: 80 // offset for fixed header
    }
  }

  // Scroll to top on new navigation, unless same route
  if (to.path !== from.path) {
    return { top: 0, behavior: 'smooth' }
  }

  // Stay in place for query-only changes
  return false
}

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
  scrollBehavior
})
```

Delayed scroll for async content:

```ts
const scrollBehavior: RouterScrollBehavior = (to, from, savedPosition) => {
  if (to.hash) {
    // Wait for async content to render before scrolling
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({ el: to.hash, behavior: 'smooth' })
      }, 300)
    })
  }
  return savedPosition ?? { top: 0 }
}
```

## 404 and Catch-All Routes

Handle unmatched routes:

```ts
const routes: RouteRecordRaw[] = [
  // ... all other routes first ...

  // Catch-all 404 (must be last)
  {
    path: '/:pathMatch(.*)*',
    name: 'not-found',
    component: () => import('@/views/NotFoundView.vue'),
    meta: {
      title: 'Page Not Found',
      layout: 'minimal'
    }
  }
]
```

404 component:

```vue
<!-- views/NotFoundView.vue -->
<script setup lang="ts">
import { useRoute, useRouter } from 'vue-router'

const route = useRoute()
const router = useRouter()

const attemptedPath = route.params.pathMatch
</script>

<template>
  <div class="not-found">
    <h1>404 - Page Not Found</h1>
    <p>The path <code>{{ attemptedPath }}</code> does not exist.</p>
    <button @click="router.push({ name: 'home' })">Go Home</button>
    <button @click="router.back()">Go Back</button>
  </div>
</template>
```

Nested catch-all for specific sections:

```ts
const routes: RouteRecordRaw[] = [
  {
    path: '/admin',
    component: () => import('@/layouts/AdminLayout.vue'),
    children: [
      {
        path: '',
        name: 'admin-home',
        component: () => import('@/views/admin/AdminHomeView.vue')
      },
      // Catch-all within admin section
      {
        path: ':pathMatch(.*)*',
        name: 'admin-not-found',
        component: () => import('@/views/admin/AdminNotFoundView.vue')
      }
    ]
  }
]
```

## Composables for Route Logic

### useBreadcrumbs

Generate breadcrumbs from route meta:

```ts
// composables/useBreadcrumbs.ts
import { computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import type { RouteLocationMatched } from 'vue-router'

interface Breadcrumb {
  label: string
  to: string | null
}

export function useBreadcrumbs() {
  const route = useRoute()
  const router = useRouter()

  const breadcrumbs = computed<Breadcrumb[]>(() => {
    return route.matched
      .filter((match: RouteLocationMatched) => match.meta.breadcrumb)
      .map((match: RouteLocationMatched, index: number, arr: RouteLocationMatched[]) => ({
        label: match.meta.breadcrumb as string,
        to: index === arr.length - 1
          ? null // current page is not a link
          : router.resolve({ name: match.name }).fullPath
      }))
  })

  return { breadcrumbs }
}
```

Usage:

```vue
<script setup lang="ts">
import { useBreadcrumbs } from '@/composables/useBreadcrumbs'

const { breadcrumbs } = useBreadcrumbs()
</script>

<template>
  <nav aria-label="Breadcrumb">
    <ol>
      <li v-for="(crumb, index) in breadcrumbs" :key="index">
        <router-link v-if="crumb.to" :to="crumb.to">{{ crumb.label }}</router-link>
        <span v-else aria-current="page">{{ crumb.label }}</span>
      </li>
    </ol>
  </nav>
</template>
```

### useRouteQuery

Type-safe query parameter access:

```ts
// composables/useRouteQuery.ts
import { computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'

export function useRouteQuery(key: string, defaultValue: string = '') {
  const route = useRoute()
  const router = useRouter()

  const value = computed({
    get: () => (route.query[key] as string) ?? defaultValue,
    set: (newValue: string) => {
      const query = { ...route.query }
      if (newValue === defaultValue || newValue === '') {
        delete query[key]
      } else {
        query[key] = newValue
      }
      router.push({ query })
    }
  })

  return value
}

export function useRouteQueryNumber(key: string, defaultValue: number = 0) {
  const route = useRoute()
  const router = useRouter()

  const value = computed({
    get: () => {
      const raw = Number(route.query[key])
      return isNaN(raw) ? defaultValue : raw
    },
    set: (newValue: number) => {
      const query = { ...route.query }
      if (newValue === defaultValue) {
        delete query[key]
      } else {
        query[key] = String(newValue)
      }
      router.push({ query })
    }
  })

  return value
}
```

Usage:

```vue
<script setup lang="ts">
import { useRouteQuery, useRouteQueryNumber } from '@/composables/useRouteQuery'

const search = useRouteQuery('q')
const page = useRouteQueryNumber('page', 1)
</script>

<template>
  <input v-model="search" placeholder="Search..." />
  <button @click="page--" :disabled="page <= 1">Previous</button>
  <span>Page {{ page }}</span>
  <button @click="page++">Next</button>
</template>
```

### useRequireAuth

Redirect unauthenticated users from within a component:

```ts
// composables/useRequireAuth.ts
import { watch } from 'vue'
import { useRouter } from 'vue-router'
import { useAuthStore } from '@/stores/useAuthStore'

export function useRequireAuth() {
  const authStore = useAuthStore()
  const router = useRouter()

  watch(
    () => authStore.isAuthenticated,
    (isAuthenticated) => {
      if (!isAuthenticated) {
        router.push({
          name: 'login',
          query: { redirect: router.currentRoute.value.fullPath }
        })
      }
    },
    { immediate: true }
  )

  return {
    user: authStore.user,
    isAuthenticated: authStore.isAuthenticated
  }
}
```

### useRouteTransitionDirection

Determine slide direction based on route depth:

```ts
// composables/useRouteTransition.ts
import { ref, watch } from 'vue'
import { useRoute } from 'vue-router'

export function useRouteTransition() {
  const route = useRoute()
  const transitionName = ref('fade')

  watch(
    () => route.path,
    (newPath, oldPath) => {
      if (!oldPath) {
        transitionName.value = 'fade'
        return
      }
      const newDepth = newPath.split('/').length
      const oldDepth = oldPath.split('/').length
      transitionName.value = newDepth > oldDepth ? 'slide-left' : 'slide-right'
    }
  )

  return { transitionName }
}
```

## Best Practices

**DO:**
- Use named routes for all navigation (resilient to path changes)
- Lazy load every route component with dynamic imports
- Augment `RouteMeta` interface with typed meta fields
- Centralize guards in dedicated files, not inline in route config
- Use `props: true` to pass route params as component props
- Handle navigation failures explicitly
- Define catch-all 404 route as the last route

**DON'T:**
- Navigate by hardcoded path strings throughout the codebase
- Import route components eagerly at the top of the routes file
- Store route state in Pinia (use `useRoute()` directly)
- Mutate `route.query` or `route.params` directly
- Use `beforeRouteEnter` in `<script setup>` (it is not supported; use global or per-route guards)
- Nest routes deeper than 3 levels without strong justification
- Forget to handle the async component loading state for lazy routes

## Guidelines

**Essential:**
- Type all route meta fields via module augmentation
- Protect authenticated routes with a global `beforeEach` guard
- Always provide a catch-all 404 route
- Use `<script setup>` with `useRoute()` and `useRouter()` composables

**Recommended:**
- Extract query parameter logic into composables (`useRouteQuery`)
- Generate breadcrumbs from route meta, not hardcoded arrays
- Configure scroll behavior for hash links and saved positions
- Group related routes into chunk names for code splitting
- Use route transitions with `mode="out-in"` to avoid layout thrashing

**Advanced:**
- Implement per-section catch-all routes for custom 404 pages
- Prefetch likely next routes after initial load
- Build composables for common route patterns (pagination, search, filters)
- Use `onBeforeRouteLeave` for unsaved changes warnings
- Implement navigation progress indicators using `beforeEach`/`afterEach`

## Benefits

Type safety. Augmented `RouteMeta` and typed params catch errors at compile time.

Code splitting. Lazy loading routes reduces initial bundle size.

Centralized access control. Guards enforce authentication and authorization in one place.

Predictable navigation. Named routes and programmatic navigation decouple components from URL structure.

Composability. Route-related logic extracts cleanly into reusable composables.

Scroll preservation. Configured scroll behavior provides expected UX on back/forward navigation.

## Related

- [composition-api-basics.md](./composition-api-basics.md) - Vue 3 Composition API fundamentals
- [vue-component-structure.md](./vue-component-structure.md) - Component organization patterns
- [vue-error-handling.md](./vue-error-handling.md) - Error handling in Vue applications
