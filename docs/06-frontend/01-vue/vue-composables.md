# Vue Composables

Reusable stateful logic extraction for Vue 3 applications. Composables encapsulate reactive state, computed properties, watchers, and lifecycle hooks into standalone functions.

Keywords: composables, useXxx, reusable logic, Composition API, stateful functions, VueUse

## Principle

Composables are functions that leverage Vue's Composition API to encapsulate and reuse stateful logic across components. They follow the `useXxx` naming convention, return reactive state, and handle their own lifecycle cleanup. Composables replace mixins (Vue 2) with a type-safe, transparent, and composable alternative.

## What Composables Are

A composable is a function that:
- Uses Vue reactivity APIs (ref, computed, watch)
- May hook into component lifecycle (onMounted, onUnmounted)
- Returns reactive state and methods for the consuming component
- Can be called from `<script setup>` or another composable

Composables extract logic that would otherwise be duplicated across components. They are the primary reuse mechanism in Vue 3.

```ts
// composables/useCounter.ts
import { ref, computed } from 'vue'

export function useCounter(initialValue = 0) {
  const count = ref(initialValue)
  const isPositive = computed(() => count.value > 0)

  function increment() {
    count.value++
  }

  function decrement() {
    count.value--
  }

  function reset() {
    count.value = initialValue
  }

  return { count, isPositive, increment, decrement, reset }
}
```

Each call to `useCounter()` creates an independent instance with its own state. No shared global mutation, no naming collisions.

## Naming Convention

All composables must follow the `useXxx` prefix convention:

```ts
// Correct naming
useCounter()
useMouse()
useLocalStorage()
useDarkMode()
useFormValidation()
useDebounce()
useInfiniteScroll()

// Incorrect naming
counter()          // Missing use prefix
getMousePosition() // Not a composable name
createToggle()     // Factory, not composable convention
```

The `use` prefix signals to developers and tooling that the function:
- Uses Vue reactivity internally
- Must be called within a setup context (or another composable)
- May hook into the component lifecycle

## Basic Composable Patterns

### useToggle

Simple boolean state toggling:

```ts
// composables/useToggle.ts
import { ref } from 'vue'
import type { Ref } from 'vue'

export function useToggle(initialValue = false): {
  value: Ref<boolean>
  toggle: () => void
  setTrue: () => void
  setFalse: () => void
} {
  const value = ref(initialValue)

  function toggle() {
    value.value = !value.value
  }

  function setTrue() {
    value.value = true
  }

  function setFalse() {
    value.value = false
  }

  return { value, toggle, setTrue, setFalse }
}
```

Usage in a component:

```vue
<script setup lang="ts">
import { useToggle } from '@/composables/useToggle'

const { value: isModalOpen, toggle: toggleModal, setFalse: closeModal } = useToggle()
</script>

<template>
  <button @click="toggleModal">Toggle Modal</button>
  <dialog :open="isModalOpen" @close="closeModal">
    <p>Modal content</p>
    <button @click="closeModal">Close</button>
  </dialog>
</template>
```

### useMouse

Tracking mouse position with lifecycle cleanup:

```ts
// composables/useMouse.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  function update(event: MouseEvent) {
    x.value = event.pageX
    y.value = event.pageY
  }

  onMounted(() => {
    window.addEventListener('mousemove', update)
  })

  onUnmounted(() => {
    window.removeEventListener('mousemove', update)
  })

  return { x, y }
}
```

```vue
<script setup lang="ts">
import { useMouse } from '@/composables/useMouse'

const { x, y } = useMouse()
</script>

<template>
  <div>Mouse position: {{ x }}, {{ y }}</div>
</template>
```

### useLocalStorage

Persistent state backed by localStorage:

```ts
// composables/useLocalStorage.ts
import { ref, watch } from 'vue'
import type { Ref } from 'vue'

export function useLocalStorage<T>(key: string, defaultValue: T): Ref<T> {
  const stored = localStorage.getItem(key)
  const value = ref<T>(stored ? JSON.parse(stored) : defaultValue) as Ref<T>

  watch(
    value,
    (newValue) => {
      localStorage.setItem(key, JSON.stringify(newValue))
    },
    { deep: true }
  )

  return value
}
```

```vue
<script setup lang="ts">
import { useLocalStorage } from '@/composables/useLocalStorage'

const theme = useLocalStorage<'light' | 'dark'>('theme', 'light')
const recentSearches = useLocalStorage<string[]>('recent-searches', [])
</script>

<template>
  <select v-model="theme">
    <option value="light">Light</option>
    <option value="dark">Dark</option>
  </select>
</template>
```

## Composables with Parameters

### Static Parameters

Accept configuration values at creation time:

```ts
// composables/useDebounce.ts
import { ref, watch } from 'vue'
import type { Ref } from 'vue'

export function useDebounce<T>(source: Ref<T>, delay = 300): Ref<T> {
  const debounced = ref(source.value) as Ref<T>
  let timeout: ReturnType<typeof setTimeout>

  watch(source, (newValue) => {
    clearTimeout(timeout)
    timeout = setTimeout(() => {
      debounced.value = newValue
    }, delay)
  })

  return debounced
}
```

### Reactive Options

Accept refs or getters so the composable reacts to parameter changes:

```ts
// composables/usePagination.ts
import { ref, computed, watch, toValue } from 'vue'
import type { MaybeRefOrGetter } from 'vue'

interface UsePaginationOptions {
  totalItems: MaybeRefOrGetter<number>
  pageSize?: MaybeRefOrGetter<number>
}

export function usePagination(options: UsePaginationOptions) {
  const currentPage = ref(1)

  const pageSize = computed(() => toValue(options.pageSize) ?? 10)
  const totalItems = computed(() => toValue(options.totalItems))

  const totalPages = computed(() =>
    Math.ceil(totalItems.value / pageSize.value)
  )

  const offset = computed(() =>
    (currentPage.value - 1) * pageSize.value
  )

  const isFirstPage = computed(() => currentPage.value === 1)
  const isLastPage = computed(() => currentPage.value >= totalPages.value)

  // Reset to page 1 when total items changes
  watch(totalItems, () => {
    currentPage.value = 1
  })

  function nextPage() {
    if (!isLastPage.value) {
      currentPage.value++
    }
  }

  function prevPage() {
    if (!isFirstPage.value) {
      currentPage.value--
    }
  }

  function goToPage(page: number) {
    currentPage.value = Math.max(1, Math.min(page, totalPages.value))
  }

  return {
    currentPage,
    totalPages,
    pageSize,
    offset,
    isFirstPage,
    isLastPage,
    nextPage,
    prevPage,
    goToPage
  }
}
```

Using `MaybeRefOrGetter` with `toValue()` allows the caller to pass a plain value, a ref, or a getter function. The composable will react to changes in any case.

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { usePagination } from '@/composables/usePagination'

const totalItems = ref(100)

const { currentPage, totalPages, isFirstPage, isLastPage, nextPage, prevPage } =
  usePagination({
    totalItems,
    pageSize: 20
  })
</script>

<template>
  <div>
    <p>Page {{ currentPage }} of {{ totalPages }}</p>
    <button :disabled="isFirstPage" @click="prevPage">Previous</button>
    <button :disabled="isLastPage" @click="nextPage">Next</button>
  </div>
</template>
```

## Return Value Conventions

Composables should return a plain object containing refs and functions. Do not return a reactive object.

```ts
// CORRECT: Return object with individual refs
export function useUser(id: string) {
  const user = ref<User | null>(null)
  const loading = ref(false)
  const error = ref<Error | null>(null)

  // ... logic

  return { user, loading, error }
}

