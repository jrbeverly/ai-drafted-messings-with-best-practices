# Vue Testing with Vitest

Vue 3 component and unit testing using Vitest, Vue Test Utils, and TypeScript. Fast, type-safe, Vite-native test runner.

**Keywords:** vitest, vue-test-utils, component-testing, unit-testing, mocking, pinia-testing, composable-testing, snapshot-testing, coverage

## Principle

Tests verify behavior, not implementation. Write tests that interact with components the way users do: render, find elements, trigger actions, assert outcomes. Fast feedback loops from Vitest enable confident refactoring and AI-assisted iteration.

## Vitest Setup

### Installation

```bash
npm install -D vitest @vue/test-utils @vitejs/plugin-vue jsdom
npm install -D @pinia/testing  # If using Pinia
```

### Configuration (vitest.config.ts)

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import vue from '@vitejs/plugin-vue'
import { resolve } from 'path'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
    },
  },
  test: {
    globals: true,
    environment: 'jsdom',
    include: ['src/**/*.{test,spec}.{ts,tsx}'],
    exclude: ['node_modules', 'dist', 'e2e'],
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html', 'lcov'],
      include: ['src/**/*.{ts,vue}'],
      exclude: [
        'src/**/*.d.ts',
        'src/**/*.{test,spec}.ts',
        'src/test/**',
        'src/main.ts',
      ],
      thresholds: {
        statements: 80,
        branches: 80,
        functions: 80,
        lines: 80,
      },
    },
  },
})
```

### Global Setup File

```ts
// src/test/setup.ts
import { config } from '@vue/test-utils'
import { createTestingPinia } from '@pinia/testing'
import { vi } from 'vitest'

// Global stubs for common components
config.global.stubs = {
  // Stub router-link and router-view globally
  RouterLink: true,
  RouterView: true,
}

// Clean up after each test
afterEach(() => {
  vi.restoreAllMocks()
})
```

### TypeScript Configuration

```json
// tsconfig.json (add to compilerOptions)
{
  "compilerOptions": {
    "types": ["vitest/globals"]
  }
}
```

## Component Testing with Vue Test Utils

### mount vs shallowMount

```ts
// components/UserCard.vue
// <script setup lang="ts">
// import UserAvatar from './UserAvatar.vue'
// defineProps<{ name: string; email: string }>()
// </script>

import { describe, it, expect } from 'vitest'
import { mount, shallowMount } from '@vue/test-utils'
import UserCard from '@/components/UserCard.vue'

describe('UserCard', () => {
  // mount: renders full component tree (children included)
  it('renders with full child components', () => {
    const wrapper = mount(UserCard, {
      props: { name: 'Alice', email: 'alice@example.com' },
    })

    // UserAvatar is fully rendered
    expect(wrapper.findComponent({ name: 'UserAvatar' }).exists()).toBe(true)
  })

  // shallowMount: stubs child components (renders only this component)
  it('renders with stubbed children', () => {
    const wrapper = shallowMount(UserCard, {
      props: { name: 'Alice', email: 'alice@example.com' },
    })

    // UserAvatar is replaced with a stub
    expect(wrapper.text()).toContain('Alice')
    expect(wrapper.text()).toContain('alice@example.com')
  })
})
```

Use `mount` for integration-style tests where child behavior matters. Use `shallowMount` to isolate the component under test.

### Creating a Reusable Factory

```ts
import { mount, type VueWrapper } from '@vue/test-utils'
import UserCard from '@/components/UserCard.vue'

function createWrapper(overrides: Record<string, unknown> = {}): VueWrapper {
  return mount(UserCard, {
    props: {
      name: 'Default Name',
      email: 'default@example.com',
      ...overrides,
    },
    global: {
      stubs: { RouterLink: true },
    },
  })
}

describe('UserCard', () => {
  it('renders name from props', () => {
    const wrapper = createWrapper({ name: 'Bob' })
    expect(wrapper.text()).toContain('Bob')
  })
})
```

## Testing Props

```vue
<!-- components/StatusBadge.vue -->
<script setup lang="ts">
interface Props {
  status: 'active' | 'inactive' | 'pending'
  label?: string
}

const props = withDefaults(defineProps<Props>(), {
  label: '',
})
</script>

<template>
  <span :class="`badge badge--${status}`" data-testid="status-badge">
    {{ label || status }}
  </span>
</template>
```

```ts
// components/__tests__/StatusBadge.test.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import StatusBadge from '@/components/StatusBadge.vue'

