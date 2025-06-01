# Vue Performance Optimization

Best practices for optimizing Vue 3 application performance. Rendering efficiency, reactivity tuning, bundle size reduction, and profiling techniques.

`keywords: vue, performance, v-once, v-memo, shallowRef, computed, virtual-scrolling, lazy-loading, keep-alive, tree-shaking, profiling`

## Principle

Performance optimization in Vue 3 starts with understanding the reactivity system. Every reactive dependency creates tracking overhead, every component re-render walks the virtual DOM, and every imported module increases bundle size. Optimize by reducing unnecessary reactivity, minimizing re-renders, loading code on demand, and measuring before and after every change.

## Static Content with v-once

Use `v-once` to render content exactly once. Vue skips the element and all its children during subsequent re-renders. This is appropriate for content that never changes after the initial render.

```vue
<script setup lang="ts">
import { ref } from 'vue'

const appVersion = ref('2.4.1')
const counter = ref(0)

function increment() {
  counter.value++
}
</script>

<template>
  <!-- This header renders once and is never patched again -->
  <header v-once>
    <h1>Application Dashboard</h1>
    <p>Version {{ appVersion }}</p>
    <p>Built with Vue 3 + TypeScript</p>
  </header>

  <!-- This section re-renders normally -->
  <section>
    <p>Counter: {{ counter }}</p>
    <button @click="increment">Increment</button>
  </section>
</template>
```

When to use `v-once`:
- Static text, headings, and labels that never change
- Legal notices, copyright footers, and version stamps
- Large blocks of markup with no dynamic bindings

When NOT to use `v-once`:
- Content that depends on reactive state that changes
- Components that receive updated props
- Elements with event handlers that depend on changing state

## Conditional Re-render Caching with v-memo

`v-memo` caches a template subtree and only re-renders when specified dependencies change. It is particularly effective inside `v-for` loops where only a few items change at a time.

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface ListItem {
  id: string
  name: string
  selected: boolean
}

const items = ref<ListItem[]>([
  { id: '1', name: 'Alpha', selected: false },
  { id: '2', name: 'Beta', selected: false },
  { id: '3', name: 'Gamma', selected: true },
])

function toggleSelection(id: string) {
  const item = items.value.find(i => i.id === id)
  if (item) {
    item.selected = !item.selected
  }
}
</script>

<template>
  <ul>
    <!--
      v-memo skips re-rendering this li unless item.selected changes.
      Without v-memo, every li re-renders when any item in the array changes.
    -->
    <li
      v-for="item in items"
      :key="item.id"
      v-memo="[item.selected]"
      :class="{ active: item.selected }"
      @click="toggleSelection(item.id)"
    >
      {{ item.name }} {{ item.selected ? '(selected)' : '' }}
    </li>
  </ul>
</template>
```

The dependency array `[item.selected]` tells Vue to reuse the cached vnode for this list item unless `item.selected` has changed since the last render. Use `v-memo` when you have large lists where individual item re-renders are expensive.

## Shallow Reactivity with shallowRef and shallowReactive

By default, `ref()` and `reactive()` deeply convert nested objects into reactive proxies. For large data structures where you only need reactivity at the top level, use `shallowRef` or `shallowReactive` to avoid deep proxy overhead.

### shallowRef

```vue
<script setup lang="ts">
import { shallowRef, triggerRef } from 'vue'

interface DataPoint {
  timestamp: number
  value: number
}

// Only .value assignment triggers reactivity, not nested mutations
const chartData = shallowRef<DataPoint[]>([])

// This WILL trigger a re-render (reassigning .value)
function replaceData(newData: DataPoint[]) {
  chartData.value = newData
}

// This will NOT trigger a re-render (mutating nested content)
function addPointSilently(point: DataPoint) {
  chartData.value.push(point)
  // Manually trigger if you need the update
  triggerRef(chartData)
}