// The caller can destructure and rename freely
const { user, loading: userLoading, error: userError } = useUser('123')
```

```ts
// INCORRECT: Returning reactive() breaks destructuring reactivity
export function useUser(id: string) {
  const state = reactive({
    user: null as User | null,
    loading: false,
    error: null as Error | null
  })

  // ... logic

  // Destructuring this loses reactivity
  return state
}

// This destructuring loses reactivity on primitives
const { loading } = useUser('123') // loading is NOT reactive
```

Why individual refs:
- Destructuring preserves reactivity for every property
- Callers can rename properties to avoid conflicts (`loading: userLoading`)
- Each ref is independently trackable by Vue's reactivity system
- TypeScript inference works cleanly on each property

## Async Composables

### useAsyncData

Generic async data loading pattern:

```ts
// composables/useAsyncData.ts
import { ref, watch, toValue } from 'vue'
import type { Ref, MaybeRefOrGetter } from 'vue'

interface UseAsyncDataOptions<T> {
  immediate?: boolean
  defaultValue?: T
}

interface UseAsyncDataReturn<T> {
  data: Ref<T | null>
  error: Ref<Error | null>
  loading: Ref<boolean>
  execute: () => Promise<void>
}

export function useAsyncData<T>(
  fetcher: () => Promise<T>,
  options: UseAsyncDataOptions<T> = {}
): UseAsyncDataReturn<T> {
  const { immediate = true, defaultValue = null } = options

  const data = ref<T | null>(defaultValue) as Ref<T | null>
  const error = ref<Error | null>(null)
  const loading = ref(false)

  async function execute() {
    loading.value = true
    error.value = null

    try {
      data.value = await fetcher()
    } catch (e) {
      error.value = e instanceof Error ? e : new Error(String(e))
    } finally {
      loading.value = false
    }
  }

  if (immediate) {
    execute()
  }

  return { data, error, loading, execute }
}
```

```vue
<script setup lang="ts">
import { useAsyncData } from '@/composables/useAsyncData'
import { fetchUsers } from '@/api/users'

const { data: users, loading, error, execute: refresh } = useAsyncData(
  () => fetchUsers(),
  { immediate: true }
)
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">Error: {{ error.message }}</div>
  <div v-else>
    <ul>
      <li v-for="user in users" :key="user.id">{{ user.name }}</li>
    </ul>
    <button @click="refresh">Refresh</button>
  </div>
</template>
```

### useFetch with Reactive URL

Refetch when the URL changes:

```ts
// composables/useFetch.ts
import { ref, watch, toValue, isRef } from 'vue'
import type { Ref, MaybeRefOrGetter } from 'vue'