describe('StatusBadge', () => {
  it('renders the status text when no label provided', () => {
    const wrapper = mount(StatusBadge, {
      props: { status: 'active' },
    })

    expect(wrapper.text()).toBe('active')
  })

  it('renders custom label over status text', () => {
    const wrapper = mount(StatusBadge, {
      props: { status: 'active', label: 'Online' },
    })

    expect(wrapper.text()).toBe('Online')
  })

  it('applies correct CSS class based on status', () => {
    const wrapper = mount(StatusBadge, {
      props: { status: 'inactive' },
    })

    const badge = wrapper.find('[data-testid="status-badge"]')
    expect(badge.classes()).toContain('badge--inactive')
  })

  it.each([
    ['active', 'badge--active'],
    ['inactive', 'badge--inactive'],
    ['pending', 'badge--pending'],
  ])('status "%s" applies class "%s"', (status, expectedClass) => {
    const wrapper = mount(StatusBadge, {
      props: { status: status as 'active' | 'inactive' | 'pending' },
    })

    expect(wrapper.find('[data-testid="status-badge"]').classes()).toContain(expectedClass)
  })
})
```

## Testing Emits

```vue
<!-- components/ConfirmDialog.vue -->
<script setup lang="ts">
interface Props {
  title: string
  message: string
}

defineProps<Props>()

const emit = defineEmits<{
  (e: 'confirm'): void
  (e: 'cancel'): void
}>()
</script>

<template>
  <div class="dialog" role="dialog">
    <h2>{{ title }}</h2>
    <p>{{ message }}</p>
    <button data-testid="confirm-btn" @click="emit('confirm')">Confirm</button>
    <button data-testid="cancel-btn" @click="emit('cancel')">Cancel</button>
  </div>
</template>
```

```ts
// components/__tests__/ConfirmDialog.test.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import ConfirmDialog from '@/components/ConfirmDialog.vue'

describe('ConfirmDialog', () => {
  function createWrapper() {
    return mount(ConfirmDialog, {
      props: { title: 'Delete Item', message: 'Are you sure?' },
    })
  }

  it('emits confirm event when confirm button clicked', async () => {
    const wrapper = createWrapper()

    await wrapper.find('[data-testid="confirm-btn"]').trigger('click')

    expect(wrapper.emitted('confirm')).toHaveLength(1)
  })

  it('emits cancel event when cancel button clicked', async () => {
    const wrapper = createWrapper()

    await wrapper.find('[data-testid="cancel-btn"]').trigger('click')

    expect(wrapper.emitted('cancel')).toHaveLength(1)
  })

  it('does not emit confirm when cancel is clicked', async () => {
    const wrapper = createWrapper()

    await wrapper.find('[data-testid="cancel-btn"]').trigger('click')

    expect(wrapper.emitted('confirm')).toBeUndefined()
  })
})
```

## Testing Slots

```vue
<!-- components/Card.vue -->
<script setup lang="ts">
defineProps<{ title: string }>()
</script>

<template>
  <div class="card">
    <header class="card__header">
      <slot name="header">{{ title }}</slot>
    </header>
    <div class="card__body">
      <slot />
    </div>
    <footer v-if="$slots.footer" class="card__footer">
      <slot name="footer" />
    </footer>
  </div>
</template>
```

```ts
// components/__tests__/Card.test.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import Card from '@/components/Card.vue'

describe('Card', () => {
  it('renders default slot content in body', () => {
    const wrapper = mount(Card, {
      props: { title: 'My Card' },
      slots: {
        default: '<p>Card body content</p>',
      },
    })

    expect(wrapper.find('.card__body').html()).toContain('Card body content')
  })

  it('renders title in header when no header slot provided', () => {
    const wrapper = mount(Card, {
      props: { title: 'My Card' },
    })

    expect(wrapper.find('.card__header').text()).toBe('My Card')
  })

  it('renders named header slot over title prop', () => {
    const wrapper = mount(Card, {
      props: { title: 'My Card' },
      slots: {
        header: '<h1>Custom Header</h1>',
      },
    })

    expect(wrapper.find('.card__header').text()).toBe('Custom Header')
  })

  it('renders footer only when footer slot is provided', () => {
    const withoutFooter = mount(Card, { props: { title: 'Test' } })
    expect(withoutFooter.find('.card__footer').exists()).toBe(false)

    const withFooter = mount(Card, {
      props: { title: 'Test' },
      slots: { footer: '<span>Footer</span>' },
    })
    expect(withFooter.find('.card__footer').exists()).toBe(true)
    expect(withFooter.find('.card__footer').text()).toBe('Footer')
  })
})
```

## Testing User Interactions

### trigger and setValue

```vue
<!-- components/SearchForm.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const emit = defineEmits<{
  (e: 'search', query: string): void
}>()

