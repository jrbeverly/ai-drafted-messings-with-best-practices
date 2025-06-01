# Vue Component Structure

Best practices for structuring Vue 3 Single File Components. Consistent structure, naming, and design patterns for maintainable frontends.

`keywords: vue, sfc, component, structure, props, emits, slots, composition-api, typescript`

## Principle

Vue Single File Components (SFCs) colocate template, logic, and styles in one file. Consistent internal structure, clear naming conventions, and deliberate component design reduce cognitive load and make components predictable across the codebase.

## Single File Component Order

Every SFC follows the same block order: script, template, style. The script block comes first because it defines the component's contract (props, emits, state) that the template consumes.

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Props {
  title: string
  count?: number
}

const props = withDefaults(defineProps<Props>(), {
  count: 0,
})

const doubled = computed(() => props.count * 2)
</script>

<template>
  <div class="summary-card">
    <h2>{{ title }}</h2>
    <p>Count: {{ count }}, Doubled: {{ doubled }}</p>
  </div>
</template>

<style scoped>
.summary-card {
  padding: 1rem;
  border: 1px solid #e2e8f0;
  border-radius: 0.5rem;
}
</style>
```

Why this order:
- Script first: Defines the API surface (props, emits) before usage in template
- Template second: Consumes everything declared in script
- Style last: Presentational concern, least critical to understanding logic

## Script Setup Ordering Convention

Within `<script setup lang="ts">`, declarations follow a consistent top-to-bottom order. This mirrors the flow from external dependencies to internal state to side effects.

```vue
<script setup lang="ts">
// 1. Imports (external libraries, then internal modules)
import { ref, computed, watch, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import type { User } from '@/types'
import { useAuth } from '@/composables/useAuth'
import { formatDate } from '@/utils/formatters'

// 2. Props
interface Props {
  userId: string
  showActions?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  showActions: true,
})

// 3. Emits
const emit = defineEmits<{
  (e: 'select', user: User): void
  (e: 'delete', userId: string): void
}>()

// 4. Composables and injections
const router = useRouter()
const { currentUser } = useAuth()

// 5. Reactive state (refs and reactive)
const isEditing = ref(false)
const formData = ref({ name: '', email: '' })

// 6. Computed properties
const isOwnProfile = computed(() => currentUser.value?.id === props.userId)
const canEdit = computed(() => isOwnProfile.value && props.showActions)

// 7. Watchers
watch(
  () => props.userId,
  (newId) => {
    loadUser(newId)
  },
  { immediate: true }
)

// 8. Methods (functions)
function handleSelect(user: User) {
  emit('select', user)
}

function handleDelete() {
  emit('delete', props.userId)
}

async function loadUser(id: string) {
  // fetch user data
}

// 9. Lifecycle hooks
onMounted(() => {
  console.log('UserProfile mounted')
})
</script>
```

Why this order matters:
- Imports at the top: Standard convention, dependencies visible immediately
- Props and emits next: These define the component's public API
- Composables after API: External logic the component depends on
- State, computed, watchers: Internal reactive logic, in order of derivation
- Methods: Business logic that operates on state
- Lifecycle hooks last: Side effects that run at specific moments

## Component Naming Conventions

Components use PascalCase, multi-word names. Single-word names risk colliding with current or future HTML elements.

```vue
<!-- File: UserProfileCard.vue -->
<script setup lang="ts">
// Component name derived from filename
</script>

<!-- Usage in parent template -->
<template>
  <UserProfileCard :user="selectedUser" />
  <BookListTable :books="filteredBooks" />
  <AppNavigationSidebar />
</template>
```

Naming rules:
- PascalCase for all component filenames: `UserProfile.vue`, not `user-profile.vue`
- Multi-word names required: `UserProfile.vue`, not `Profile.vue`
- Prefix with domain or scope: `BookListItem.vue`, `LoanStatusBadge.vue`
- Base components use a consistent prefix: `BaseButton.vue`, `BaseInput.vue`, `BaseModal.vue`
- Single-instance components use `The` prefix: `TheNavbar.vue`, `TheSidebar.vue`, `TheFooter.vue`

```
components/
├── base/
│   ├── BaseButton.vue
│   ├── BaseInput.vue
│   └── BaseModal.vue
├── book/
│   ├── BookListItem.vue
│   ├── BookDetailCard.vue
│   └── BookSearchFilter.vue
├── user/
│   ├── UserProfileCard.vue
│   └── UserAvatarBadge.vue
├── TheNavbar.vue
└── TheSidebar.vue
```

## Props Design

Props are the component's input contract. Keep them minimal, typed, and provide sensible defaults for optional props.

```vue
<script setup lang="ts">
// Minimal props: only what the component needs
interface Props {
  // Required: no default possible
  bookId: string
  title: string