// Preferred pattern: replace the entire array to trigger reactivity
function addPoint(point: DataPoint) {
  chartData.value = [...chartData.value, point]
}
</script>

<template>
  <div>
    <p>Data points: {{ chartData.length }}</p>
    <div v-for="point in chartData" :key="point.timestamp">
      {{ point.timestamp }}: {{ point.value }}
    </div>
  </div>
</template>
```

### shallowReactive

```vue
<script setup lang="ts">
import { shallowReactive } from 'vue'

interface TableState {
  rows: Record<string, unknown>[]
  columns: string[]
  sortField: string
  sortOrder: 'asc' | 'desc'
}

// Top-level properties are reactive, nested objects are NOT proxied
const tableState = shallowReactive<TableState>({
  rows: [],
  columns: ['name', 'email', 'role'],
  sortField: 'name',
  sortOrder: 'asc',
})

// This triggers reactivity (top-level property assignment)
function updateSort(field: string) {
  tableState.sortField = field
}

// This triggers reactivity (replacing the entire array reference)
function setRows(newRows: Record<string, unknown>[]) {
  tableState.rows = newRows
}
</script>

<template>
  <table>
    <thead>
      <tr>
        <th
          v-for="col in tableState.columns"
          :key="col"
          @click="updateSort(col)"
        >
          {{ col }}
          <span v-if="tableState.sortField === col">
            {{ tableState.sortOrder === 'asc' ? '▲' : '▼' }}
          </span>
        </th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="(row, index) in tableState.rows" :key="index">
        <td v-for="col in tableState.columns" :key="col">
          {{ row[col] }}
        </td>
      </tr>
    </tbody>
  </table>
</template>
```

When to use shallow reactivity:
- Large arrays of objects (hundreds or thousands of items)
- Data from external sources that is replaced wholesale, not mutated
- Integration with third-party libraries that manage their own state (chart libraries, map libraries)
- Read-heavy data structures where deep proxy creation is wasted overhead

## Computed Property Caching

Computed properties cache their result and only re-evaluate when their reactive dependencies change. Always prefer `computed` over methods for derived state.

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Product {
  id: string
  name: string
  price: number
  category: string
  inStock: boolean
}

const products = ref<Product[]>([])
const searchQuery = ref('')
const selectedCategory = ref<string | null>(null)
const showInStockOnly = ref(false)

// Computed caches the result. Re-evaluates only when
// products, searchQuery, selectedCategory, or showInStockOnly change.
const filteredProducts = computed(() => {
  let result = products.value

  if (searchQuery.value) {
    const query = searchQuery.value.toLowerCase()
    result = result.filter(p => p.name.toLowerCase().includes(query))
  }

  if (selectedCategory.value) {
    result = result.filter(p => p.category === selectedCategory.value)
  }

  if (showInStockOnly.value) {
    result = result.filter(p => p.inStock)
  }

  return result
})

// Derived from filteredProducts. Only recalculates when filteredProducts changes.
const totalValue = computed(() =>
  filteredProducts.value.reduce((sum, p) => sum + p.price, 0)
)

const averagePrice = computed(() => {
  if (filteredProducts.value.length === 0) return 0
  return totalValue.value / filteredProducts.value.length
})
</script>

<template>
  <div>
    <input v-model="searchQuery" placeholder="Search products..." />
    <label>
      <input type="checkbox" v-model="showInStockOnly" />
      In stock only
    </label>
    <p>{{ filteredProducts.length }} products, average price: ${{ averagePrice.toFixed(2) }}</p>
    <ul>
      <li v-for="product in filteredProducts" :key="product.id">
        {{ product.name }} - ${{ product.price }}
      </li>
    </ul>
  </div>
</template>
```

Why computed over methods: A method called in the template re-executes on every render. A computed property only re-evaluates when one of its tracked dependencies changes. For expensive filtering, sorting, or aggregation, computed properties avoid redundant work.

## Virtual Scrolling for Large Lists

