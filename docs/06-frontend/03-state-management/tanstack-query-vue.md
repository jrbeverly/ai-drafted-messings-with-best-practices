# TanStack Query for Vue

TanStack Query (Vue Query) for server state management. Handle data fetching, caching, and synchronization.

## Principle

TanStack Query manages server state (data from APIs). Pinia manages client state (UI state). Keep them separate.

## Installation

Add TanStack Query:

```bash
npm install @tanstack/vue-query
```

## Setup

Configure QueryClient:

```ts
// main.ts
import { createApp } from 'vue'
import { VueQueryPlugin, QueryClient } from '@tanstack/vue-query'
import App from './App.vue'

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 minutes
      retry: 1
    }
  }
})

createApp(App)
  .use(VueQueryPlugin, { queryClient })
  .mount('#app')
```

## Basic Query

Fetch data with useQuery:

```vue
<script setup lang="ts">
import { useQuery } from '@tanstack/vue-query'

interface User {
  id: string
  name: string
  email: string
}

const { data, isLoading, isError, error } = useQuery({
  queryKey: ['user', '123'],
  queryFn: async () => {
    const response = await fetch('/api/users/123')
    if (!response.ok) throw new Error('Failed to fetch user')
    return response.json() as Promise<User>
  }
})
</script>

<template>
  <div v-if="isLoading">Loading...</div>
  <div v-else-if="isError">Error: {{ error.message }}</div>
  <div v-else-if="data">
    <h1>{{ data.name }}</h1>
    <p>{{ data.email }}</p>
  </div>
</template>
```

## Query Keys

Unique identifiers for queries:

```ts
// Simple key
['users']

// With parameters
['user', userId]

// Complex key
['users', { status: 'active', role: 'admin' }]

// Hierarchical keys
['users', 'list', { page: 1, limit: 10 }]
```

## Dependent Queries

Query depends on another query:

```vue
<script setup lang="ts">
import { useQuery } from '@tanstack/vue-query'

const props = defineProps<{ userId: string }>()

// First query: Get user
const { data: user } = useQuery({
  queryKey: ['user', props.userId],
  queryFn: () => fetchUser(props.userId)
})

// Second query: Get user's posts (depends on user)
const { data: posts } = useQuery({
  queryKey: ['posts', props.userId],
  queryFn: () => fetchUserPosts(props.userId),
  enabled: !!user.value // Only run when user exists
})
</script>
```

## Mutations

Update data with useMutation:

```vue
<script setup lang="ts">
import { useMutation, useQueryClient } from '@tanstack/vue-query'

const queryClient = useQueryClient()

const createUserMutation = useMutation({
  mutationFn: async (user: CreateUserRequest) => {
    const response = await fetch('/api/users', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(user)
    })
    if (!response.ok) throw new Error('Failed to create user')
    return response.json()
  },
  onSuccess: () => {
    // Invalidate and refetch
    queryClient.invalidateQueries({ queryKey: ['users'] })
  }
})

async function handleSubmit(formData: CreateUserRequest) {
  await createUserMutation.mutateAsync(formData)
}
</script>

<template>
  <form @submit.prevent="handleSubmit(formData)">
    <!-- Form fields -->
    <button
      type="submit"
      :disabled="createUserMutation.isPending"
    >
      {{ createUserMutation.isPending ? 'Creating...' : 'Create User' }}
    </button>
  </form>
</template>
```

## Optimistic Updates

Update UI immediately:

```vue
<script setup lang="ts">
import { useMutation, useQueryClient } from '@tanstack/vue-query'

const queryClient = useQueryClient()

const updateUserMutation = useMutation({
  mutationFn: async (user: User) => {
    const response = await fetch(`/api/users/${user.id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(user)
    })
    return response.json()
  },
  onMutate: async (newUser) => {
    // Cancel outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['user', newUser.id] })

    // Snapshot previous value
    const previousUser = queryClient.getQueryData(['user', newUser.id])

    // Optimistically update
    queryClient.setQueryData(['user', newUser.id], newUser)

    // Return context with previous value
    return { previousUser }
  },
  onError: (err, newUser, context) => {
    // Rollback on error
    queryClient.setQueryData(['user', newUser.id], context?.previousUser)
  },
  onSettled: (data, err, variables) => {
    // Refetch after mutation
    queryClient.invalidateQueries({ queryKey: ['user', variables.id] })
  }
})
</script>
```

## Pagination

Paginated queries:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import { useQuery } from '@tanstack/vue-query'

const page = ref(1)
const limit = 10

const { data, isLoading, isPlaceholderData } = useQuery({
  queryKey: ['users', { page: page.value, limit }],
  queryFn: () => fetchUsers(page.value, limit),
  placeholderData: (previousData) => previousData // Keep previous data while loading
})

const hasNextPage = computed(() => {
  return data.value && data.value.length === limit
})

function nextPage() {
  if (hasNextPage.value) page.value++
}

function previousPage() {
  if (page.value > 1) page.value--
}
</script>

<template>
  <div>
    <div v-if="isLoading">Loading...</div>
    <ul v-else>
      <li v-for="user in data" :key="user.id">
        {{ user.name }}
      </li>
    </ul>

    <button @click="previousPage" :disabled="page === 1">
      Previous
    </button>
    <span>Page {{ page }}</span>
    <button
      @click="nextPage"
      :disabled="!hasNextPage || isPlaceholderData"
    >
      Next
    </button>
  </div>
</template>
```