  // Optional: provide defaults
  showAuthor?: boolean
  maxDescriptionLength?: number
  variant?: 'compact' | 'detailed'
}

const props = withDefaults(defineProps<Props>(), {
  showAuthor: true,
  maxDescriptionLength: 200,
  variant: 'detailed',
})
</script>
```

Props design rules:

```vue
<!-- DO: Flat, primitive props where possible -->
<script setup lang="ts">
interface Props {
  userName: string
  userEmail: string
  isActive: boolean
}

defineProps<Props>()
</script>

<!-- DON'T: Passing entire objects when only a few fields are needed -->
<script setup lang="ts">
interface Props {
  user: User  // Component only uses name and email
}

defineProps<Props>()
</script>
```

When to pass objects vs individual props:
- Pass objects when the component needs 4+ fields from the same entity
- Pass individual props when the component only uses 1-3 fields
- Pass objects when the component is a "detail view" for that entity

```vue
<!-- Object prop is justified: component displays many user fields -->
<script setup lang="ts">
interface Props {
  user: User  // Uses name, email, avatar, role, lastLogin, createdAt
}

defineProps<Props>()
</script>

<template>
  <div class="user-detail">
    <img :src="user.avatar" :alt="user.name" />
    <h2>{{ user.name }}</h2>
    <p>{{ user.email }}</p>
    <span>{{ user.role }}</span>
    <time>{{ user.lastLogin }}</time>
  </div>
</template>
```

## Events and Emits Design

Components communicate upward through emits. Props flow down, events flow up. This creates a unidirectional data flow that is easy to trace and debug.

```vue
<!-- Child: emits events up to parent -->
<script setup lang="ts">
interface Props {
  item: TodoItem
}

const props = defineProps<Props>()

const emit = defineEmits<{
  (e: 'toggle', id: string): void
  (e: 'delete', id: string): void
  (e: 'update', id: string, changes: Partial<TodoItem>): void
}>()

function handleToggle() {
  emit('toggle', props.item.id)
}

function handleTitleChange(newTitle: string) {
  emit('update', props.item.id, { title: newTitle })
}
</script>

<template>
  <div class="todo-item">
    <input
      type="checkbox"
      :checked="item.completed"
      @change="handleToggle"
    />
    <input
      :value="item.title"
      @input="handleTitleChange(($event.target as HTMLInputElement).value)"
    />
    <button @click="emit('delete', item.id)">Remove</button>
  </div>
</template>
```

```vue
<!-- Parent: handles events, owns the state -->
<script setup lang="ts">
import { ref } from 'vue'
import type { TodoItem } from '@/types'

const todos = ref<TodoItem[]>([])

function toggleTodo(id: string) {
  const todo = todos.value.find(t => t.id === id)
  if (todo) {
    todo.completed = !todo.completed
  }
}

function deleteTodo(id: string) {
  todos.value = todos.value.filter(t => t.id !== id)
}

function updateTodo(id: string, changes: Partial<TodoItem>) {
  const todo = todos.value.find(t => t.id === id)
  if (todo) {
    Object.assign(todo, changes)
  }
}
</script>

<template>
  <TodoListItem
    v-for="todo in todos"
    :key="todo.id"
    :item="todo"
    @toggle="toggleTodo"
    @delete="deleteTodo"
    @update="updateTodo"
  />
</template>
```

Emit naming conventions:
- Use verb or verb-noun: `select`, `delete`, `update`, `toggleSidebar`
- Use present tense for commands: `submit`, not `submitted`
- Use past tense only for notification events: `loaded`, `transitionEnd`
- Prefix with domain when ambiguous: `userSelect` vs `bookSelect`

## Slots for Content Distribution

Slots allow parent components to inject content into child component layouts. Use default slots for simple content, named slots for multi-region layouts, and scoped slots when the child needs to pass data back.

### Default Slot

```vue
<!-- BaseCard.vue -->
<script setup lang="ts">
interface Props {
  title: string
}