export function useFetch<T>(url: MaybeRefOrGetter<string>) {
  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<Error | null>(null)
  const loading = ref(false)

  async function fetchData() {
    loading.value = true
    error.value = null

    try {
      const response = await fetch(toValue(url))
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`)
      }
      data.value = await response.json()
    } catch (e) {
      error.value = e instanceof Error ? e : new Error(String(e))
    } finally {
      loading.value = false
    }
  }

  // Watch for URL changes and refetch
  watch(() => toValue(url), fetchData, { immediate: true })

  return { data, error, loading, refetch: fetchData }
}
```

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import { useFetch } from '@/composables/useFetch'

const userId = ref('123')
const url = computed(() => `/api/users/${userId.value}`)

// Automatically refetches when userId changes
const { data: user, loading, error } = useFetch<User>(url)

function loadNextUser() {
  userId.value = String(Number(userId.value) + 1)
}
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="user">{{ user.name }}</div>
  <button @click="loadNextUser">Next User</button>
</template>
```

Note: For production applications, prefer TanStack Query over custom fetch composables. TanStack Query provides caching, deduplication, background refetching, and retry logic out of the box.

## Composable Lifecycle Management

Composables that register event listeners, timers, or subscriptions must clean up when the component unmounts.

### Event Listener Cleanup

```ts
// composables/useEventListener.ts
import { onMounted, onUnmounted, toValue } from 'vue'
import type { MaybeRefOrGetter } from 'vue'

export function useEventListener<K extends keyof WindowEventMap>(
  target: MaybeRefOrGetter<EventTarget | null>,
  event: K,
  handler: (event: WindowEventMap[K]) => void
) {
  onMounted(() => {
    const el = toValue(target)
    el?.addEventListener(event, handler as EventListener)
  })

  onUnmounted(() => {
    const el = toValue(target)
    el?.removeEventListener(event, handler as EventListener)
  })
}
```

### Timer Cleanup

```ts
// composables/useInterval.ts
import { ref, onUnmounted } from 'vue'

export function useInterval(callback: () => void, intervalMs: number) {
  const isActive = ref(true)
  let intervalId: ReturnType<typeof setInterval> | null = null

  function start() {
    if (intervalId !== null) return
    intervalId = setInterval(callback, intervalMs)
    isActive.value = true
  }

  function stop() {
    if (intervalId !== null) {
      clearInterval(intervalId)
      intervalId = null
    }
    isActive.value = false
  }

  start()

  onUnmounted(() => {
    stop()
  })

  return { isActive, start, stop }
}
```

### Watcher Cleanup

Watchers created inside `<script setup>` are automatically stopped when the component unmounts. Watchers created manually (outside setup) must be stopped explicitly:

```ts
// composables/useAutoSave.ts
import { watch, onUnmounted } from 'vue'
import type { Ref } from 'vue'

export function useAutoSave<T>(source: Ref<T>, saveFn: (value: T) => Promise<void>) {
  const stopWatch = watch(
    source,
    async (newValue) => {
      await saveFn(newValue)
    },
    { deep: true }
  )

  // Explicit cleanup if needed before unmount
  onUnmounted(() => {
    stopWatch()
  })

  return { stopWatch }
}
```

## Composable Composition

Composables can use other composables internally. This is the primary mechanism for building complex behavior from simple parts.

```ts
// composables/useWindowScroll.ts
import { ref } from 'vue'
import { useEventListener } from '@/composables/useEventListener'
import { useDebounce } from '@/composables/useDebounce'

export function useWindowScroll(debounceMs = 100) {
  const x = ref(window.scrollX)
  const y = ref(window.scrollY)

  function update() {
    x.value = window.scrollX
    y.value = window.scrollY
  }

  // Uses useEventListener for automatic cleanup
  useEventListener(() => window, 'scroll', update)

  // Uses useDebounce for performance
  const debouncedY = useDebounce(y, debounceMs)

  const isScrolled = computed(() => debouncedY.value > 0)

  return { x, y, debouncedY, isScrolled }
}
```

A more complex composition example combining multiple concerns:

```ts
// composables/useSearchWithHistory.ts
import { ref, computed } from 'vue'
import { useDebounce } from '@/composables/useDebounce'
import { useLocalStorage } from '@/composables/useLocalStorage'
import { useFetch } from '@/composables/useFetch'

interface UseSearchOptions {
  maxHistory?: number
  debounceMs?: number
}

export function useSearchWithHistory<T>(
  baseUrl: string,
  options: UseSearchOptions = {}
) {
  const { maxHistory = 10, debounceMs = 300 } = options

  const query = ref('')
  const debouncedQuery = useDebounce(query, debounceMs)
  const searchHistory = useLocalStorage<string[]>('search-history', [])

  const searchUrl = computed(() => {
    if (!debouncedQuery.value) return ''
    return `${baseUrl}?q=${encodeURIComponent(debouncedQuery.value)}`
  })

  const { data: results, loading, error } = useFetch<T[]>(searchUrl)

  function addToHistory(term: string) {
    const filtered = searchHistory.value.filter((t) => t !== term)
    searchHistory.value = [term, ...filtered].slice(0, maxHistory)
  }

  function clearHistory() {
    searchHistory.value = []
  }

  function search(term: string) {
    query.value = term
    if (term.trim()) {
      addToHistory(term.trim())
    }
  }

  return {
    query,
    results,
    loading,
    error,
    searchHistory,
    search,
    clearHistory
  }
}
```

## Side Effects in Composables

Composables may introduce side effects: event listeners, DOM mutations, timers, network requests, and watchers. Follow these rules:

1. Always clean up in `onUnmounted`.
2. Document what side effects the composable creates.
3. Provide start/stop controls when the side effect is optional.

```ts
// composables/useOnlineStatus.ts
import { ref, onMounted, onUnmounted } from 'vue'

/**
 * Tracks browser online/offline status.
 * Side effects: registers 'online' and 'offline' event listeners on window.
 */
export function useOnlineStatus() {
  const isOnline = ref(navigator.onLine)

  function handleOnline() {
    isOnline.value = true
  }

  function handleOffline() {
    isOnline.value = false
  }

  onMounted(() => {
    window.addEventListener('online', handleOnline)
    window.addEventListener('offline', handleOffline)
  })

  onUnmounted(() => {
    window.removeEventListener('online', handleOnline)
    window.removeEventListener('offline', handleOffline)
  })

  return { isOnline }
}
```

```ts
// composables/useDocumentTitle.ts
import { watch, onUnmounted } from 'vue'
import type { MaybeRefOrGetter } from 'vue'
import { toValue } from 'vue'

/**
 * Reactively sets document.title.
 * Side effects: mutates document.title. Restores original title on unmount.
 */
export function useDocumentTitle(title: MaybeRefOrGetter<string>) {
  const originalTitle = document.title

  watch(
    () => toValue(title),
    (newTitle) => {
      document.title = newTitle
    },
    { immediate: true }
  )

  onUnmounted(() => {
    document.title = originalTitle
  })
}
```

## Testing Composables in Isolation

Composables can be tested without mounting a full component by using a minimal wrapper.

### Testing with a wrapper component

```ts
// composables/__tests__/useCounter.test.ts
import { describe, it, expect } from 'vitest'
import { useCounter } from '@/composables/useCounter'
import { mount } from '@vue/test-utils'
import { defineComponent } from 'vue'

function withSetup<T>(composable: () => T): { result: T; wrapper: ReturnType<typeof mount> } {
  let result!: T
  const wrapper = mount(
    defineComponent({
      setup() {
        result = composable()
        return {}
      },
      template: '<div />'
    })
  )
  return { result, wrapper }
}

describe('useCounter', () => {
  it('initializes with default value', () => {
    const { result } = withSetup(() => useCounter())
    expect(result.count.value).toBe(0)
  })

  it('initializes with custom value', () => {
    const { result } = withSetup(() => useCounter(10))
    expect(result.count.value).toBe(10)
  })

  it('increments count', () => {
    const { result } = withSetup(() => useCounter())
    result.increment()
    expect(result.count.value).toBe(1)
  })

  it('decrements count', () => {
    const { result } = withSetup(() => useCounter(5))
    result.decrement()
    expect(result.count.value).toBe(4)
  })

  it('resets to initial value', () => {
    const { result } = withSetup(() => useCounter(5))
    result.increment()
    result.increment()
    result.reset()
    expect(result.count.value).toBe(5)
  })

  it('computes isPositive correctly', () => {
    const { result } = withSetup(() => useCounter(0))
    expect(result.isPositive.value).toBe(false)
    result.increment()
    expect(result.isPositive.value).toBe(true)
  })
})
```

### Testing composables with lifecycle hooks

For composables that use `onMounted` or `onUnmounted`, the wrapper component approach handles lifecycle automatically:

```ts
// composables/__tests__/useOnlineStatus.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { useOnlineStatus } from '@/composables/useOnlineStatus'
import { mount } from '@vue/test-utils'
import { defineComponent, nextTick } from 'vue'

describe('useOnlineStatus', () => {
  it('registers and removes event listeners', () => {
    const addSpy = vi.spyOn(window, 'addEventListener')
    const removeSpy = vi.spyOn(window, 'removeEventListener')

    const wrapper = mount(
      defineComponent({
        setup() {
          useOnlineStatus()
          return {}
        },
        template: '<div />'
      })
    )

    expect(addSpy).toHaveBeenCalledWith('online', expect.any(Function))
    expect(addSpy).toHaveBeenCalledWith('offline', expect.any(Function))

    wrapper.unmount()

    expect(removeSpy).toHaveBeenCalledWith('online', expect.any(Function))
    expect(removeSpy).toHaveBeenCalledWith('offline', expect.any(Function))

    addSpy.mockRestore()
    removeSpy.mockRestore()
  })
})
```

### Testing async composables

```ts
// composables/__tests__/useAsyncData.test.ts
import { describe, it, expect, vi } from 'vitest'
import { useAsyncData } from '@/composables/useAsyncData'
import { mount, flushPromises } from '@vue/test-utils'
import { defineComponent } from 'vue'

describe('useAsyncData', () => {
  it('fetches data and updates state', async () => {
    const mockData = { id: '1', name: 'Test User' }
    const fetcher = vi.fn().mockResolvedValue(mockData)

    let result: ReturnType<typeof useAsyncData>

    mount(
      defineComponent({
        setup() {
          result = useAsyncData(fetcher)
          return {}
        },
        template: '<div />'
      })
    )

    expect(result!.loading.value).toBe(true)

    await flushPromises()

    expect(result!.loading.value).toBe(false)
    expect(result!.data.value).toEqual(mockData)
    expect(result!.error.value).toBeNull()
  })

  it('handles errors', async () => {
    const fetcher = vi.fn().mockRejectedValue(new Error('Network error'))

    let result: ReturnType<typeof useAsyncData>

    mount(
      defineComponent({
        setup() {
          result = useAsyncData(fetcher)
          return {}
        },
        template: '<div />'
      })
    )

    await flushPromises()

    expect(result!.loading.value).toBe(false)
    expect(result!.data.value).toBeNull()
    expect(result!.error.value?.message).toBe('Network error')
  })
})
```

## VueUse Library

VueUse is a collection of production-ready composables. Prefer VueUse over custom implementations for common utilities.

```bash
npm install @vueuse/core
```

Commonly used VueUse composables:

```vue
<script setup lang="ts">
import {
  useLocalStorage,
  useDebounceFn,
  useWindowSize,
  useMediaQuery,
  useClipboard,
  useEventListener,
  onClickOutside,
  useIntersectionObserver
} from '@vueuse/core'
import { ref } from 'vue'

// Persistent state
const theme = useLocalStorage('theme', 'light')

// Debounced function
const onSearch = useDebounceFn((term: string) => {
  console.log('Searching:', term)
}, 300)

// Responsive design
const { width, height } = useWindowSize()
const isMobile = useMediaQuery('(max-width: 768px)')

// Clipboard access
const { text, copy, isSupported } = useClipboard()

// Click outside detection
const dropdownRef = ref<HTMLElement | null>(null)
onClickOutside(dropdownRef, () => {
  // Close dropdown
})

// Intersection observer for lazy loading
const targetRef = ref<HTMLElement | null>(null)
const isVisible = ref(false)

useIntersectionObserver(targetRef, ([entry]) => {
  isVisible.value = entry.isIntersecting
})
</script>
```

When to use VueUse vs custom composable:
- VueUse already provides it: use VueUse
- Logic is project-specific or domain-specific: write a custom composable
- You need a thin wrapper around VueUse with project defaults: write a custom composable that composes VueUse internally

## When NOT to Use Composables

Not all logic should be extracted into a composable. Avoid composable extraction when:

**One-off logic in a single component:**

```vue
<script setup lang="ts">
// This is fine inline. No need for useFormattedDate() composable.
import { computed } from 'vue'

const props = defineProps<{ createdAt: string }>()

const formattedDate = computed(() =>
  new Date(props.createdAt).toLocaleDateString()
)
</script>
```

**Pure utility functions with no reactive state:**

```ts
// utils/formatCurrency.ts - NOT a composable, just a function
export function formatCurrency(amount: number, currency = 'USD'): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency
  }).format(amount)
}
```

**Simple computed properties that depend only on props:**

```vue
<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{ firstName: string; lastName: string }>()