const query = ref('')

function handleSubmit() {
  if (query.value.trim()) {
    emit('search', query.value.trim())
  }
}
</script>

<template>
  <form @submit.prevent="handleSubmit" data-testid="search-form">
    <input
      v-model="query"
      type="text"
      placeholder="Search..."
      data-testid="search-input"
    />
    <button type="submit" data-testid="search-btn">Search</button>
  </form>
</template>
```

```ts
// components/__tests__/SearchForm.test.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import SearchForm from '@/components/SearchForm.vue'

describe('SearchForm', () => {
  it('emits search event with query on submit', async () => {
    const wrapper = mount(SearchForm)

    await wrapper.find('[data-testid="search-input"]').setValue('vue testing')
    await wrapper.find('[data-testid="search-form"]').trigger('submit')

    expect(wrapper.emitted('search')).toHaveLength(1)
    expect(wrapper.emitted('search')![0]).toEqual(['vue testing'])
  })

  it('trims whitespace from query', async () => {
    const wrapper = mount(SearchForm)

    await wrapper.find('[data-testid="search-input"]').setValue('  hello  ')
    await wrapper.find('[data-testid="search-form"]').trigger('submit')

    expect(wrapper.emitted('search')![0]).toEqual(['hello'])
  })

  it('does not emit search when query is empty', async () => {
    const wrapper = mount(SearchForm)

    await wrapper.find('[data-testid="search-input"]').setValue('   ')
    await wrapper.find('[data-testid="search-form"]').trigger('submit')

    expect(wrapper.emitted('search')).toBeUndefined()
  })

  it('does not emit search when query is not entered', async () => {
    const wrapper = mount(SearchForm)

    await wrapper.find('[data-testid="search-form"]').trigger('submit')

    expect(wrapper.emitted('search')).toBeUndefined()
  })
})
```

## Testing Async Behavior

### nextTick and flushPromises

```vue
<!-- components/UserProfile.vue -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'

interface User {
  id: string
  name: string
  email: string
}

const props = defineProps<{ userId: string }>()

const user = ref<User | null>(null)
const loading = ref(false)
const error = ref<string | null>(null)

async function fetchUser() {
  loading.value = true
  error.value = null
  try {
    const response = await fetch(`/api/users/${props.userId}`)
    if (!response.ok) throw new Error('User not found')
    user.value = await response.json()
  } catch (e) {
    error.value = (e as Error).message
  } finally {
    loading.value = false
  }
}

onMounted(fetchUser)
</script>

<template>
  <div>
    <div v-if="loading" data-testid="loading">Loading...</div>
    <div v-else-if="error" data-testid="error">{{ error }}</div>
    <div v-else-if="user" data-testid="user-info">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>
  </div>
</template>
```

```ts
// components/__tests__/UserProfile.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mount, flushPromises } from '@vue/test-utils'
import UserProfile from '@/components/UserProfile.vue'

const mockUser = { id: '1', name: 'Alice', email: 'alice@example.com' }

describe('UserProfile', () => {
  beforeEach(() => {
    vi.restoreAllMocks()
  })

  it('shows loading state initially', () => {
    // Mock fetch that never resolves during this test
    vi.stubGlobal('fetch', vi.fn(() => new Promise(() => {})))

    const wrapper = mount(UserProfile, {
      props: { userId: '1' },
    })

    expect(wrapper.find('[data-testid="loading"]').exists()).toBe(true)
  })

  it('renders user data after successful fetch', async () => {
    vi.stubGlobal(
      'fetch',
      vi.fn(() =>
        Promise.resolve({
          ok: true,
          json: () => Promise.resolve(mockUser),
        })
      )
    )

    const wrapper = mount(UserProfile, {
      props: { userId: '1' },
    })

    // Wait for all promises to resolve
    await flushPromises()

    expect(wrapper.find('[data-testid="loading"]').exists()).toBe(false)
    expect(wrapper.find('[data-testid="user-info"]').text()).toContain('Alice')
    expect(wrapper.find('[data-testid="user-info"]').text()).toContain('alice@example.com')
  })

  it('renders error message on fetch failure', async () => {
    vi.stubGlobal(
      'fetch',
      vi.fn(() =>
        Promise.resolve({
          ok: false,
        })
      )
    )

    const wrapper = mount(UserProfile, {
      props: { userId: '999' },
    })

    await flushPromises()

    expect(wrapper.find('[data-testid="error"]').text()).toBe('User not found')
    expect(wrapper.find('[data-testid="user-info"]').exists()).toBe(false)
  })
})
```

### nextTick for DOM updates

```ts
import { nextTick } from 'vue'