defineProps<Props>()
</script>

<template>
  <div class="card">
    <h3 class="card-title">{{ title }}</h3>
    <div class="card-body">
      <slot />
    </div>
  </div>
</template>

<!-- Usage -->
<template>
  <BaseCard title="User Details">
    <p>This content is projected into the card body.</p>
    <UserAvatarBadge :user="currentUser" />
  </BaseCard>
</template>
```

### Named Slots

```vue
<!-- PageLayout.vue -->
<template>
  <div class="page-layout">
    <header class="page-header">
      <slot name="header" />
    </header>
    <aside class="page-sidebar">
      <slot name="sidebar" />
    </aside>
    <main class="page-content">
      <slot />
    </main>
    <footer class="page-footer">
      <slot name="footer">
        <!-- Default footer content if no slot provided -->
        <p>&copy; 2026 My App</p>
      </slot>
    </footer>
  </div>
</template>

<!-- Usage -->
<template>
  <PageLayout>
    <template #header>
      <h1>Dashboard</h1>
      <UserMenu />
    </template>

    <template #sidebar>
      <NavigationLinks />
    </template>

    <!-- Default slot: main content -->
    <DashboardWidgets />

    <!-- Footer slot omitted: uses default content -->
  </PageLayout>
</template>
```

### Scoped Slots

Scoped slots let the child component expose data to the parent's slot content. Useful for renderless or data-provider components.

```vue
<!-- DataList.vue -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'

interface Props {
  fetchFn: () => Promise<unknown[]>
}

const props = defineProps<Props>()

const items = ref<unknown[]>([])
const loading = ref(true)
const error = ref<Error | null>(null)

onMounted(async () => {
  try {
    items.value = await props.fetchFn()
  } catch (e) {
    error.value = e as Error
  } finally {
    loading.value = false
  }
})
</script>

<template>
  <div>
    <slot v-if="loading" name="loading">
      <p>Loading...</p>
    </slot>
    <slot v-else-if="error" name="error" :error="error">
      <p>Error: {{ error.message }}</p>
    </slot>
    <slot v-else :items="items" :count="items.length" />
  </div>
</template>

<!-- Usage -->
<template>
  <DataList :fetch-fn="fetchUsers">
    <template #default="{ items: users, count }">
      <p>Found {{ count }} users</p>
      <UserCard v-for="user in users" :key="user.id" :user="user" />
    </template>
    <template #error="{ error }">
      <ErrorAlert :message="error.message" @retry="refetch" />
    </template>
  </DataList>
</template>
```

## Component Size Guidelines

A component that is too large becomes difficult to understand, test, and maintain. Split when a component exceeds practical thresholds.

When to split a component:
- Template exceeds ~100 lines
- Script setup exceeds ~100 lines of logic (excluding imports and types)
- Component handles more than one distinct responsibility
- A section of template has its own independent state and logic
- You find yourself adding comments like "// User section" or "// Filters section"

```vue
<!-- BEFORE: Monolithic component doing too much -->
<script setup lang="ts">
import { ref, computed } from 'vue'

// User search state
const searchQuery = ref('')
const sortField = ref('name')
const sortOrder = ref<'asc' | 'desc'>('asc')

// User list state
const users = ref<User[]>([])
const selectedUsers = ref<Set<string>>(new Set())
const isLoading = ref(false)

// Pagination state
const currentPage = ref(1)
const pageSize = ref(20)
const totalCount = ref(0)

// Dialog state
const showCreateDialog = ref(false)
const showDeleteDialog = ref(false)

// ... 30+ more lines of methods for each concern
</script>

<template>
  <!-- 150+ line template with search, table, pagination, dialogs -->
</template>
```

```vue
<!-- AFTER: Split into focused components -->

<!-- UserManagementPage.vue (container) -->
<script setup lang="ts">
import { ref } from 'vue'
import type { UserFilters } from '@/types'

const filters = ref<UserFilters>({ query: '', sort: 'name', order: 'asc' })
const selectedUserIds = ref<Set<string>>(new Set())
</script>