When rendering hundreds or thousands of items, virtual scrolling renders only the visible items plus a small buffer. This keeps DOM node count constant regardless of list size.

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface RowItem {
  id: string
  name: string
  email: string
}

const allItems = ref<RowItem[]>([])
const ROW_HEIGHT = 48
const VISIBLE_COUNT = 20
const BUFFER = 5

const containerRef = ref<HTMLElement | null>(null)
const scrollTop = ref(0)

function handleScroll(event: Event) {
  const target = event.target as HTMLElement
  scrollTop.value = target.scrollTop
}

const startIndex = computed(() =>
  Math.max(0, Math.floor(scrollTop.value / ROW_HEIGHT) - BUFFER)
)

const endIndex = computed(() =>
  Math.min(allItems.value.length, startIndex.value + VISIBLE_COUNT + BUFFER * 2)
)

const visibleItems = computed(() =>
  allItems.value.slice(startIndex.value, endIndex.value)
)

const totalHeight = computed(() => allItems.value.length * ROW_HEIGHT)
const offsetY = computed(() => startIndex.value * ROW_HEIGHT)
</script>

<template>
  <div
    ref="containerRef"
    class="virtual-list-container"
    :style="{ height: `${VISIBLE_COUNT * ROW_HEIGHT}px`, overflow: 'auto' }"
    @scroll="handleScroll"
  >
    <div :style="{ height: `${totalHeight}px`, position: 'relative' }">
      <div :style="{ transform: `translateY(${offsetY}px)` }">
        <div
          v-for="item in visibleItems"
          :key="item.id"
          :style="{ height: `${ROW_HEIGHT}px` }"
          class="virtual-list-row"
        >
          {{ item.name }} - {{ item.email }}
        </div>
      </div>
    </div>
  </div>
</template>
```

For production use, prefer a library such as `@tanstack/vue-virtual` which handles variable row heights, horizontal scrolling, and edge cases:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useVirtualizer } from '@tanstack/vue-virtual'

interface RowItem {
  id: string
  name: string
  email: string
}

const items = ref<RowItem[]>([])
const parentRef = ref<HTMLElement | null>(null)

const virtualizer = useVirtualizer({
  count: items.value.length,
  getScrollElement: () => parentRef.value,
  estimateSize: () => 48,
  overscan: 5,
})
</script>

<template>
  <div ref="parentRef" style="height: 600px; overflow: auto;">
    <div
      :style="{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }"
    >
      <div
        v-for="virtualRow in virtualizer.getVirtualItems()"
        :key="virtualRow.key"
        :style="{
          position: 'absolute',
          top: 0,
          left: 0,
          width: '100%',
          height: `${virtualRow.size}px`,
          transform: `translateY(${virtualRow.start}px)`,
        }"
      >
        {{ items[virtualRow.index].name }} - {{ items[virtualRow.index].email }}
      </div>
    </div>
  </div>
</template>
```

## Lazy Loading Components with defineAsyncComponent

Load components on demand instead of including them in the initial bundle. This reduces the initial JavaScript payload and speeds up time to interactive.

```vue
<script setup lang="ts">
import { defineAsyncComponent, ref } from 'vue'

// Basic lazy loading
const HeavyChart = defineAsyncComponent(
  () => import('@/components/HeavyChart.vue')
)

// With loading and error states
const AdminDashboard = defineAsyncComponent({
  loader: () => import('@/components/AdminDashboard.vue'),
  loadingComponent: () => import('@/components/LoadingSpinner.vue'),
  errorComponent: () => import('@/components/ErrorFallback.vue'),
  delay: 200,       // Show loading after 200ms (avoids flicker for fast loads)
  timeout: 10000,   // Error after 10 seconds
})

const showChart = ref(false)
const isAdmin = ref(false)
</script>

<template>
  <div>
    <button @click="showChart = !showChart">Toggle Chart</button>

    <!-- Component JS is fetched only when showChart becomes true -->
    <HeavyChart v-if="showChart" :data="chartData" />

    <!-- Admin dashboard loads only for admin users -->
    <AdminDashboard v-if="isAdmin" />
  </div>
</template>
```