it('updates DOM after reactive state change', async () => {
  const wrapper = mount(Counter)

  // Trigger state change
  await wrapper.find('[data-testid="increment"]').trigger('click')

  // nextTick ensures Vue has updated the DOM
  await nextTick()

  expect(wrapper.find('[data-testid="count"]').text()).toBe('1')
})
```

## Testing Composables

### Direct Invocation (Simple Composables)

```ts
// composables/useCounter.ts
import { ref, computed } from 'vue'

export function useCounter(initial = 0) {
  const count = ref(initial)
  const isPositive = computed(() => count.value > 0)

  function increment() {
    count.value++
  }

  function decrement() {
    count.value--
  }

  function reset() {
    count.value = initial
  }

  return { count, isPositive, increment, decrement, reset }
}
```

```ts
// composables/__tests__/useCounter.test.ts
import { describe, it, expect } from 'vitest'
import { useCounter } from '@/composables/useCounter'

describe('useCounter', () => {
  it('initializes with default value of 0', () => {
    const { count } = useCounter()
    expect(count.value).toBe(0)
  })

  it('initializes with custom value', () => {
    const { count } = useCounter(10)
    expect(count.value).toBe(10)
  })

  it('increments count', () => {
    const { count, increment } = useCounter()
    increment()
    expect(count.value).toBe(1)
  })

  it('decrements count', () => {
    const { count, decrement } = useCounter(5)
    decrement()
    expect(count.value).toBe(4)
  })

  it('resets to initial value', () => {
    const { count, increment, reset } = useCounter(3)
    increment()
    increment()
    expect(count.value).toBe(5)

    reset()
    expect(count.value).toBe(3)
  })

  it('computes isPositive correctly', () => {
    const { isPositive, increment, decrement } = useCounter(0)
    expect(isPositive.value).toBe(false)

    increment()
    expect(isPositive.value).toBe(true)

    decrement()
    decrement()
    expect(isPositive.value).toBe(false)
  })
})
```

### Wrapper Component (Composables Requiring Lifecycle)

```ts
// composables/useWindowSize.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useWindowSize() {
  const width = ref(window.innerWidth)
  const height = ref(window.innerHeight)

  function update() {
    width.value = window.innerWidth
    height.value = window.innerHeight
  }

  onMounted(() => window.addEventListener('resize', update))
  onUnmounted(() => window.removeEventListener('resize', update))

  return { width, height }
}
```

```ts
// composables/__tests__/useWindowSize.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mount } from '@vue/test-utils'
import { defineComponent } from 'vue'
import { useWindowSize } from '@/composables/useWindowSize'

// Helper to test composables that need a component lifecycle
function withSetup<T>(composableFn: () => T) {
  let result!: T
  const TestComponent = defineComponent({
    setup() {
      result = composableFn()
      return {}
    },
    template: '<div />',
  })

  const wrapper = mount(TestComponent)
  return { result, wrapper }
}

describe('useWindowSize', () => {
  it('returns current window dimensions', () => {
    const { result } = withSetup(() => useWindowSize())

    expect(result.width.value).toBe(window.innerWidth)
    expect(result.height.value).toBe(window.innerHeight)
  })

  it('updates on window resize', async () => {
    const { result } = withSetup(() => useWindowSize())

    // Simulate resize
    Object.defineProperty(window, 'innerWidth', { value: 800, writable: true })
    Object.defineProperty(window, 'innerHeight', { value: 600, writable: true })
    window.dispatchEvent(new Event('resize'))

    expect(result.width.value).toBe(800)
    expect(result.height.value).toBe(600)
  })

  it('cleans up event listener on unmount', () => {
    const removeSpy = vi.spyOn(window, 'removeEventListener')
    const { wrapper } = withSetup(() => useWindowSize())

    wrapper.unmount()

    expect(removeSpy).toHaveBeenCalledWith('resize', expect.any(Function))
  })
})
```

## Mocking

### vi.fn (Mock Functions)

```ts
import { describe, it, expect, vi } from 'vitest'

