# TypeScript in Vue

TypeScript integration with Vue 3. Type-safe components, props, and emits.

## Principle

Use TypeScript for all Vue components. Full type safety from props to state to emits.

## Component Types

Basic component with TypeScript:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

// State with types
const count = ref<number>(0)
const message = ref<string>('Hello')

// Arrays and objects
const users = ref<User[]>([])
const settings = ref<AppSettings>({
  theme: 'light',
  language: 'en'
})

// Computed with inferred return type
const doubled = computed(() => count.value * 2)

// Explicitly typed computed
const formattedMessage = computed<string>(() => {
  return message.value.toUpperCase()
})
</script>
```

## Props Types

Strongly-typed props:

```vue
<script setup lang="ts">
interface Props {
  userId: string
  name: string
  age?: number
  isActive?: boolean
  roles: string[]
}

const props = defineProps<Props>()

// With defaults
const propsWithDefaults = withDefaults(defineProps<Props>(), {
  age: 18,
  isActive: true
})
</script>

<template>
  <div>
    <h1>{{ name }}</h1>
    <p>Age: {{ age }}</p>
    <p>Active: {{ isActive }}</p>
  </div>
</template>
```

## Emits Types

Type-safe event emitters:

```vue
<script setup lang="ts">
interface Emits {
  (e: 'update', value: string): void
  (e: 'delete', id: string): void
  (e: 'submit', data: FormData): void
}

const emit = defineEmits<Emits>()

function handleUpdate(value: string) {
  emit('update', value) // Type-checked
}

function handleDelete() {
  emit('delete', props.userId) // Type-checked
}
</script>
```

## Composables

Typed composables:

```ts
// composables/useUser.ts
import { ref, computed } from 'vue'
import type { Ref } from 'vue'

export interface User {
  id: string
  email: string
  name: string
  isActive: boolean
}

export interface UseUserReturn {
  user: Ref<User | null>
  loading: Ref<boolean>
  error: Ref<Error | null>
  fetchUser: (id: string) => Promise<void>
  updateUser: (updates: Partial<User>) => void
}