### Route-Level Lazy Loading

Combine with Vue Router for route-level code splitting:

```ts
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import type { RouteRecordRaw } from 'vue-router'

const routes: RouteRecordRaw[] = [
  {
    path: '/',
    component: () => import('@/views/HomePage.vue'),
  },
  {
    path: '/dashboard',
    component: () => import('@/views/DashboardPage.vue'),
  },
  {
    path: '/admin',
    component: () => import('@/views/AdminPage.vue'),
  },
  {
    path: '/reports',
    // Named chunk for easier debugging in network tab
    component: () => import(/* webpackChunkName: "reports" */ '@/views/ReportsPage.vue'),
  },
]

const router = createRouter({
  history: createWebHistory(),
  routes,
})

export default router
```

## Caching Component State with KeepAlive

`<KeepAlive>` caches inactive component instances instead of destroying them. When a user navigates away and returns, the component retains its state without re-initializing.

```vue
<script setup lang="ts">
import { ref, onActivated, onDeactivated } from 'vue'

const currentTab = ref<'users' | 'settings' | 'reports'>('users')
</script>

<template>
  <nav>
    <button @click="currentTab = 'users'">Users</button>
    <button @click="currentTab = 'settings'">Settings</button>
    <button @click="currentTab = 'reports'">Reports</button>
  </nav>

  <!-- Cache up to 5 component instances -->
  <KeepAlive :max="5">
    <component :is="currentTab === 'users' ? UserList
      : currentTab === 'settings' ? SettingsPanel
      : ReportsView"
    />
  </KeepAlive>
</template>
```

Use `onActivated` and `onDeactivated` lifecycle hooks in cached components:

```vue
<!-- UserList.vue -->
<script setup lang="ts">
import { ref, onActivated, onDeactivated } from 'vue'

const users = ref<User[]>([])
let refreshInterval: ReturnType<typeof setInterval> | null = null

onActivated(() => {
  // Resume polling when component becomes visible again
  refreshInterval = setInterval(fetchUsers, 30000)
  fetchUsers()
})

onDeactivated(() => {
  // Pause polling when component is cached but not visible
  if (refreshInterval) {
    clearInterval(refreshInterval)
    refreshInterval = null
  }
})

async function fetchUsers() {
  const response = await fetch('/api/users')
  users.value = await response.json()
}
</script>

<template>
  <ul>
    <li v-for="user in users" :key="user.id">{{ user.name }}</li>
  </ul>
</template>
```

Control which components are cached:

```vue
<template>
  <!-- Only cache UserList and SettingsPanel, not ReportsView -->
  <KeepAlive include="UserList,SettingsPanel">
    <component :is="currentView" />
  </KeepAlive>

  <!-- Cache everything except ReportsView -->
  <KeepAlive exclude="ReportsView">
    <component :is="currentView" />
  </KeepAlive>

  <!-- Cache at most 10 instances (LRU eviction) -->
  <KeepAlive :max="10">
    <component :is="currentView" />
  </KeepAlive>
</template>
```

## Avoiding Unnecessary Reactivity

Not every piece of data needs to be reactive. Constants, configuration objects, and data that never changes in the component lifecycle should remain plain values.

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

// These are constants. No need for ref() or reactive().
const PAGE_SIZE = 20
const API_BASE_URL = '/api/v2'
const STATUS_LABELS: Record<string, string> = {
  active: 'Active',
  inactive: 'Inactive',
  pending: 'Pending Review',
}

// Column definitions are static configuration
const columns = [
  { key: 'name', label: 'Name', sortable: true },
  { key: 'email', label: 'Email', sortable: true },
  { key: 'role', label: 'Role', sortable: false },
] as const

// Only the data that actually changes needs reactivity
const currentPage = ref(1)
const searchQuery = ref('')
const users = ref<User[]>([])