it('calls onSubmit handler with form data', async () => {
  const onSubmit = vi.fn()

  const wrapper = mount(ContactForm, {
    props: { onSubmit },
  })

  await wrapper.find('[data-testid="name"]').setValue('Alice')
  await wrapper.find('[data-testid="email"]').setValue('alice@example.com')
  await wrapper.find('form').trigger('submit')

  expect(onSubmit).toHaveBeenCalledOnce()
  expect(onSubmit).toHaveBeenCalledWith({
    name: 'Alice',
    email: 'alice@example.com',
  })
})
```

### vi.mock (Module Mocking)

```ts
// Mock an entire module
import { describe, it, expect, vi } from 'vitest'
import { mount, flushPromises } from '@vue/test-utils'
import UserList from '@/components/UserList.vue'
import { fetchUsers } from '@/api/users'

// Mock the API module
vi.mock('@/api/users', () => ({
  fetchUsers: vi.fn(),
}))

const mockedFetchUsers = vi.mocked(fetchUsers)

describe('UserList', () => {
  it('renders list of users from API', async () => {
    mockedFetchUsers.mockResolvedValue([
      { id: '1', name: 'Alice' },
      { id: '2', name: 'Bob' },
    ])

    const wrapper = mount(UserList)
    await flushPromises()

    const items = wrapper.findAll('[data-testid="user-item"]')
    expect(items).toHaveLength(2)
    expect(items[0].text()).toContain('Alice')
    expect(items[1].text()).toContain('Bob')
  })

  it('shows empty state when no users returned', async () => {
    mockedFetchUsers.mockResolvedValue([])

    const wrapper = mount(UserList)
    await flushPromises()

    expect(wrapper.find('[data-testid="empty-state"]').exists()).toBe(true)
  })
})
```

### vi.spyOn (Spy on Methods)

```ts
import { describe, it, expect, vi } from 'vitest'

it('calls console.error on validation failure', async () => {
  const errorSpy = vi.spyOn(console, 'error').mockImplementation(() => {})

  const wrapper = mount(Form)
  await wrapper.find('form').trigger('submit')

  expect(errorSpy).toHaveBeenCalledWith('Validation failed')
  errorSpy.mockRestore()
})

it('calls localStorage.setItem when saving preferences', async () => {
  const setItemSpy = vi.spyOn(Storage.prototype, 'setItem')

  const wrapper = mount(Settings)
  await wrapper.find('[data-testid="save-btn"]').trigger('click')

  expect(setItemSpy).toHaveBeenCalledWith('theme', 'dark')
})
```

### Mocking Timers

```ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'

describe('Debounced search', () => {
  beforeEach(() => {
    vi.useFakeTimers()
  })

  afterEach(() => {
    vi.useRealTimers()
  })

  it('debounces search input by 300ms', async () => {
    const wrapper = mount(SearchInput)

    await wrapper.find('input').setValue('hello')

    // Emit should not fire yet
    expect(wrapper.emitted('search')).toBeUndefined()

    // Advance time by 300ms
    vi.advanceTimersByTime(300)

    expect(wrapper.emitted('search')).toHaveLength(1)
    expect(wrapper.emitted('search')![0]).toEqual(['hello'])
  })
})
```

## Testing with Pinia Stores

### createTestingPinia

```ts
// stores/useUserStore.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useUserStore = defineStore('user', () => {
  const user = ref<{ id: string; name: string; role: string } | null>(null)
  const isLoggedIn = computed(() => user.value !== null)
  const isAdmin = computed(() => user.value?.role === 'admin')

  async function login(email: string, password: string) {
    const response = await fetch('/api/login', {
      method: 'POST',
      body: JSON.stringify({ email, password }),
    })
    user.value = await response.json()
  }

  function logout() {
    user.value = null
  }

  return { user, isLoggedIn, isAdmin, login, logout }
})
```

```ts
// components/__tests__/NavBar.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mount } from '@vue/test-utils'
import { createTestingPinia } from '@pinia/testing'
import NavBar from '@/components/NavBar.vue'
import { useUserStore } from '@/stores/useUserStore'