<template>
  <div class="user-management">
    <UserSearchBar v-model="filters" />
    <UserDataTable
      :filters="filters"
      v-model:selected="selectedUserIds"
    />
    <UserBulkActions
      :selected-ids="selectedUserIds"
      @action-complete="selectedUserIds.clear()"
    />
  </div>
</template>

<!-- UserSearchBar.vue (presentational) -->
<script setup lang="ts">
import type { UserFilters } from '@/types'

const filters = defineModel<UserFilters>({ required: true })
</script>

<template>
  <div class="search-bar">
    <input v-model="filters.query" placeholder="Search users..." />
    <select v-model="filters.sort">
      <option value="name">Name</option>
      <option value="email">Email</option>
    </select>
  </div>
</template>
```

## Presentational vs Container Components

Separate components into two categories to clarify responsibility.

**Presentational components** receive data via props, emit events, and contain no business logic or data fetching. They are reusable and easy to test.

```vue
<!-- UserCard.vue (presentational) -->
<script setup lang="ts">
interface Props {
  name: string
  email: string
  avatarUrl: string
  role: 'admin' | 'member' | 'viewer'
}

defineProps<Props>()

const emit = defineEmits<{
  (e: 'edit'): void
  (e: 'delete'): void
}>()
</script>

<template>
  <div class="user-card">
    <img :src="avatarUrl" :alt="name" class="avatar" />
    <div class="info">
      <h3>{{ name }}</h3>
      <p>{{ email }}</p>
      <span class="role-badge">{{ role }}</span>
    </div>
    <div class="actions">
      <button @click="emit('edit')">Edit</button>
      <button @click="emit('delete')">Delete</button>
    </div>
  </div>
</template>
```

**Container components** (also called smart components or pages) fetch data, manage state, and orchestrate presentational components. They contain application-specific logic.

```vue
<!-- UserListPage.vue (container) -->
<script setup lang="ts">
import { useUsers } from '@/composables/useUsers'
import { useDeleteUser } from '@/composables/useDeleteUser'
import { useRouter } from 'vue-router'

const router = useRouter()
const { data: users, isLoading, error } = useUsers()
const { mutateAsync: deleteUser } = useDeleteUser()

function handleEdit(userId: string) {
  router.push({ name: 'user-edit', params: { id: userId } })
}

async function handleDelete(userId: string) {
  if (confirm('Delete this user?')) {
    await deleteUser(userId)
  }
}
</script>

<template>
  <div class="user-list-page">
    <h1>Users</h1>
    <LoadingSpinner v-if="isLoading" />
    <ErrorAlert v-else-if="error" :message="error.message" />
    <div v-else class="user-grid">
      <UserCard
        v-for="user in users"
        :key="user.id"
        :name="user.name"
        :email="user.email"
        :avatar-url="user.avatarUrl"
        :role="user.role"
        @edit="handleEdit(user.id)"
        @delete="handleDelete(user.id)"
      />
    </div>
  </div>
</template>
```

Where each type lives:

```
src/
├── components/         # Presentational (reusable)
│   ├── UserCard.vue
│   ├── BookListItem.vue
│   └── base/
│       ├── BaseButton.vue
│       └── BaseModal.vue
├── views/              # Container (page-level)
│   ├── UserListPage.vue
│   ├── UserEditPage.vue
│   └── BookDetailPage.vue
```

## Provide/Inject Pattern

Use `provide`/`inject` for deeply nested component trees where prop drilling becomes unwieldy. Common for themes, authentication context, and feature flags.

```vue
<!-- App.vue or a high-level layout component -->
<script setup lang="ts">
import { provide, ref } from 'vue'
import type { InjectionKey, Ref } from 'vue'

// Define typed injection key in a shared file
// keys/theme.ts
export interface ThemeContext {
  isDark: Ref<boolean>
  toggleTheme: () => void
}
export const ThemeKey: InjectionKey<ThemeContext> = Symbol('theme')

// Provide in ancestor component
const isDark = ref(false)

function toggleTheme() {
  isDark.value = !isDark.value
}

provide(ThemeKey, { isDark, toggleTheme })
</script>
```

```vue
<!-- Any deeply nested descendant component -->
<script setup lang="ts">
import { inject } from 'vue'
import { ThemeKey } from '@/keys/theme'
import type { ThemeContext } from '@/keys/theme'