const offset = computed(() => (currentPage.value - 1) * PAGE_SIZE)
</script>

<template>
  <table>
    <thead>
      <tr>
        <th v-for="col in columns" :key="col.key">{{ col.label }}</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="user in users" :key="user.id">
        <td>{{ user.name }}</td>
        <td>{{ user.email }}</td>
        <td>{{ STATUS_LABELS[user.status] }}</td>
      </tr>
    </tbody>
  </table>
</template>
```

Common sources of unnecessary reactivity:
- Wrapping constants in `ref()` or `reactive()`
- Making entire API response objects deeply reactive when only a few fields change
- Using `reactive()` for objects that are replaced wholesale (use `shallowRef` instead)

## Key Attribute for Efficient Re-renders

The `:key` attribute helps Vue's virtual DOM diffing algorithm identify which elements have changed, been added, or been removed. Correct key usage prevents unnecessary DOM operations.

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Task {
  id: string
  title: string
  completed: boolean
}

const tasks = ref<Task[]>([])
const filterMode = ref<'all' | 'active' | 'completed'>('all')
</script>

<template>
  <!-- DO: Use a stable, unique identifier as key -->
  <TransitionGroup name="list">
    <div v-for="task in tasks" :key="task.id" class="task-item">
      <input type="checkbox" v-model="task.completed" />
      <span>{{ task.title }}</span>
    </div>
  </TransitionGroup>

  <!-- Force component re-initialization when filter changes -->
  <TaskListView :key="filterMode" :filter="filterMode" />
</template>
```

```vue
<!-- DO: Stable unique ID -->
<li v-for="user in users" :key="user.id">{{ user.name }}</li>

<!-- DON'T: Array index as key when list is reordered or filtered -->
<li v-for="(user, index) in users" :key="index">{{ user.name }}</li>

<!-- DO: Force fresh component instance on route change -->
<router-view :key="$route.fullPath" />

<!-- DO: Force re-creation when switching between similar components -->
<UserForm v-if="mode === 'create'" :key="'create'" />
<UserForm v-else :key="editingUserId" :user-id="editingUserId" />
```

When `:key` changes on a component, Vue destroys the old instance and creates a new one. This is useful when you want a clean reset of component state rather than patching the existing instance.

## Functional Components

For leaf components that render markup with no internal state, lifecycle hooks, or methods, a simple function returning a render template is the lightest option. In Vue 3, all `<script setup>` components are already compiled efficiently, so the main benefit of functional components is communicating intent: this component is stateless.

```vue
<!-- StaticBadge.vue - stateless presentational component -->
<script setup lang="ts">
interface Props {
  label: string
  color?: 'green' | 'red' | 'yellow' | 'gray'
}

withDefaults(defineProps<Props>(), {
  color: 'gray',
})
</script>

<template>
  <span :class="['badge', `badge-${color}`]">{{ label }}</span>
</template>

<style scoped>
.badge {
  padding: 0.125rem 0.5rem;
  border-radius: 9999px;
  font-size: 0.75rem;
  font-weight: 600;
}
.badge-green { background: #dcfce7; color: #166534; }
.badge-red { background: #fee2e2; color: #991b1b; }
.badge-yellow { background: #fef9c3; color: #854d0e; }
.badge-gray { background: #f3f4f6; color: #374151; }
</style>
```

For programmatic render functions without templates:

```ts
// components/RenderIcon.ts
import { h } from 'vue'
import type { FunctionalComponent } from 'vue'

interface IconProps {
  name: string
  size?: number
}

const RenderIcon: FunctionalComponent<IconProps> = (props) => {
  return h('svg', {
    width: props.size ?? 24,
    height: props.size ?? 24,
    class: `icon icon-${props.name}`,
  })
}

RenderIcon.props = {
  name: { type: String, required: true },
  size: { type: Number, default: 24 },
}

export default RenderIcon
```