describe('NavBar', () => {
  it('shows login button when user is not logged in', () => {
    const wrapper = mount(NavBar, {
      global: {
        plugins: [
          createTestingPinia({
            initialState: {
              user: { user: null },
            },
          }),
        ],
      },
    })

    expect(wrapper.find('[data-testid="login-btn"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="logout-btn"]').exists()).toBe(false)
  })

  it('shows user name and logout button when logged in', () => {
    const wrapper = mount(NavBar, {
      global: {
        plugins: [
          createTestingPinia({
            initialState: {
              user: { user: { id: '1', name: 'Alice', role: 'user' } },
            },
          }),
        ],
      },
    })

    expect(wrapper.text()).toContain('Alice')
    expect(wrapper.find('[data-testid="logout-btn"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="login-btn"]').exists()).toBe(false)
  })

  it('shows admin link only for admin users', () => {
    const wrapper = mount(NavBar, {
      global: {
        plugins: [
          createTestingPinia({
            initialState: {
              user: { user: { id: '1', name: 'Alice', role: 'admin' } },
            },
          }),
        ],
      },
    })

    expect(wrapper.find('[data-testid="admin-link"]').exists()).toBe(true)
  })

  it('calls logout action when logout button clicked', async () => {
    const wrapper = mount(NavBar, {
      global: {
        plugins: [
          createTestingPinia({
            createSpy: vi.fn,
            initialState: {
              user: { user: { id: '1', name: 'Alice', role: 'user' } },
            },
          }),
        ],
      },
    })

    const store = useUserStore()

    await wrapper.find('[data-testid="logout-btn"]').trigger('click')

    expect(store.logout).toHaveBeenCalledOnce()
  })
})
```

### Testing the Store Directly

```ts
// stores/__tests__/useUserStore.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { useUserStore } from '@/stores/useUserStore'

describe('useUserStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('starts with no user', () => {
    const store = useUserStore()
    expect(store.user).toBeNull()
    expect(store.isLoggedIn).toBe(false)
  })

  it('logs in user', async () => {
    vi.stubGlobal(
      'fetch',
      vi.fn(() =>
        Promise.resolve({
          json: () => Promise.resolve({ id: '1', name: 'Alice', role: 'user' }),
        })
      )
    )

    const store = useUserStore()
    await store.login('alice@example.com', 'password')

    expect(store.user).toEqual({ id: '1', name: 'Alice', role: 'user' })
    expect(store.isLoggedIn).toBe(true)
  })

  it('logs out user', () => {
    const store = useUserStore()
    store.user = { id: '1', name: 'Alice', role: 'user' }

    store.logout()

    expect(store.user).toBeNull()
    expect(store.isLoggedIn).toBe(false)
  })
})
```

## Testing with Vue Router

### Mocking the Router

```ts
// components/__tests__/BreadcrumbNav.test.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import { createRouter, createMemoryHistory } from 'vue-router'
import BreadcrumbNav from '@/components/BreadcrumbNav.vue'

function createMockRouter(initialRoute = '/') {
  return createRouter({
    history: createMemoryHistory(),
    routes: [
      { path: '/', name: 'home', component: { template: '<div />' } },
      { path: '/users', name: 'users', component: { template: '<div />' } },
      { path: '/users/:id', name: 'user-detail', component: { template: '<div />' } },
    ],
  })
}

describe('BreadcrumbNav', () => {
  it('renders breadcrumbs for current route', async () => {
    const router = createMockRouter()
    await router.push('/users/123')
    await router.isReady()

    const wrapper = mount(BreadcrumbNav, {
      global: {
        plugins: [router],
      },
    })

    const crumbs = wrapper.findAll('[data-testid="breadcrumb"]')
    expect(crumbs.length).toBeGreaterThan(0)
  })

  it('navigates when breadcrumb is clicked', async () => {
    const router = createMockRouter()
    await router.push('/users/123')
    await router.isReady()

    const pushSpy = vi.spyOn(router, 'push')

    const wrapper = mount(BreadcrumbNav, {
      global: {
        plugins: [router],
      },
    })

    await wrapper.find('[data-testid="breadcrumb-home"]').trigger('click')

    expect(pushSpy).toHaveBeenCalledWith({ name: 'home' })
  })
})
```

### Testing Route Guards in Components

```ts
import { describe, it, expect, vi } from 'vitest'
import { mount, flushPromises } from '@vue/test-utils'
import { createRouter, createMemoryHistory } from 'vue-router'
import ProtectedPage from '@/pages/ProtectedPage.vue'
import { useUserStore } from '@/stores/useUserStore'
import { createTestingPinia } from '@pinia/testing'