export function useUser(): UseUserReturn {
  const user = ref<User | null>(null)
  const loading = ref(false)
  const error = ref<Error | null>(null)

  async function fetchUser(id: string) {
    loading.value = true
    error.value = null
    try {
      const response = await fetch(`/api/users/${id}`)
      user.value = await response.json()
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }

  function updateUser(updates: Partial<User>) {
    if (user.value) {
      user.value = { ...user.value, ...updates }
    }
  }

  return {
    user,
    loading,
    error,
    fetchUser,
    updateUser
  }
}
```

Usage:

```vue
<script setup lang="ts">
import { useUser } from '@/composables/useUser'
import { onMounted } from 'vue'

const props = defineProps<{ userId: string }>()

const { user, loading, error, fetchUser } = useUser()

onMounted(() => {
  fetchUser(props.userId)
})
</script>
```

## Template Refs

Typed template refs:

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

// HTML element
const inputRef = ref<HTMLInputElement | null>(null)

// Component ref
import UserCard from './UserCard.vue'
const userCardRef = ref<InstanceType<typeof UserCard> | null>(null)

onMounted(() => {
  inputRef.value?.focus()
  userCardRef.value?.someMethod()
})
</script>

<template>
  <input ref="inputRef" type="text" />
  <UserCard ref="userCardRef" />
</template>
```

## API Types

Define API response types:

```ts
// types/api.ts
export interface ApiResponse<T> {
  data: T
  message: string
  status: number
}

export interface PaginatedResponse<T> {
  items: T[]
  total: number
  page: number
  limit: number
}

export interface User {
  id: string
  email: string
  name: string
  createdAt: string
}

export interface CreateUserRequest {
  email: string
  name: string
}

export interface UpdateUserRequest {
  name?: string
  email?: string
}
```

Usage:

```vue
<script setup lang="ts">
import { useQuery } from '@tanstack/vue-query'
import type { User, ApiResponse } from '@/types/api'

const { data } = useQuery({
  queryKey: ['user', props.userId],
  queryFn: async (): Promise<ApiResponse<User>> => {
    const response = await fetch(`/api/users/${props.userId}`)
    return response.json()
  }
})
</script>

<template>
  <div v-if="data">
    {{ data.data.name }}
  </div>
</template>
```

## Generic Components

Reusable typed components:

```vue
<!-- components/DataTable.vue -->
<script setup lang="ts" generic="T">
import { computed } from 'vue'

interface Props {
  items: T[]
  columns: Column<T>[]
}

interface Column<T> {
  key: keyof T
  label: string
  formatter?: (value: T[keyof T]) => string
}

const props = defineProps<Props>()

const formattedData = computed(() => {
  return props.items.map(item => {
    return props.columns.map(col => {
      const value = item[col.key]
      return col.formatter ? col.formatter(value) : String(value)
    })
  })
})
</script>

<template>
  <table>
    <thead>
      <tr>
        <th v-for="col in columns" :key="String(col.key)">
          {{ col.label }}
        </th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="(row, i) in formattedData" :key="i">
        <td v-for="(cell, j) in row" :key="j">
          {{ cell }}
        </td>
      </tr>
    </tbody>
  </table>
</template>
```

Usage:

```vue
<script setup lang="ts">
import DataTable from '@/components/DataTable.vue'
import type { User } from '@/types/api'

const users = ref<User[]>([])

const columns = [
  { key: 'name' as const, label: 'Name' },
  { key: 'email' as const, label: 'Email' },
  {
    key: 'createdAt' as const,
    label: 'Created',
    formatter: (value: string) => new Date(value).toLocaleDateString()
  }
]
</script>

<template>
  <DataTable :items="users" :columns="columns" />
</template>
```

## Type Guards

Runtime type checking:

```ts
// utils/typeGuards.ts
import type { User, Admin } from '@/types'

export function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'email' in value &&
    'name' in value
  )
}

export function isAdmin(user: User): user is Admin {
  return 'adminLevel' in user
}
```

Usage:

```vue
<script setup lang="ts">
import { isUser, isAdmin } from '@/utils/typeGuards'

async function fetchData() {
  const response = await fetch('/api/user')
  const data = await response.json()

  if (isUser(data)) {
    // data is User type here
    console.log(data.email)

    if (isAdmin(data)) {
      // data is Admin type here
      console.log(data.adminLevel)
    }
  }
}
</script>
```

## Enum Types

Use TypeScript enums:

```ts
// types/enums.ts
export enum UserRole {
  Admin = 'admin',
  User = 'user',
  Guest = 'guest'
}

export enum LoanStatus {
  Active = 'active',
  Returned = 'returned',
  Overdue = 'overdue'
}
```

Usage:

```vue
<script setup lang="ts">
import { UserRole } from '@/types/enums'

const props = defineProps<{
  role: UserRole
}>()

const isAdmin = computed(() => props.role === UserRole.Admin)
</script>

<template>
  <div v-if="isAdmin">
    Admin panel
  </div>
</template>
```

## Configuration

TypeScript configuration:

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "module": "ESNext",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "preserve",

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,

    /* Path aliases */
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*.ts", "src/**/*.tsx", "src/**/*.vue"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

## Guidelines

**Props:**
- Always define prop types with interface
- Use withDefaults for optional props
- Avoid any type

**Emits:**
- Define emit signatures
- Type-check event payloads
- Use descriptive event names

**Composables:**
- Export return type interface
- Type all parameters and returns
- Document complex types

**API Types:**
- Co-locate with API service
- Share types between frontend and backend
- Use DTOs for requests/responses

## Benefits

Type safety. Catch errors at compile time.

IntelliSense. Better IDE autocomplete.

Refactoring. Safe large-scale changes.

Documentation. Types document intent.

## Related

- [composition-api-basics.md](../01-vue/composition-api-basics.md) - Vue patterns
- [pinia-state-management.md](../03-state-management/pinia-state-management.md) - Typed stores
- [tanstack-query-vue.md](../03-state-management/tanstack-query-vue.md) - Typed queries