Keep components in `<script setup>` SFC format unless you have a specific reason to use a render function. Vue 3's compiler optimizations (static hoisting, patch flags) work on templates, not render functions.

## Bundle Size Analysis and Tree Shaking

### Analyze Bundle Size

Use `rollup-plugin-visualizer` or `webpack-bundle-analyzer` to inspect what ends up in your production bundle:

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { visualizer } from 'rollup-plugin-visualizer'

export default defineConfig({
  plugins: [
    vue(),
    visualizer({
      filename: 'dist/stats.html',
      open: true,
      gzipSize: true,
    }),
  ],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Separate vendor chunks for better caching
          'vue-vendor': ['vue', 'vue-router', 'pinia'],
          'query-vendor': ['@tanstack/vue-query'],
        },
      },
    },
  },
})
```

### Tree Shaking

Import only what you use. Named imports enable tree shaking; default imports of large modules do not.

```ts
// DO: Named imports are tree-shakeable
import { format, parseISO } from 'date-fns'
import { debounce } from 'lodash-es'
import { useVirtualizer } from '@tanstack/vue-virtual'

// DON'T: Default import pulls the entire library
import _ from 'lodash'
import dayjs from 'dayjs'       // dayjs is small, but the pattern matters
import * as dateFns from 'date-fns'  // Imports everything
```

```vue
<script setup lang="ts">
// DO: Import specific Vue APIs
import { ref, computed, watch } from 'vue'

// DO: Import specific icons instead of the entire icon set
import { IconUser, IconSettings } from '@tabler/icons-vue'

// DON'T: Import all icons
// import * as Icons from '@tabler/icons-vue'
</script>
```

### Conditional Feature Loading

```vue
<script setup lang="ts">
import { ref, defineAsyncComponent } from 'vue'

const showMarkdownEditor = ref(false)

// Heavy dependency (markdown parser) only loaded when needed
const MarkdownEditor = defineAsyncComponent(
  () => import('@/components/MarkdownEditor.vue')
)
</script>

<template>
  <button @click="showMarkdownEditor = true">Open Editor</button>
  <MarkdownEditor v-if="showMarkdownEditor" />
</template>
```

## Watchers vs Computed: Prefer Computed

Computed properties are synchronous, cached, and declarative. Watchers are imperative and execute side effects. When you need derived state, always use `computed`. Reserve `watch` for side effects that cannot be expressed as derivations.

```vue
<script setup lang="ts">
import { ref, computed, watch } from 'vue'

const items = ref<{ price: number; quantity: number }[]>([])

// DO: Derived state as computed
const totalPrice = computed(() =>
  items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
)

const formattedTotal = computed(() => `$${totalPrice.value.toFixed(2)}`)

const hasExpensiveItems = computed(() =>
  items.value.some(item => item.price > 100)
)

// DON'T: Using watch to derive state
// This is the wrong tool for this job
const badTotal = ref(0)
watch(items, (newItems) => {
  badTotal.value = newItems.reduce((sum, item) => sum + item.price * item.quantity, 0)
}, { deep: true })