const theme = inject(ThemeKey)

if (!theme) {
  throw new Error('ThemeContext not provided. Wrap component tree with ThemeProvider.')
}
</script>

<template>
  <div :class="{ 'dark-mode': theme.isDark.value }">
    <button @click="theme.toggleTheme()">
      {{ theme.isDark.value ? 'Light Mode' : 'Dark Mode' }}
    </button>
    <slot />
  </div>
</template>
```

When to use provide/inject:
- Cross-cutting concerns needed by many descendants (theme, locale, auth)
- Avoiding prop drilling through 3+ intermediate components
- Library or framework-level features

When NOT to use provide/inject:
- Communication between parent and direct child (use props/emits)
- Global state that any component can access (use Pinia)
- Data that changes frequently and needs cache management (use TanStack Query)

## defineExpose for Ref Access

By default, `<script setup>` components do not expose anything to parent template refs. Use `defineExpose` to explicitly declare what a parent can access.

```vue
<!-- FormComponent.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const formData = ref({ name: '', email: '' })
const isValid = ref(false)

function validate(): boolean {
  isValid.value = formData.value.name.length > 0
    && formData.value.email.includes('@')
  return isValid.value
}

function reset() {
  formData.value = { name: '', email: '' }
  isValid.value = false
}

// Only expose what the parent needs
defineExpose({
  validate,
  reset,
})
</script>

<template>
  <form>
    <input v-model="formData.name" placeholder="Name" />
    <input v-model="formData.email" placeholder="Email" />
  </form>
</template>
```

```vue
<!-- ParentPage.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import FormComponent from './FormComponent.vue'

const formRef = ref<InstanceType<typeof FormComponent> | null>(null)

async function handleSubmit() {
  if (formRef.value?.validate()) {
    await saveData()
    formRef.value.reset()
  }
}
</script>

<template>
  <FormComponent ref="formRef" />
  <button @click="handleSubmit">Save</button>
</template>
```

Use `defineExpose` sparingly. Prefer props and emits for standard parent-child communication. Reserve `defineExpose` for imperative actions like `validate()`, `reset()`, `focus()`, or `scrollTo()`.

## Multi-Root Components (Fragments)

Vue 3 supports multiple root elements in a template (fragments). No wrapper `<div>` required.

```vue
<!-- TableRow.vue - multi-root is natural for table rows -->
<script setup lang="ts">
interface Props {
  name: string
  email: string
  role: string
}

defineProps<Props>()
</script>

<template>
  <td>{{ name }}</td>
  <td>{{ email }}</td>
  <td>{{ role }}</td>
</template>

<!-- Usage -->
<template>
  <table>
    <thead>
      <tr>
        <th>Name</th>
        <th>Email</th>
        <th>Role</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="user in users" :key="user.id">
        <TableRow :name="user.name" :email="user.email" :role="user.role" />
      </tr>
    </tbody>
  </table>
</template>
```

When multi-root is appropriate:
- Table row components (`<td>` elements within a `<tr>`)
- List item components that render multiple sibling elements
- Layout fragments that intentionally avoid a wrapper

When to prefer a single root:
- When applying CSS classes or attributes to the component root
- When using `v-show` or transitions on the component (requires single root)
- When inheriting fallthrough attributes (`class`, `style`, `id`) from the parent

```vue
<!-- Single root preferred: needs fallthrough attributes -->
<script setup lang="ts">
interface Props {
  variant: 'primary' | 'secondary'
}

defineProps<Props>()
</script>

<template>
  <!-- Parent can add class="mt-4" and it applies to this div -->
  <div :class="`alert alert-${variant}`">
    <slot />
  </div>
</template>
```

## Style Scoping

Vue provides three approaches for scoping styles to a component. Use `scoped` as the default.

### Scoped Styles (Default)

```vue
<style scoped>
/* Styles only apply to this component's elements */
.card {
  padding: 1rem;
  border: 1px solid #e2e8f0;
}

.card-title {
  font-size: 1.25rem;
  font-weight: 600;
}
</style>
```

Scoped styles add a unique data attribute (`data-v-xxxxxx`) to each element and scope all selectors to that attribute. This prevents style leaks between components.

### Deep Selectors

When you need to style elements inside child components from a parent, use `:deep()`.

```vue
<style scoped>
/* Style child component internals from parent */
.user-form :deep(.v-input) {
  margin-bottom: 1rem;
}