## Infinite Queries

Infinite scroll:

```vue
<script setup lang="ts">
import { useInfiniteQuery } from '@tanstack/vue-query'

const {
  data,
  fetchNextPage,
  hasNextPage,
  isFetchingNextPage
} = useInfiniteQuery({
  queryKey: ['users'],
  queryFn: async ({ pageParam = 0 }) => {
    const response = await fetch(`/api/users?page=${pageParam}&limit=10`)
    return response.json()
  },
  getNextPageParam: (lastPage, allPages) => {
    return lastPage.length === 10 ? allPages.length : undefined
  },
  initialPageParam: 0
})
</script>

<template>
  <div>
    <div v-for="page in data?.pages" :key="page">
      <div v-for="user in page" :key="user.id">
        {{ user.name }}
      </div>
    </div>

    <button
      v-if="hasNextPage"
      @click="fetchNextPage()"
      :disabled="isFetchingNextPage"
    >
      {{ isFetchingNextPage ? 'Loading...' : 'Load More' }}
    </button>
  </div>
</template>
```

## Query Invalidation

Invalidate stale queries:

```ts
// Invalidate specific query
queryClient.invalidateQueries({ queryKey: ['user', userId] })

// Invalidate all users queries
queryClient.invalidateQueries({ queryKey: ['users'] })

// Invalidate multiple
queryClient.invalidateQueries({ queryKey: ['users'] })
queryClient.invalidateQueries({ queryKey: ['posts'] })

// Refetch immediately
await queryClient.invalidateQueries({
  queryKey: ['users'],
  refetchType: 'active'
})
```

## Error Handling

Handle errors gracefully:

```vue
<script setup lang="ts">
import { useQuery } from '@tanstack/vue-query'

const { data, error, isError, refetch } = useQuery({
  queryKey: ['user', userId],
  queryFn: async () => {
    const response = await fetch(`/api/users/${userId}`)
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`)
    }
    return response.json()
  },
  retry: (failureCount, error) => {
    // Don't retry on 404
    if (error.message.includes('404')) return false
    // Retry up to 3 times for other errors
    return failureCount < 3
  }
})
</script>

<template>
  <div v-if="isError">
    <p>Error: {{ error.message }}</p>
    <button @click="refetch()">Retry</button>
  </div>
</template>
```

## API Service Pattern

Centralize API calls:

```ts
// services/userApi.ts
export const userApi = {
  getAll: async (): Promise<User[]> => {
    const response = await fetch('/api/users')
    if (!response.ok) throw new Error('Failed to fetch users')
    return response.json()
  },

  getById: async (id: string): Promise<User> => {
    const response = await fetch(`/api/users/${id}`)
    if (!response.ok) throw new Error('Failed to fetch user')
    return response.json()
  },

  create: async (user: CreateUserRequest): Promise<User> => {
    const response = await fetch('/api/users', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(user)
    })
    if (!response.ok) throw new Error('Failed to create user')
    return response.json()
  },

  update: async (id: string, user: UpdateUserRequest): Promise<User> => {
    const response = await fetch(`/api/users/${id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(user)
    })
    if (!response.ok) throw new Error('Failed to update user')
    return response.json()
  },

  delete: async (id: string): Promise<void> => {
    const response = await fetch(`/api/users/${id}`, {
      method: 'DELETE'
    })
    if (!response.ok) throw new Error('Failed to delete user')
  }
}
```

Usage:

```vue
<script setup lang="ts">
import { useQuery, useMutation } from '@tanstack/vue-query'
import { userApi } from '@/services/userApi'

const { data: users } = useQuery({
  queryKey: ['users'],
  queryFn: userApi.getAll
})

const createMutation = useMutation({
  mutationFn: userApi.create,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['users'] })
  }
})
</script>
```

## Guidelines

**Server State vs Client State:**
- Server state: TanStack Query
- Client state: Pinia (UI, preferences, temporary data)
- Don't duplicate server state in Pinia

**Query Keys:**
- Use array format
- Include all variables that affect query
- Hierarchical structure

**Mutations:**
- Invalidate related queries on success
- Use optimistic updates for better UX
- Handle errors gracefully

**Caching:**
- Configure appropriate staleTime
- Use placeholderData for pagination
- Invalidate when data changes

## Benefits

Automatic caching. No manual cache management.

Background refetching. Keep data fresh automatically.

Optimistic updates. Better user experience.

Deduplication. Multiple components share same query.

## Related

- [pinia-state-management.md](./pinia-state-management.md) - Client state
- [composition-api-basics.md](../01-vue/composition-api-basics.md) - Vue composables
- [api-integration.md](./api-integration.md) - API patterns