// DO: Watch for side effects (logging, API calls, analytics)
watch(totalPrice, (newTotal, oldTotal) => {
  if (newTotal > 1000 && oldTotal <= 1000) {
    analytics.track('cart_exceeded_1000', { total: newTotal })
  }
})
</script>
```

Why computed is preferred:
- Cached: only re-evaluates when dependencies change
- Synchronous: no timing issues or stale intermediate states
- Declarative: reads like a data transformation, not a procedure
- Debuggable: Vue DevTools shows computed dependencies and values

## Avoiding Deep Watchers

Deep watchers traverse the entire object tree on every change. For large objects, this is expensive. Prefer watching specific properties or using shallow comparisons.

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'

interface FormData {
  personal: {
    name: string
    email: string
  }
  preferences: {
    theme: string
    notifications: boolean
    language: string
  }
}

const formData = ref<FormData>({
  personal: { name: '', email: '' },
  preferences: { theme: 'light', notifications: true, language: 'en' },
})

// DON'T: Deep watcher on the entire form
// Fires on ANY nested change, traverses the entire object
watch(formData, (newData) => {
  saveForm(newData)
}, { deep: true })

// DO: Watch specific paths with getter functions
watch(
  () => formData.value.personal.email,
  (newEmail) => {
    validateEmail(newEmail)
  }
)

// DO: Watch a specific sub-object when you need to track a group of related fields
watch(
  () => formData.value.preferences,
  (newPrefs) => {
    savePreferences(newPrefs)
  },
  { deep: true }  // Deep is acceptable here because preferences is a small object
)

// DO: Watch multiple specific values
watch(
  [() => formData.value.personal.name, () => formData.value.personal.email],
  ([newName, newEmail]) => {
    updateProfile(newName, newEmail)
  }
)
</script>
```

## Template Refs vs Reactive State

Use template refs for direct DOM access (focus, scroll, measurements). Use reactive state for data that drives the template. Do not store application data in template refs.

```vue
<script setup lang="ts">
import { ref, onMounted, nextTick } from 'vue'

// Template ref: DOM access only
const inputRef = ref<HTMLInputElement | null>(null)
const scrollContainerRef = ref<HTMLElement | null>(null)

// Reactive state: drives the template
const searchQuery = ref('')
const results = ref<string[]>([])

function focusInput() {
  inputRef.value?.focus()
}

function scrollToTop() {
  scrollContainerRef.value?.scrollTo({ top: 0, behavior: 'smooth' })
}

async function handleSearch() {
  results.value = await performSearch(searchQuery.value)
  // Wait for DOM update, then scroll to show results
  await nextTick()
  scrollContainerRef.value?.scrollTo({ top: 0 })
}

onMounted(() => {
  focusInput()
})
</script>

<template>
  <div>
    <input
      ref="inputRef"
      v-model="searchQuery"
      @keyup.enter="handleSearch"
      placeholder="Search..."
    />
    <div ref="scrollContainerRef" class="results-container">
      <div v-for="result in results" :key="result">{{ result }}</div>
    </div>
  </div>
</template>
```

Common template ref use cases:
- Focusing inputs (`inputRef.value?.focus()`)
- Scrolling containers (`el.scrollTo()`, `el.scrollIntoView()`)
- Reading dimensions (`el.getBoundingClientRect()`)
- Integrating third-party DOM libraries (charts, maps)

Avoid storing state in the DOM. If you find yourself reading `inputRef.value?.value` instead of using `v-model`, refactor to use reactive state.

## Performance Profiling with Vue DevTools

Vue DevTools provides a Performance tab for measuring component render times, event handling, and reactivity tracking.

### Enable Performance Tracking

```ts
// main.ts
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

// Enable performance tracking in development
if (import.meta.env.DEV) {
  app.config.performance = true
}

app.mount('#app')
```

### Identifying Slow Components

Use the `onRenderTracked` and `onRenderTriggered` hooks to understand why a component re-renders:

```vue
<script setup lang="ts">
import { ref, computed, onRenderTracked, onRenderTriggered } from 'vue'

const users = ref<User[]>([])
const filter = ref('')

const filteredUsers = computed(() =>
  users.value.filter(u => u.name.includes(filter.value))
)

// Development-only debugging hooks
if (import.meta.env.DEV) {
  onRenderTracked((event) => {
    // Logs every reactive dependency this component tracks
    console.log('Render tracked:', event)
  })

  onRenderTriggered((event) => {
    // Logs which dependency change caused a re-render
    console.log('Render triggered:', event)
    // event.key, event.target, event.type tell you exactly what changed
  })
}
</script>
```

### Profiling Checklist

1. Open Vue DevTools, navigate to the Performance tab
2. Click Record, perform the user interaction you want to profile
3. Stop recording and inspect the flame chart
4. Look for:
   - Components that render more often than expected
   - Components with long render durations
   - Computed properties that re-evaluate unexpectedly
   - Deep watchers firing on unrelated changes