// Inline computed is clearer here than useFullName(firstName, lastName)
const fullName = computed(() => `${props.firstName} ${props.lastName}`)
</script>
```

Rules of thumb:
- If the logic has no refs, computed, or lifecycle hooks, it is a utility function, not a composable.
- If the logic is used in only one component and is under 15 lines, keep it inline.
- If you are creating a composable that wraps a single ref with no additional behavior, you are over-abstracting.

## Directory Structure

Organize composables in a dedicated directory:

```
src/
├── composables/              # Application-wide composables
│   ├── useAuth.ts
│   ├── useLocalStorage.ts
│   ├── useDebounce.ts
│   ├── usePagination.ts
│   ├── useFormValidation.ts
│   └── __tests__/
│       ├── useAuth.test.ts
│       ├── useDebounce.test.ts
│       └── usePagination.test.ts
├── features/
│   └── users/
│       ├── composables/      # Feature-scoped composables
│       │   ├── useUserList.ts
│       │   └── useUserForm.ts
│       ├── components/
│       └── views/
```

Placement rules:
- `src/composables/` for composables used across multiple features
- `src/features/{feature}/composables/` for composables used only within a specific feature
- Co-locate tests in `__tests__/` subdirectory adjacent to composables
- Do not promote a feature-scoped composable to application-wide until a second feature needs it

## Best Practices

**DO:**

- Name composables with `use` prefix: `useCounter`, `useFetch`, `useAuth`
- Return plain objects with individual refs, not reactive objects
- Clean up all side effects in `onUnmounted`
- Accept `MaybeRefOrGetter` for parameters that should be reactive
- Use `toValue()` to unwrap `MaybeRefOrGetter` parameters
- Type return values explicitly for complex composables
- Document side effects with JSDoc comments
- Test composables independently from components
- Compose small composables into larger ones

**DON'T:**

- Return a `reactive()` object (destructuring loses reactivity on primitives)
- Forget to remove event listeners, clear timers, or stop watchers
- Create composables for pure utility functions with no reactive state
- Wrap a single ref with no additional behavior
- Use composables outside of `<script setup>` or `setup()` function
- Mutate input refs passed as parameters (treat them as read-only)
- Rely on global mutable state inside composables (use provide/inject instead)

## Guidelines

**Essential:**
- Every composable follows `useXxx` naming
- Return type is a plain object with refs and functions
- All side effects are cleaned up on unmount
- Composables are called synchronously at the top level of setup

**Recommended:**
- Use `MaybeRefOrGetter<T>` for parameters that callers might pass as refs or plain values
- Provide explicit TypeScript return types for composables with complex signatures
- Co-locate unit tests alongside composable files
- Extract composables only when logic is reused or the component becomes too large

**Advanced:**
- Compose multiple composables to build higher-level abstractions
- Use VueUse as a foundation and wrap it with project-specific defaults
- Implement async composables with loading, error, and retry states
- Use `effectScope()` to group and dispose of reactive effects in advanced scenarios

## Benefits

Reusability. Extract logic once, use across any number of components.

Type safety. Full TypeScript inference on parameters and return values.

Testability. Test reactive logic in isolation without rendering components.

Transparency. No hidden merging or naming conflicts (unlike mixins).

Composition. Build complex behavior by combining simple composables.

Encapsulation. Side effects and lifecycle cleanup are self-contained.

## Related

- [composition-api-basics.md](./composition-api-basics.md) - Composition API fundamentals
- [vue-component-structure.md](./vue-component-structure.md) - Component organization patterns
- [vue-testing-vitest.md](./vue-testing-vitest.md) - Testing Vue components and composables