describe('ProtectedPage', () => {
  it('redirects to login when user is not authenticated', async () => {
    const router = createRouter({
      history: createMemoryHistory(),
      routes: [
        { path: '/', component: { template: '<div>Home</div>' } },
        { path: '/login', name: 'login', component: { template: '<div>Login</div>' } },
        { path: '/protected', name: 'protected', component: ProtectedPage },
      ],
    })

    router.beforeEach((to) => {
      const userStore = useUserStore()
      if (to.name === 'protected' && !userStore.isLoggedIn) {
        return { name: 'login' }
      }
    })

    const pinia = createTestingPinia({
      initialState: { user: { user: null } },
    })

    mount(
      { template: '<router-view />' },
      { global: { plugins: [router, pinia] } }
    )

    await router.push('/protected')
    await router.isReady()
    await flushPromises()

    expect(router.currentRoute.value.name).toBe('login')
  })
})
```

## Snapshot Testing

Use sparingly. Snapshots are best for verifying complex rendered output does not change unexpectedly.

```ts
// components/__tests__/IconButton.test.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import IconButton from '@/components/IconButton.vue'

describe('IconButton', () => {
  it('matches snapshot for primary variant', () => {
    const wrapper = mount(IconButton, {
      props: { icon: 'save', variant: 'primary', label: 'Save' },
    })

    expect(wrapper.html()).toMatchSnapshot()
  })

  it('matches snapshot for danger variant', () => {
    const wrapper = mount(IconButton, {
      props: { icon: 'trash', variant: 'danger', label: 'Delete' },
    })

    expect(wrapper.html()).toMatchSnapshot()
  })

  // Inline snapshot for small, predictable output
  it('renders correct aria-label', () => {
    const wrapper = mount(IconButton, {
      props: { icon: 'save', variant: 'primary', label: 'Save' },
    })

    expect(wrapper.attributes('aria-label')).toMatchInlineSnapshot('"Save"')
  })
})
```

## Test Organization and Naming

### File Structure

```
src/
├── components/
│   ├── UserCard.vue
│   └── __tests__/
│       └── UserCard.test.ts       # Co-located tests
├── composables/
│   ├── useCounter.ts
│   └── __tests__/
│       └── useCounter.test.ts
├── stores/
│   ├── useUserStore.ts
│   └── __tests__/
│       └── useUserStore.test.ts
├── pages/
│   ├── HomePage.vue
│   └── __tests__/
│       └── HomePage.test.ts
└── test/
    ├── setup.ts                   # Global test setup
    └── helpers/                   # Shared test utilities
        ├── createWrapper.ts
        └── mockRouter.ts
```

### Naming Conventions

```ts
// Describe blocks: component or function name
describe('UserCard', () => {
  // Test names: describe behavior, not implementation
  // Pattern: "does X when Y" or "shows X for Y"

  // DO: describe user-visible behavior
  it('shows the user name in the heading', () => { /* ... */ })
  it('disables submit button when form is invalid', () => { /* ... */ })
  it('emits update event with new value on save', () => { /* ... */ })
  it('shows error message when API call fails', () => { /* ... */ })

  // DON'T: describe implementation details
  // it('sets isLoading ref to true', () => { /* ... */ })
  // it('calls the handleClick method', () => { /* ... */ })
  // it('updates the internal state', () => { /* ... */ })
})

// Group related tests with nested describe blocks
describe('UserCard', () => {
  describe('when user is admin', () => {
    it('shows admin badge', () => { /* ... */ })
    it('shows edit button', () => { /* ... */ })
  })

  describe('when user is guest', () => {
    it('does not show admin badge', () => { /* ... */ })
    it('does not show edit button', () => { /* ... */ })
  })
})
```

## Coverage Configuration

### Running Coverage

```bash
# Run tests with coverage
npx vitest run --coverage

# Watch mode (no coverage, faster feedback)
npx vitest

# Run specific test file
npx vitest run src/components/__tests__/UserCard.test.ts

# Run tests matching a pattern
npx vitest run -t "UserCard"
```

### Coverage Thresholds in vitest.config.ts

```ts
// vitest.config.ts (coverage section)
test: {
  coverage: {
    provider: 'v8',
    reporter: ['text', 'json', 'html', 'lcov'],
    include: ['src/**/*.{ts,vue}'],
    exclude: [
      'src/**/*.d.ts',
      'src/**/*.{test,spec}.ts',
      'src/test/**',
      'src/main.ts',
      'src/App.vue',
      'src/router/index.ts',
    ],
    thresholds: {
      statements: 80,
      branches: 80,
      functions: 80,
      lines: 80,
    },
    // Per-file thresholds for critical paths
    watermarks: {
      statements: [70, 90],
      branches: [70, 90],
      functions: [70, 90],
      lines: [70, 90],
    },
  },
}
```

## Component Testing vs Unit Testing vs E2E

| Aspect | Unit Test | Component Test | E2E Test |
|---|---|---|---|
| **Scope** | Single function/composable | Single component in isolation | Full application flow |
| **Speed** | Fastest (ms) | Fast (10-100ms) | Slow (seconds) |
| **Dependencies** | None (pure logic) | Mocked/stubbed children | Real browser, real APIs |
| **Tool** | Vitest | Vitest + Vue Test Utils | Cypress / Playwright |
| **What to test** | Business logic, transforms, validators | Rendering, props, emits, interactions | User journeys, critical flows |
| **Proportion** | 60% of tests | 30% of tests | 10% of tests |

### What Goes Where

```ts
// UNIT TEST: Pure logic, no Vue dependency
// composables/__tests__/useCounter.test.ts
describe('useCounter', () => {
  it('increments count', () => {
    const { count, increment } = useCounter()
    increment()
    expect(count.value).toBe(1)
  })
})