.user-form :deep(.v-btn) {
  text-transform: none;
}
</style>
```

Use `:deep()` sparingly. It creates implicit coupling between parent and child component internals. Prefer:
1. Props on the child component for customization
2. CSS custom properties (CSS variables) for theming
3. `:deep()` only for third-party component overrides (e.g., Vuetify)

### CSS Modules

CSS Modules generate unique class names at build time. Use when you need programmatic access to class names in script.

```vue
<script setup lang="ts">
import { computed } from 'vue'

interface Props {
  variant: 'success' | 'warning' | 'error'
}

const props = defineProps<Props>()

// Access class names in script when needed
const badgeClass = computed(() => {
  return {
    [$style.badge]: true,
    [$style[props.variant]]: true,
  }
})
</script>

<template>
  <span :class="badgeClass">
    <slot />
  </span>
</template>

<style module>
.badge {
  padding: 0.25rem 0.75rem;
  border-radius: 9999px;
  font-size: 0.875rem;
}

.success {
  background-color: #dcfce7;
  color: #166534;
}

.warning {
  background-color: #fef9c3;
  color: #854d0e;
}

.error {
  background-color: #fee2e2;
  color: #991b1b;
}
</style>
```

### Style Scoping Decision Guide

| Approach | Use When |
|----------|----------|
| `scoped` | Default for all components |
| `:deep()` | Overriding third-party component styles |
| CSS Modules | Need class names in script logic |
| Unscoped | Global styles only (App.vue, base reset) |

## Best Practices

**DO:**
- Use `<script setup lang="ts">` for all components
- Follow the script setup ordering convention consistently
- Use PascalCase, multi-word component names
- Define typed interfaces for props and emits
- Provide defaults for optional props with `withDefaults`
- Keep components focused on a single responsibility
- Split when template or script exceeds ~100 lines
- Use `scoped` styles by default
- Use slots for flexible content distribution
- Separate presentational and container components

**DON'T:**
- Use the Options API for new components
- Use single-word component names (`Table.vue`, `Button.vue`)
- Pass entire objects as props when only 1-2 fields are needed
- Mutate props directly (emit events instead)
- Use `defineExpose` for standard data flow (prefer props/emits)
- Use `:deep()` to reach into your own components (refactor the child instead)
- Skip TypeScript types on props and emits
- Put data fetching in presentational components
- Use `provide`/`inject` where props/emits would suffice
- Create wrapper divs unnecessarily (use fragments when appropriate)

## Guidelines

**Essential:**
- Script-template-style block order in every SFC
- Typed props with `defineProps<T>()` and typed emits with `defineEmits<T>()`
- PascalCase multi-word component filenames
- Scoped styles by default
- Unidirectional data flow: props down, events up

**Recommended:**
- Script setup ordering convention (imports, props, emits, composables, state, computed, watchers, methods, lifecycle)
- Presentational vs container component separation
- Typed injection keys for provide/inject
- Base component prefix (`BaseButton`, `BaseInput`) for generic reusables
- `The` prefix for single-instance components (`TheNavbar`, `TheSidebar`)

**Advanced:**
- Scoped slots for renderless data-provider components
- CSS Modules when class names need programmatic access in script
- `defineExpose` for imperative child component APIs (validate, reset, focus)
- `defineModel` for two-way binding on custom components (Vue 3.4+)
- Dynamic components with `<component :is="...">` for runtime component selection

## Benefits

Predictability. Every component follows the same internal structure.

Maintainability. Clear separation of concerns makes changes localized.

Type safety. TypeScript interfaces on props and emits catch errors at compile time.

Reusability. Presentational components with slots and props work across contexts.

Testability. Presentational components are pure functions of their props.

Scaleability. Consistent patterns make large codebases navigable.

## Related

- [composition-api-basics.md](./composition-api-basics.md) - Composition API fundamentals
- [vue-composables.md](./vue-composables.md) - Extracting reusable logic into composables
- [vue-error-handling.md](./vue-error-handling.md) - Error boundaries and error handling patterns
- [typescript-vue-integration.md](../02-typescript/typescript-vue-integration.md) - TypeScript in Vue components