5. Apply targeted optimizations (v-memo, shallowRef, computed caching)
6. Record again and compare before/after metrics

### Measuring in Production

```ts
// utils/performanceMark.ts
export function measureRender(label: string) {
  if (import.meta.env.DEV) {
    performance.mark(`${label}-start`)
    return () => {
      performance.mark(`${label}-end`)
      performance.measure(label, `${label}-start`, `${label}-end`)
      const measure = performance.getEntriesByName(label).pop()
      if (measure) {
        console.log(`${label}: ${measure.duration.toFixed(2)}ms`)
      }
    }
  }
  return () => {}
}
```

```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { measureRender } from '@/utils/performanceMark'

onMounted(() => {
  const end = measureRender('UserDashboard-mount')
  // ... initialization logic
  end()
})
</script>
```

## Best Practices

**DO:**
- Use `v-once` for truly static content
- Use `v-memo` in large `v-for` loops with selective updates
- Use `shallowRef` for large data structures that are replaced, not mutated
- Prefer `computed` over `watch` for derived state
- Lazy load routes and heavy components with `defineAsyncComponent`
- Use `<KeepAlive>` for tab interfaces and wizard flows
- Profile before optimizing: measure, change, re-measure
- Use stable unique IDs for `:key` in `v-for`
- Split vendor code into separate chunks for caching
- Use named imports for tree-shakeable libraries

**DON'T:**
- Wrap constants and static configuration in `ref()` or `reactive()`
- Use `{ deep: true }` watchers on large objects when you only need one field
- Use array index as `:key` when list items can be reordered, added, or removed
- Import entire libraries when you only need a few functions
- Add `v-memo` or `shallowRef` without profiling first (premature optimization)
- Store application state in template refs or the DOM
- Use `watch` when `computed` would suffice
- Skip `<KeepAlive>` max limit (unbounded caching leaks memory)
- Forget to clean up intervals and listeners in `onDeactivated` for kept-alive components

## Guidelines

**Essential:**
- Prefer `computed` over `watch` for all derived state
- Use stable, unique `:key` values in `v-for` loops
- Lazy load routes with dynamic `import()` in the router
- Enable `app.config.performance = true` in development
- Keep constants and static config outside the reactivity system

**Recommended:**
- Use `shallowRef` for large arrays and objects that are replaced wholesale
- Apply `v-memo` to large lists where individual items update independently
- Split vendor and application code into separate chunks
- Use `<KeepAlive>` with `:max` for tab or wizard navigation
- Analyze bundle size with `rollup-plugin-visualizer` after feature milestones

**Advanced:**
- Implement virtual scrolling for lists exceeding 100 items
- Use `onRenderTracked` and `onRenderTriggered` to diagnose unexpected re-renders
- Profile with Vue DevTools Performance tab before and after optimization
- Use `defineAsyncComponent` with loading and error states for heavy components
- Use `triggerRef` with `shallowRef` for fine-grained manual reactivity control

## Benefits

Faster initial load. Lazy loading and tree shaking reduce the JavaScript shipped to the browser.

Smooth interactions. Virtual scrolling and cached computed properties keep frame rates high.

Lower memory usage. `shallowRef` avoids deep proxy creation; `<KeepAlive>` max limits prevent unbounded caching.

Predictable rendering. `v-memo` and proper `:key` usage eliminate unnecessary DOM patches.

Measurable improvements. Vue DevTools profiling provides concrete before/after metrics.

Maintainable optimizations. Each technique is localized and can be applied or removed independently.

## Related

- [vue-component-structure.md](./vue-component-structure.md) - Component organization and SFC patterns
- [vue-composables.md](./vue-composables.md) - Extracting reusable logic into composables
- [composition-api-basics.md](./composition-api-basics.md) - Composition API fundamentals
