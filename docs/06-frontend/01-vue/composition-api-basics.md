# Composition API Basics

Vue 3 Composition API for component logic. Organize code by feature, not lifecycle.

## Principle

Composition API uses setup() function to define component logic. More flexible and reusable than Options API.

## Basic Setup

Define component with setup():

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

// Reactive state
const count = ref(0)

// Computed property
const doubleCount = computed(() => count.value * 2)

// Method
function increment() {
  count.value++
}
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <p>Double: {{ doubleCount }}</p>
    <button @click="increment">Increment</button>
  </div>
</template>
```

script setup is syntactic sugar for Composition API.

## Reactive State

Use ref() for reactive values:

```vue
<script setup lang="ts">
import { ref } from 'vue'

// Primitive values
const count = ref(0)
const message = ref('Hello')
const isActive = ref(true)

// Access with .value in script
function updateCount() {
  count.value = 10
}

// Objects and arrays
const user = ref({ name: 'John', email: 'john@example.com' })
const items = ref(['apple', 'banana', 'orange'])
</script>

<template>
  <!-- No .value needed in template -->
  <div>{{ count }}</div>
  <div>{{ message }}</div>
  <div>{{ user.name }}</div>
</template>
```

## Reactive Objects

Use reactive() for objects:

```vue
<script setup lang="ts">
import { reactive } from 'vue'

// Reactive object
const state = reactive({
  count: 0,
  message: 'Hello',
  user: {
    name: 'John',
    email: 'john@example.com'
  }
})

function updateUser() {
  // No .value needed
  state.user.name = 'Jane'
}
</script>

<template>
  <div>{{ state.count }}</div>
  <div>{{ state.user.name }}</div>
</template>
```

ref() vs reactive():
- ref: Any value, access with .value
- reactive: Objects only, no .value needed

## Computed Properties

Derived state with computed():

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const firstName = ref('John')
const lastName = ref('Doe')

// Read-only computed
const fullName = computed(() => {
  return `${firstName.value} ${lastName.value}`
})

// Writable computed
const upperName = computed({
  get: () => fullName.value.toUpperCase(),
  set: (value: string) => {
    const parts = value.split(' ')
    firstName.value = parts[0]
    lastName.value = parts[1]
  }
})
</script>

<template>
  <div>{{ fullName }}</div>
  <input v-model="upperName" />
</template>
```

## Lifecycle Hooks

Component lifecycle with onMounted, onUnmounted:

```vue
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const data = ref<User[]>([])

onMounted(async () => {
  console.log('Component mounted')
  data.value = await fetchUsers()
})

onUnmounted(() => {
  console.log('Component unmounted')
  // Cleanup
})
</script>
```

Available lifecycle hooks:
- onBeforeMount
- onMounted
- onBeforeUpdate
- onUpdated
- onBeforeUnmount
- onUnmounted

## Watchers

React to state changes with watch():

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'

const count = ref(0)

// Watch single ref
watch(count, (newValue, oldValue) => {
  console.log(`Count changed from ${oldValue} to ${newValue}`)
})

// Watch multiple sources
const firstName = ref('John')
const lastName = ref('Doe')

watch([firstName, lastName], ([newFirst, newLast]) => {
  console.log(`Name: ${newFirst} ${newLast}`)
})

// Watch with options
watch(
  count,
  (newValue) => {
    console.log('Count:', newValue)
  },
  { immediate: true } // Run immediately on mount
)
</script>
```

## Props

Receive props with defineProps():

```vue
<script setup lang="ts">
import { computed } from 'vue'

interface Props {
  userId: string
  name?: string
  isActive?: boolean
}

const props = defineProps<Props>()

// Use props
const displayName = computed(() => props.name || 'Unknown')
</script>

<template>
  <div>User ID: {{ userId }}</div>
  <div>Name: {{ displayName }}</div>
</template>
```

With defaults:

```vue
<script setup lang="ts">
interface Props {
  userId: string
  name?: string
  isActive?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  name: 'Unknown',
  isActive: true
})
</script>
```

## Emits

Emit events with defineEmits():

```vue
<script setup lang="ts">
interface Emits {
  (e: 'update', value: string): void
  (e: 'delete', id: string): void
}

const emit = defineEmits<Emits>()

function handleUpdate(value: string) {
  emit('update', value)
}

function handleDelete() {
  emit('delete', props.userId)
}
</script>

<template>
  <button @click="handleUpdate('new value')">Update</button>
  <button @click="handleDelete">Delete</button>
</template>
```

## Composables

Extract reusable logic:

```ts
// composables/useCounter.ts
import { ref, computed } from 'vue'

export function useCounter(initialValue = 0) {
  const count = ref(initialValue)
  const doubled = computed(() => count.value * 2)

  function increment() {
    count.value++
  }

  function decrement() {
    count.value--
  }

  return {
    count,
    doubled,
    increment,
    decrement
  }
}
```

Usage:

```vue
<script setup lang="ts">
import { useCounter } from '@/composables/useCounter'

const { count, doubled, increment, decrement } = useCounter(10)
</script>

<template>
  <div>Count: {{ count }}</div>
  <div>Doubled: {{ doubled }}</div>
  <button @click="increment">+</button>
  <button @click="decrement">-</button>
</template>
```

## Template Refs

Access DOM elements:

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const inputRef = ref<HTMLInputElement | null>(null)

onMounted(() => {
  inputRef.value?.focus()
})
</script>

<template>
  <input ref="inputRef" type="text" />
</template>
```

## Async Setup

Async data loading:

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const user = ref<User | null>(null)
const loading = ref(false)
const error = ref<Error | null>(null)

onMounted(async () => {
  loading.value = true
  try {
    const response = await fetch('/api/users/123')
    user.value = await response.json()
  } catch (e) {
    error.value = e as Error
  } finally {
    loading.value = false
  }
})
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">Error: {{ error.message }}</div>
  <div v-else-if="user">
    <h1>{{ user.name }}</h1>
    <p>{{ user.email }}</p>
  </div>
</template>
```

## Guidelines

**State Management:**
- Use ref() for primitives
- Use reactive() for objects (or ref with objects)
- Use computed() for derived state

**Composition:**
- Extract reusable logic to composables
- Organize by feature, not lifecycle
- Keep components focused and small

**Props and Emits:**
- Define TypeScript interfaces
- Use defineProps and defineEmits
- Provide defaults where appropriate

**Watchers:**
- Use sparingly (computed is often better)
- Clean up side effects in onUnmounted
- Consider immediate option

## Benefits

Reusability. Extract and share logic with composables.

Type safety. Full TypeScript support.

Organization. Group related logic together.

Flexibility. More flexible than Options API.

## Related

- [composables-patterns.md](./composables-patterns.md) - Reusable logic patterns
- [typescript-vue-integration.md](../02-typescript/typescript-vue-integration.md) - TypeScript in Vue
- [tanstack-query-vue.md](../03-state-management/tanstack-query-vue.md) - Server state
