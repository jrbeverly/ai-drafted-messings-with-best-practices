# Pinia State Management

Pinia for client-side state management. Handle UI state, user preferences, and temporary data.

## Principle

Pinia manages client state only. Server state goes in TanStack Query. Keep concerns separated.

## Installation

Add Pinia:

```bash
npm install pinia
```

## Setup

Configure Pinia:

```ts
// main.ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'

const pinia = createPinia()

createApp(App)
  .use(pinia)
  .mount('#app')
```

## Basic Store

Define store with defineStore:

```ts
// stores/useCounterStore.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useCounterStore = defineStore('counter', () => {
  // State
  const count = ref(0)

  // Getters
  const doubleCount = computed(() => count.value * 2)

  // Actions
  function increment() {
    count.value++
  }

  function decrement() {
    count.value--
  }

  return {
    count,
    doubleCount,
    increment,
    decrement
  }
})
```

Usage in component:

```vue
<script setup lang="ts">
import { useCounterStore } from '@/stores/useCounterStore'

const counter = useCounterStore()
</script>

<template>
  <div>
    <p>Count: {{ counter.count }}</p>
    <p>Double: {{ counter.doubleCount }}</p>
    <button @click="counter.increment()">+</button>
    <button @click="counter.decrement()">-</button>
  </div>
</template>
```

## UI State Store

Manage UI state:

```ts
// stores/useUIStore.ts
import { defineStore } from 'pinia'
import { ref } from 'vue'

export const useUIStore = defineStore('ui', () => {
  // Sidebar state
  const isSidebarOpen = ref(false)

  function toggleSidebar() {
    isSidebarOpen.value = !isSidebarOpen.value
  }

  function openSidebar() {
    isSidebarOpen.value = true
  }

  function closeSidebar() {
    isSidebarOpen.value = false
  }

  // Modal state
  const isModalOpen = ref(false)
  const modalContent = ref<string | null>(null)

  function openModal(content: string) {
    isModalOpen.value = true
    modalContent.value = content
  }

  function closeModal() {
    isModalOpen.value = false
    modalContent.value = null
  }

  // Theme
  const theme = ref<'light' | 'dark'>('light')

  function toggleTheme() {
    theme.value = theme.value === 'light' ? 'dark' : 'light'
  }

  return {
    isSidebarOpen,
    toggleSidebar,
    openSidebar,
    closeSidebar,
    isModalOpen,
    modalContent,
    openModal,
    closeModal,
    theme,
    toggleTheme
  }
})
```

## Auth State

Current user and auth status:

```ts
// stores/useAuthStore.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

interface User {
  id: string
  email: string
  name: string
  roles: string[]
}

export const useAuthStore = defineStore('auth', () => {
  const user = ref<User | null>(null)
  const token = ref<string | null>(null)

  const isAuthenticated = computed(() => !!token.value)
  const isAdmin = computed(() => user.value?.roles.includes('admin') ?? false)

  function setAuth(newUser: User, newToken: string) {
    user.value = newUser
    token.value = newToken
    localStorage.setItem('token', newToken)
  }

  function clearAuth() {
    user.value = null
    token.value = null
    localStorage.removeItem('token')
  }

  function loadFromStorage() {
    const savedToken = localStorage.getItem('token')
    if (savedToken) {
      token.value = savedToken
      // Fetch user data with token
    }
  }

  return {
    user,
    token,
    isAuthenticated,
    isAdmin,
    setAuth,
    clearAuth,
    loadFromStorage
  }
})
```

## Form State

Temporary form data:

```ts
// stores/useCreateUserFormStore.ts
import { defineStore } from 'pinia'
import { ref } from 'vue'

interface CreateUserForm {
  email: string
  name: string
  role: string
}

export const useCreateUserFormStore = defineStore('createUserForm', () => {
  const form = ref<CreateUserForm>({
    email: '',
    name: '',
    role: 'user'
  })

  const errors = ref<Record<string, string>>({})

  function updateField(field: keyof CreateUserForm, value: string) {
    form.value[field] = value
    // Clear error when user types
    delete errors.value[field]
  }

  function setErrors(newErrors: Record<string, string>) {
    errors.value = newErrors
  }

  function reset() {
    form.value = {
      email: '',
      name: '',
      role: 'user'
    }
    errors.value = {}
  }

  return {
    form,
    errors,
    updateField,
    setErrors,
    reset
  }
})
```

## Persisted State

Persist to localStorage:

```ts
// stores/usePreferencesStore.ts
import { defineStore } from 'pinia'
import { ref, watch } from 'vue'

export const usePreferencesStore = defineStore('preferences', () => {
  const itemsPerPage = ref(10)
  const defaultView = ref<'grid' | 'list'>('grid')
  const notifications = ref(true)

  // Load from localStorage on init
  function load() {
    const saved = localStorage.getItem('preferences')
    if (saved) {
      const parsed = JSON.parse(saved)
      itemsPerPage.value = parsed.itemsPerPage ?? 10
      defaultView.value = parsed.defaultView ?? 'grid'
      notifications.value = parsed.notifications ?? true
    }
  }

  // Save to localStorage on change
  watch(
    [itemsPerPage, defaultView, notifications],
    () => {
      localStorage.setItem('preferences', JSON.stringify({
        itemsPerPage: itemsPerPage.value,
        defaultView: defaultView.value,
        notifications: notifications.value
      }))
    }
  )

  // Initialize
  load()

  return {
    itemsPerPage,
    defaultView,
    notifications
  }
})
```

## Store Composition

Use one store from another:

```ts
// stores/useCartStore.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { useAuthStore } from './useAuthStore'

export const useCartStore = defineStore('cart', () => {
  const authStore = useAuthStore()

  const items = ref<CartItem[]>([])

  const total = computed(() => {
    return items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
  })

  // Discount for authenticated users
  const discount = computed(() => {
    return authStore.isAuthenticated ? total.value * 0.1 : 0
  })

  const finalTotal = computed(() => {
    return total.value - discount.value
  })

  function addItem(item: CartItem) {
    items.value.push(item)
  }

  function removeItem(id: string) {
    items.value = items.value.filter(item => item.id !== id)
  }

  return {
    items,
    total,
    discount,
    finalTotal,
    addItem,
    removeItem
  }
})
```

## Async Actions

Handle async operations:

```ts
// stores/useNotificationStore.ts
import { defineStore } from 'pinia'
import { ref } from 'vue'

interface Notification {
  id: string
  message: string
  type: 'success' | 'error' | 'info'
}

export const useNotificationStore = defineStore('notifications', () => {
  const notifications = ref<Notification[]>([])

  function show(message: string, type: Notification['type'] = 'info') {
    const id = crypto.randomUUID()
    notifications.value.push({ id, message, type })

    // Auto-dismiss after 3 seconds
    setTimeout(() => {
      dismiss(id)
    }, 3000)
  }

  function dismiss(id: string) {
    notifications.value = notifications.value.filter(n => n.id !== id)
  }

  return {
    notifications,
    show,
    dismiss
  }
})
```

## Reset Store

Reset to initial state:

```ts
// stores/useFilterStore.ts
import { defineStore } from 'pinia'
import { ref } from 'vue'

export const useFilterStore = defineStore('filters', () => {
  const searchQuery = ref('')
  const selectedCategory = ref<string | null>(null)
  const minPrice = ref(0)
  const maxPrice = ref(1000)

  function reset() {
    searchQuery.value = ''
    selectedCategory.value = null
    minPrice.value = 0
    maxPrice.value = 1000
  }

  return {
    searchQuery,
    selectedCategory,
    minPrice,
    maxPrice,
    reset
  }
})
```

## Testing Stores

Unit test stores:

```ts
// stores/__tests__/useCounterStore.spec.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { useCounterStore } from '../useCounterStore'

describe('Counter Store', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('increments count', () => {
    const store = useCounterStore()

    expect(store.count).toBe(0)

    store.increment()

    expect(store.count).toBe(1)
  })

  it('calculates double count', () => {
    const store = useCounterStore()

    store.count = 5

    expect(store.doubleCount).toBe(10)
  })
})
```

## Guidelines

**What to Store in Pinia:**
- UI state (sidebar open/closed, modals)
- User preferences (theme, language, items per page)
- Current user and auth token
- Temporary form data
- Client-side filters and sorting

**What NOT to Store:**
- Server data (use TanStack Query)
- Derived server data
- Cache (handled by TanStack Query)

**Store Organization:**
- One store per feature/domain
- Composition API style (setup syntax)
- Co-locate related state and actions

**Performance:**
- Use computed for derived state
- Don't duplicate server state
- Reset stores when appropriate

## Benefits

Type-safe. Full TypeScript support.

Devtools. Vue DevTools integration.

Simple API. Intuitive and straightforward.

Modular. One store per feature.

## Related

- [tanstack-query-vue.md](./tanstack-query-vue.md) - Server state (don't duplicate)
- [composition-api-basics.md](../01-vue/composition-api-basics.md) - Vue composables
- [typescript-vue-integration.md](../02-typescript/typescript-vue-integration.md) - TypeScript patterns