// COMPONENT TEST: Vue component behavior
// components/__tests__/LoginForm.test.ts
describe('LoginForm', () => {
  it('disables submit when fields are empty', () => {
    const wrapper = mount(LoginForm)
    expect(wrapper.find('[data-testid="submit"]').attributes('disabled')).toBeDefined()
  })
})

// E2E TEST: Full user journey (Playwright/Cypress, not Vitest)
// e2e/login.spec.ts
test('user can log in and see dashboard', async ({ page }) => {
  await page.goto('/login')
  await page.fill('[data-testid="email"]', 'alice@example.com')
  await page.fill('[data-testid="password"]', 'password')
  await page.click('[data-testid="submit"]')
  await expect(page.locator('[data-testid="dashboard"]')).toBeVisible()
})
```

## Best Practices

### DO

- Use `data-testid` attributes for querying elements in tests.
- Write tests that verify behavior, not implementation details.
- Use factory functions to reduce boilerplate across tests.
- Clean up mocks with `vi.restoreAllMocks()` in `afterEach`.
- Use `flushPromises()` for async operations in components.
- Test error states and edge cases, not just the happy path.
- Co-locate test files alongside source files in `__tests__/` directories.
- Use `it.each` for parameterized tests with multiple inputs.
- Keep tests independent; avoid shared mutable state between tests.
- Prefer `mount` for tests that verify child component integration.

### DON'T

- Do not test framework behavior (Vue reactivity, v-model internals).
- Do not rely on CSS class names or HTML structure for assertions; use `data-testid`.
- Do not test implementation details like internal ref values or private methods.
- Do not write snapshot tests for frequently changing components.
- Do not mock everything; real Pinia stores and routers are fast enough for most tests.
- Do not use `wrapper.vm` to access internal component state in tests.
- Do not write E2E-style tests with Vitest; use Playwright or Cypress for those.
- Do not skip writing tests for "simple" components; simple now does not mean simple later.

## Guidelines

### Essential

- Configure Vitest with jsdom environment for all Vue component tests.
- Use `@vue/test-utils` mount/shallowMount for rendering components.
- Use `createTestingPinia` when components depend on Pinia stores.
- Run `vi.restoreAllMocks()` after each test to prevent leaking mocks.
- Await all user interaction triggers (`trigger`, `setValue`) as they are async.

### Recommended

- Create a `withSetup` helper for testing composables that require lifecycle hooks.
- Use `createMemoryHistory` for router mocks in tests (avoids browser history side effects).
- Set up per-file coverage thresholds for critical business logic modules.
- Use `vi.useFakeTimers` for testing debounced/throttled behavior.
- Write a shared test setup file for global stubs and cleanup.

### Advanced

- Use `vi.mock` with factory functions for complex module mocking scenarios.
- Test route guards independently from the components they protect.
- Use snapshot tests only for stable, rarely-changing UI components.
- Configure CI to fail on coverage regression (compare against baseline).
- Create custom matchers with `expect.extend` for domain-specific assertions.

## Benefits

- Fast feedback loops. Vitest runs tests in milliseconds, enabling rapid iteration.
- Vite-native. Shares configuration and plugin pipeline with the dev server.
- Type-safe mocks. `vi.mocked()` preserves TypeScript types on mocked functions.
- First-class Vue support. Vue Test Utils integrates seamlessly with Vitest.
- Confident refactoring. Behavior-focused tests survive implementation changes.
- Coverage enforcement. Threshold configuration prevents quality regression.

## Related

- [vue-composables.md](./vue-composables.md) - Composable patterns and extraction
- [vue-component-structure.md](./vue-component-structure.md) - Component organization conventions
- [vue-error-handling.md](./vue-error-handling.md) - Error boundaries and error handling
- [pinia-state-management.md](../03-state-management/pinia-state-management.md) - Pinia store patterns
