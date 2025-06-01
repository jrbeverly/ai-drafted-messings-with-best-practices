# Vue Error Handling

Best practices for handling errors in Vue 3 applications. Global handlers, error boundaries, async error patterns, and user-friendly error reporting with TypeScript.

`keywords: vue, error-handling, error-boundary, onErrorCaptured, errorHandler, try-catch, result-type, toast, notification, composable, form-validation, network-error`

## Principle

Errors are inevitable. The goal is to catch them at the right level, show users something actionable, and give developers enough information to diagnose the problem. Vue 3 provides a layered error handling system: global handlers catch everything that escapes, component-level hooks catch errors from descendants, and explicit try/catch handles async operations. Use all three layers together for defense in depth.

## Global Error Handler

The `app.config.errorHandler` catches any unhandled error that occurs during component rendering, lifecycle hooks, event handlers, and watchers. It is the last line of defense before an error reaches the browser console unhandled.

```ts
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'
import { reportError } from '@/services/errorReporting'

const app = createApp(App)

app.config.errorHandler = (err, instance, info) => {
  // err: the actual error object
  // instance: the component instance that triggered the error (or null)
  // info: a string describing where the error was caught (e.g. "render function", "watcher callback")

  console.error(`[Global Error] ${info}:`, err)

  // Send to error reporting service (Sentry, Datadog, etc.)
  reportError({
    error: err instanceof Error ? err : new Error(String(err)),
    componentName: instance?.$options?.__name ?? 'Unknown',
    hook: info,
    timestamp: new Date().toISOString(),
  })
}

// Also catch warnings in development
if (import.meta.env.DEV) {
  app.config.warnHandler = (msg, instance, trace) => {
    console.warn(`[Vue Warning] ${msg}`, trace)
  }
}

app.use(router)
app.mount('#app')
```

The global handler should:
- Always log the error for developer visibility
- Forward errors to a reporting service in production
- Never throw another error (this would cause an infinite loop)
- Never display UI directly (it has no template context)

## onErrorCaptured Lifecycle Hook

The `onErrorCaptured` hook catches errors from any descendant component. It runs in the parent that declares it, giving that parent the opportunity to display fallback UI or suppress the error from propagating further.

```vue
<!-- ParentPage.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const error = ref<Error | null>(null)

onErrorCaptured((err: Error, instance, info) => {
  error.value = err

  // Return false to stop the error from propagating to parent components
  // and the global errorHandler
  return false
})

function clearError() {
  error.value = null
}
</script>

<template>
  <div v-if="error" class="error-container">
    <h2>Something went wrong</h2>
    <p>{{ error.message }}</p>
    <button @click="clearError">Try Again</button>
  </div>
  <div v-else>
    <ChildComponent />
  </div>
</template>
```

Propagation behavior:
- Returning `false` from `onErrorCaptured` stops the error from propagating further up the component tree and from reaching `app.config.errorHandler`.
- Returning nothing (or `true`) allows the error to continue propagating to ancestor `onErrorCaptured` hooks and eventually to the global handler.
- Multiple `onErrorCaptured` hooks in the same component all run. Each can independently decide to stop propagation.

## ErrorBoundary Component Pattern

Encapsulate the `onErrorCaptured` logic into a reusable component that wraps any subtree and provides fallback UI when an error occurs.

```vue
<!-- components/ErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

interface Props {
  fallbackMessage?: string
}

const props = withDefaults(defineProps<Props>(), {
  fallbackMessage: 'An unexpected error occurred.',
})

const emit = defineEmits<{
  (e: 'error', error: Error, info: string): void
}>()

const error = ref<Error | null>(null)
const errorInfo = ref<string>('')

onErrorCaptured((err: Error, instance, info) => {
  error.value = err
  errorInfo.value = info

  emit('error', err, info)

  // Stop propagation: this boundary handles it
  return false
})

function reset() {
  error.value = null
  errorInfo.value = ''
}
</script>

<template>
  <slot v-if="!error" />
  <slot v-else name="fallback" :error="error" :error-info="errorInfo" :reset="reset">
    <div role="alert" class="error-boundary">
      <p>{{ fallbackMessage }}</p>
      <pre v-if="$env?.DEV">{{ error.message }}\n{{ errorInfo }}</pre>
      <button @click="reset">Retry</button>
    </div>
  </slot>
</template>

<style scoped>
.error-boundary {
  padding: 1.5rem;
  border: 1px solid #fca5a5;
  border-radius: 0.5rem;
  background-color: #fef2f2;
  color: #991b1b;
}
</style>
```

Usage with default fallback:

```vue
<script setup lang="ts">
import ErrorBoundary from '@/components/ErrorBoundary.vue'
import UserProfile from '@/components/UserProfile.vue'
</script>

<template>
  <ErrorBoundary fallback-message="Could not load user profile.">
    <UserProfile :user-id="userId" />
  </ErrorBoundary>
</template>
```

Usage with custom fallback via scoped slot:

```vue
<template>
  <ErrorBoundary @error="logToService">
    <template #default>
      <DashboardWidgets />
    </template>
    <template #fallback="{ error, reset }">
      <div class="custom-error">
        <h3>Dashboard failed to load</h3>
        <p>{{ error.message }}</p>
        <button @click="reset">Reload Dashboard</button>
      </div>
    </template>
  </ErrorBoundary>
</template>
```

Nest error boundaries for granular recovery. Each boundary isolates failures to the smallest possible section of the page:

```vue
<template>
  <div class="dashboard">
    <ErrorBoundary fallback-message="Header failed to load.">
      <DashboardHeader />
    </ErrorBoundary>

    <div class="dashboard-grid">
      <ErrorBoundary fallback-message="Chart unavailable.">
        <RevenueChart />
      </ErrorBoundary>

      <ErrorBoundary fallback-message="Activity feed unavailable.">
        <ActivityFeed />
      </ErrorBoundary>

      <ErrorBoundary fallback-message="Stats unavailable.">
        <StatsPanel />
      </ErrorBoundary>
    </div>
  </div>
</template>
```

## Async Error Handling in Composables

Vue's `onErrorCaptured` and `app.config.errorHandler` catch errors thrown synchronously during rendering and lifecycle hooks. They also catch errors from async functions that Vue tracks (such as async `setup()`). However, fire-and-forget async operations inside event handlers or watchers may not be caught. Always use explicit try/catch for async work in composables.

### Basic Async Composable Pattern

```ts
// composables/useAsyncOperation.ts
import { ref } from 'vue'
import type { Ref } from 'vue'

interface AsyncState<T> {
  data: Ref<T | null>
  error: Ref<Error | null>
  loading: Ref<boolean>
  execute: () => Promise<void>
}

export function useAsyncOperation<T>(
  operation: () => Promise<T>
): AsyncState<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<Error | null>(null)
  const loading = ref(false)

  async function execute() {
    loading.value = true
    error.value = null

    try {
      data.value = await operation()
    } catch (e) {
      error.value = e instanceof Error ? e : new Error(String(e))
    } finally {
      loading.value = false
    }
  }

  return { data, error, loading, execute }
}
```

### Composable with Retry Logic

```ts
// composables/useRetry.ts
import { ref } from 'vue'
import type { Ref } from 'vue'

interface UseRetryOptions {
  maxRetries?: number
  delayMs?: number
  backoff?: 'fixed' | 'exponential'
}

interface UseRetryReturn<T> {
  data: Ref<T | null>
  error: Ref<Error | null>
  loading: Ref<boolean>
  retryCount: Ref<number>
  execute: () => Promise<void>
}

export function useRetry<T>(
  operation: () => Promise<T>,
  options: UseRetryOptions = {}
): UseRetryReturn<T> {
  const { maxRetries = 3, delayMs = 1000, backoff = 'exponential' } = options

  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<Error | null>(null)
  const loading = ref(false)
  const retryCount = ref(0)

  function getDelay(attempt: number): number {
    if (backoff === 'exponential') {
      return delayMs * Math.pow(2, attempt)
    }
    return delayMs
  }

  function sleep(ms: number): Promise<void> {
    return new Promise((resolve) => setTimeout(resolve, ms))
  }

  async function execute() {
    loading.value = true
    error.value = null
    retryCount.value = 0

    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      try {
        data.value = await operation()
        loading.value = false
        return
      } catch (e) {
        retryCount.value = attempt
        error.value = e instanceof Error ? e : new Error(String(e))

        if (attempt < maxRetries) {
          await sleep(getDelay(attempt))
        }
      }
    }

    loading.value = false
  }

  return { data, error, loading, retryCount, execute }
}
```

Usage in a component:

```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { useRetry } from '@/composables/useRetry'
import { fetchDashboardData } from '@/api/dashboard'
import type { DashboardData } from '@/types'

const {
  data: dashboard,
  error,
  loading,
  retryCount,
  execute: loadDashboard,
} = useRetry<DashboardData>(
  () => fetchDashboardData(),
  { maxRetries: 3, delayMs: 1000, backoff: 'exponential' }
)

onMounted(() => {
  loadDashboard()
})
</script>

<template>
  <div v-if="loading">
    Loading...
    <span v-if="retryCount > 0">(retry {{ retryCount }})</span>
  </div>
  <div v-else-if="error" class="error-state">
    <p>Failed after {{ retryCount + 1 }} attempts: {{ error.message }}</p>
    <button @click="loadDashboard">Try Again</button>
  </div>
  <div v-else-if="dashboard">
    <DashboardContent :data="dashboard" />
  </div>
</template>
```

## API Error Handling Patterns

### Try/Catch with Typed Errors

Define a structured API error type and use it consistently across all API calls:

```ts
// types/api.ts
export interface ApiError {
  status: number
  code: string
  message: string
  details?: Record<string, string[]>
}

export class ApiRequestError extends Error {
  public readonly status: number
  public readonly code: string
  public readonly details?: Record<string, string[]>

  constructor(apiError: ApiError) {
    super(apiError.message)
    this.name = 'ApiRequestError'
    this.status = apiError.status
    this.code = apiError.code
    this.details = apiError.details
  }

  get isNotFound(): boolean {
    return this.status === 404
  }

  get isUnauthorized(): boolean {
    return this.status === 401
  }

  get isForbidden(): boolean {
    return this.status === 403
  }

  get isValidationError(): boolean {
    return this.status === 422
  }

  get isServerError(): boolean {
    return this.status >= 500
  }
}
```

```ts
// services/apiClient.ts
import { ApiRequestError } from '@/types/api'
import type { ApiError } from '@/types/api'

async function handleResponse<T>(response: Response): Promise<T> {
  if (!response.ok) {
    const body = await response.json().catch(() => ({
      status: response.status,
      code: 'UNKNOWN_ERROR',
      message: response.statusText,
    }))

    throw new ApiRequestError(body as ApiError)
  }

  return response.json()
}

export const apiClient = {
  async get<T>(url: string): Promise<T> {
    const response = await fetch(url, {
      headers: { 'Content-Type': 'application/json' },
    })
    return handleResponse<T>(response)
  },

  async post<T>(url: string, data: unknown): Promise<T> {
    const response = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    })
    return handleResponse<T>(response)
  },
}
```

### Result Type Pattern

For operations where errors are expected and should be handled explicitly rather than thrown, use a discriminated union Result type:

```ts
// types/result.ts
export type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E }

export function ok<T>(value: T): Result<T, never> {
  return { ok: true, value }
}

export function err<E>(error: E): Result<never, E> {
  return { ok: false, error }
}
```

```ts
// services/userService.ts
import { apiClient } from '@/services/apiClient'
import { ok, err } from '@/types/result'
import { ApiRequestError } from '@/types/api'
import type { Result } from '@/types/result'
import type { User } from '@/types'

export async function getUser(id: string): Promise<Result<User, ApiRequestError>> {
  try {
    const user = await apiClient.get<User>(`/api/users/${id}`)
    return ok(user)
  } catch (e) {
    if (e instanceof ApiRequestError) {
      return err(e)
    }
    return err(new ApiRequestError({
      status: 0,
      code: 'NETWORK_ERROR',
      message: 'Unable to reach the server. Check your connection.',
    }))
  }
}
```

Using the Result type in a composable:

```ts
// composables/useUser.ts
import { ref, watch, toValue } from 'vue'
import type { Ref, MaybeRefOrGetter } from 'vue'
import { getUser } from '@/services/userService'
import type { User } from '@/types'

export function useUser(userId: MaybeRefOrGetter<string>) {
  const user = ref<User | null>(null) as Ref<User | null>
  const error = ref<string | null>(null)
  const loading = ref(false)

  async function load() {
    loading.value = true
    error.value = null

    const result = await getUser(toValue(userId))

    if (result.ok) {
      user.value = result.value
    } else {
      // Handle specific error cases
      if (result.error.isNotFound) {
        error.value = 'User not found.'
      } else if (result.error.isUnauthorized) {
        error.value = 'You must be logged in to view this user.'
      } else {
        error.value = result.error.message
      }
    }

    loading.value = false
  }

  watch(() => toValue(userId), load, { immediate: true })

  return { user, error, loading, reload: load }
}
```

## Error State in Components

Components should handle three states explicitly: loading, error, and success. Never leave error states invisible to the user.

```vue
<!-- UserProfilePage.vue -->
<script setup lang="ts">
import { useUser } from '@/composables/useUser'
import { useRoute } from 'vue-router'

const route = useRoute()
const { user, error, loading, reload } = useUser(
  () => route.params.id as string
)
</script>

<template>
  <!-- Loading state -->
  <div v-if="loading" class="loading-state">
    <LoadingSpinner />
    <p>Loading user profile...</p>
  </div>

  <!-- Error state -->
  <div v-else-if="error" class="error-state" role="alert">
    <ErrorIcon />
    <h2>Unable to load profile</h2>
    <p>{{ error }}</p>
    <button @click="reload">Retry</button>
  </div>

  <!-- Success state -->
  <div v-else-if="user" class="user-profile">
    <h1>{{ user.name }}</h1>
    <p>{{ user.email }}</p>
  </div>

  <!-- Empty state (no error, but no data) -->
  <div v-else class="empty-state">
    <p>No user data available.</p>
  </div>
</template>

<style scoped>
.error-state {
  padding: 2rem;
  text-align: center;
  color: #991b1b;
}

.loading-state {
  padding: 2rem;
  text-align: center;
  color: #6b7280;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #6b7280;
}
</style>
```

Inline error display for partial failures (one section fails but the rest of the page works):

```vue
<script setup lang="ts">
import { useUser } from '@/composables/useUser'
import { useUserActivity } from '@/composables/useUserActivity'

const props = defineProps<{ userId: string }>()

const { user, error: userError, loading: userLoading } = useUser(() => props.userId)
const { activity, error: activityError, loading: activityLoading } = useUserActivity(() => props.userId)
</script>

<template>
  <div class="profile-page">
    <!-- Critical section: user info -->
    <section v-if="userLoading">Loading profile...</section>
    <section v-else-if="userError" class="error-state" role="alert">
      <p>{{ userError }}</p>
    </section>
    <section v-else-if="user">
      <h1>{{ user.name }}</h1>
      <p>{{ user.email }}</p>
    </section>

    <!-- Non-critical section: activity feed -->
    <section v-if="activityLoading">Loading activity...</section>
    <section v-else-if="activityError" class="error-inline" role="alert">
      <p>Activity feed unavailable.</p>
      <button @click="() => {}">Retry</button>
    </section>
    <section v-else-if="activity">
      <ActivityFeed :items="activity" />
    </section>
  </div>
</template>
```

## User-Friendly Error Messages

Never show raw error messages, stack traces, or technical details to users. Map errors to human-readable messages.

```ts
// utils/errorMessages.ts
import { ApiRequestError } from '@/types/api'

const ERROR_MESSAGES: Record<string, string> = {
  NETWORK_ERROR: 'Unable to reach the server. Please check your internet connection and try again.',
  UNAUTHORIZED: 'Your session has expired. Please log in again.',
  FORBIDDEN: 'You do not have permission to perform this action.',
  NOT_FOUND: 'The requested resource was not found.',
  RATE_LIMITED: 'Too many requests. Please wait a moment and try again.',
  VALIDATION_ERROR: 'Please check your input and try again.',
  SERVER_ERROR: 'Something went wrong on our end. Please try again later.',
}

export function getUserMessage(error: unknown): string {
  if (error instanceof ApiRequestError) {
    // Check for a specific error code first
    if (error.code in ERROR_MESSAGES) {
      return ERROR_MESSAGES[error.code]
    }

    // Fall back to status-based messages
    if (error.isUnauthorized) return ERROR_MESSAGES.UNAUTHORIZED
    if (error.isForbidden) return ERROR_MESSAGES.FORBIDDEN
    if (error.isNotFound) return ERROR_MESSAGES.NOT_FOUND
    if (error.isServerError) return ERROR_MESSAGES.SERVER_ERROR

    return error.message
  }

  if (error instanceof TypeError && error.message === 'Failed to fetch') {
    return ERROR_MESSAGES.NETWORK_ERROR
  }

  return 'An unexpected error occurred. Please try again.'
}
```

Using the message mapper in a component:

```vue
<script setup lang="ts">
import { computed } from 'vue'
import { getUserMessage } from '@/utils/errorMessages'

const props = defineProps<{
  error: Error | null
  retryable?: boolean
}>()

const emit = defineEmits<{
  (e: 'retry'): void
}>()

const message = computed(() =>
  props.error ? getUserMessage(props.error) : ''
)
</script>

<template>
  <div v-if="error" role="alert" class="error-alert">
    <p>{{ message }}</p>
    <button v-if="retryable" @click="emit('retry')">Try Again</button>
  </div>
</template>
```

## Error Logging and Reporting

Create a centralized error reporting service that captures errors, enriches them with context, and forwards them to an external service.

```ts
// services/errorReporting.ts
interface ErrorReport {
  error: Error
  componentName?: string
  hook?: string
  userId?: string
  route?: string
  timestamp: string
  metadata?: Record<string, unknown>
}

interface ErrorReporter {
  captureException(report: ErrorReport): void
}

class ConsoleErrorReporter implements ErrorReporter {
  captureException(report: ErrorReport): void {
    console.error('[ErrorReport]', {
      message: report.error.message,
      stack: report.error.stack,
      component: report.componentName,
      hook: report.hook,
      route: report.route,
      timestamp: report.timestamp,
    })
  }
}

class RemoteErrorReporter implements ErrorReporter {
  private readonly endpoint: string

  constructor(endpoint: string) {
    this.endpoint = endpoint
  }

  captureException(report: ErrorReport): void {
    // Fire-and-forget: do not await, do not let reporting failures
    // cascade into user-facing errors
    fetch(this.endpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        message: report.error.message,
        stack: report.error.stack,
        component: report.componentName,
        hook: report.hook,
        userId: report.userId,
        route: report.route,
        timestamp: report.timestamp,
        metadata: report.metadata,
      }),
    }).catch(() => {
      // Silently fail: error reporting must never cause user-visible errors
    })
  }
}

// Choose reporter based on environment
const reporter: ErrorReporter = import.meta.env.PROD
  ? new RemoteErrorReporter(import.meta.env.VITE_ERROR_ENDPOINT)
  : new ConsoleErrorReporter()

export function reportError(report: Omit<ErrorReport, 'timestamp'>): void {
  reporter.captureException({
    ...report,
    timestamp: new Date().toISOString(),
  })
}
```

Integrate with the global handler and error boundaries:

```ts
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'
import { reportError } from '@/services/errorReporting'

const app = createApp(App)

app.config.errorHandler = (err, instance, info) => {
  reportError({
    error: err instanceof Error ? err : new Error(String(err)),
    componentName: instance?.$options?.__name ?? 'Unknown',
    hook: info,
    route: router.currentRoute.value.fullPath,
  })
}

// Catch unhandled promise rejections that escape Vue's tracking
window.addEventListener('unhandledrejection', (event) => {
  reportError({
    error: event.reason instanceof Error
      ? event.reason
      : new Error(String(event.reason)),
    hook: 'unhandledrejection',
    route: router.currentRoute.value.fullPath,
  })
})

app.use(router)
app.mount('#app')
```

## Vue Router Error Handling

Vue Router provides the `router.onError` hook for catching navigation failures and errors in async route components or navigation guards.

```ts
// router/index.ts
import { createRouter, createWebHistory, isNavigationFailure, NavigationFailureType } from 'vue-router'
import { reportError } from '@/services/errorReporting'

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
})

// Catch errors during navigation (lazy-load failures, guard errors)
router.onError((error, to, from) => {
  reportError({
    error,
    hook: 'router.onError',
    route: to.fullPath,
    metadata: { from: from.fullPath },
  })

  // If a lazy-loaded chunk fails to load, attempt a full page reload
  if (error.message.includes('Failed to fetch dynamically imported module')) {
    window.location.href = to.fullPath
  }
})

// Handle navigation failures in afterEach
router.afterEach((to, from, failure) => {
  if (failure) {
    if (isNavigationFailure(failure, NavigationFailureType.aborted)) {
      console.warn('Navigation aborted:', failure.message)
    } else if (isNavigationFailure(failure, NavigationFailureType.duplicated)) {
      // Navigating to current route: usually harmless, ignore
    } else {
      console.error('Navigation failure:', failure)
    }
  }
})
```

Per-route error handling with navigation guards:

```ts
// router/guards/authGuard.ts
import type { NavigationGuardWithThis } from 'vue-router'
import { useAuthStore } from '@/stores/auth'

export const authGuard: NavigationGuardWithThis<undefined> = (to, from, next) => {
  const auth = useAuthStore()

  if (to.meta.requiresAuth && !auth.isAuthenticated) {
    next({
      name: 'login',
      query: { redirect: to.fullPath },
    })
    return
  }

  if (to.meta.requiredRole && !auth.hasRole(to.meta.requiredRole as string)) {
    next({ name: 'forbidden' })
    return
  }

  next()
}
```

Handling errors inside route components:

```vue
<!-- views/UserDetailPage.vue -->
<script setup lang="ts">
import { useRoute, useRouter } from 'vue-router'
import { useUser } from '@/composables/useUser'
import { ApiRequestError } from '@/types/api'

const route = useRoute()
const router = useRouter()
const { user, error, loading } = useUser(() => route.params.id as string)

// Redirect on specific errors
function handleError() {
  if (error.value instanceof ApiRequestError) {
    if (error.value.isNotFound) {
      router.replace({ name: 'not-found' })
      return
    }
    if (error.value.isUnauthorized) {
      router.replace({ name: 'login', query: { redirect: route.fullPath } })
      return
    }
  }
}
</script>

<template>
  <LoadingSpinner v-if="loading" />
  <div v-else-if="error">
    <ErrorDisplay :error="error" @retry="handleError" />
  </div>
  <UserDetail v-else-if="user" :user="user" />
</template>
```

## Form Validation Errors

Form validation errors are expected user input errors, not application failures. Handle them separately from system errors with clear, field-level feedback.

```vue
<!-- components/UserForm.vue -->
<script setup lang="ts">
import { ref, reactive } from 'vue'
import { apiClient } from '@/services/apiClient'
import { ApiRequestError } from '@/types/api'

interface FormData {
  name: string
  email: string
  role: 'admin' | 'member' | 'viewer'
}

interface FormErrors {
  name?: string
  email?: string
  role?: string
  form?: string
}

const form = reactive<FormData>({
  name: '',
  email: '',
  role: 'member',
})

const errors = reactive<FormErrors>({})
const submitting = ref(false)

function validateField(field: keyof FormData): boolean {
  delete errors[field]

  switch (field) {
    case 'name':
      if (!form.name.trim()) {
        errors.name = 'Name is required.'
        return false
      }
      if (form.name.length < 2) {
        errors.name = 'Name must be at least 2 characters.'
        return false
      }
      return true

    case 'email':
      if (!form.email.trim()) {
        errors.email = 'Email is required.'
        return false
      }
      if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(form.email)) {
        errors.email = 'Please enter a valid email address.'
        return false
      }
      return true

    case 'role':
      if (!['admin', 'member', 'viewer'].includes(form.role)) {
        errors.role = 'Please select a valid role.'
        return false
      }
      return true

    default:
      return true
  }
}

function validateAll(): boolean {
  const fields: (keyof FormData)[] = ['name', 'email', 'role']
  return fields.map(validateField).every(Boolean)
}

async function handleSubmit() {
  errors.form = undefined

  if (!validateAll()) {
    return
  }

  submitting.value = true

  try {
    await apiClient.post('/api/users', form)
  } catch (e) {
    if (e instanceof ApiRequestError && e.isValidationError && e.details) {
      // Map server-side validation errors to form fields
      for (const [field, messages] of Object.entries(e.details)) {
        if (field in errors) {
          (errors as Record<string, string>)[field] = messages[0]
        }
      }
    } else {
      errors.form = 'Failed to save. Please try again.'
    }
  } finally {
    submitting.value = false
  }
}
</script>

<template>
  <form @submit.prevent="handleSubmit" novalidate>
    <!-- Form-level error -->
    <div v-if="errors.form" role="alert" class="form-error">
      {{ errors.form }}
    </div>

    <div class="field">
      <label for="name">Name</label>
      <input
        id="name"
        v-model="form.name"
        :aria-invalid="!!errors.name"
        :aria-describedby="errors.name ? 'name-error' : undefined"
        @blur="validateField('name')"
      />
      <p v-if="errors.name" id="name-error" class="field-error" role="alert">
        {{ errors.name }}
      </p>
    </div>

    <div class="field">
      <label for="email">Email</label>
      <input
        id="email"
        v-model="form.email"
        type="email"
        :aria-invalid="!!errors.email"
        :aria-describedby="errors.email ? 'email-error' : undefined"
        @blur="validateField('email')"
      />
      <p v-if="errors.email" id="email-error" class="field-error" role="alert">
        {{ errors.email }}
      </p>
    </div>

    <div class="field">
      <label for="role">Role</label>
      <select
        id="role"
        v-model="form.role"
        :aria-invalid="!!errors.role"
        @change="validateField('role')"
      >
        <option value="admin">Admin</option>
        <option value="member">Member</option>
        <option value="viewer">Viewer</option>
      </select>
      <p v-if="errors.role" class="field-error" role="alert">
        {{ errors.role }}
      </p>
    </div>

    <button type="submit" :disabled="submitting">
      {{ submitting ? 'Saving...' : 'Save User' }}
    </button>
  </form>
</template>

<style scoped>
.form-error {
  padding: 0.75rem;
  margin-bottom: 1rem;
  background-color: #fef2f2;
  border: 1px solid #fca5a5;
  border-radius: 0.375rem;
  color: #991b1b;
}

.field-error {
  margin-top: 0.25rem;
  font-size: 0.875rem;
  color: #dc2626;
}
</style>
```

## Network Error Recovery

Handle network connectivity issues gracefully by detecting offline state and providing retry mechanisms.

```ts
// composables/useNetworkAwareRequest.ts
import { ref, watch } from 'vue'
import { useOnlineStatus } from '@/composables/useOnlineStatus'
import type { Ref } from 'vue'

interface UseNetworkAwareRequestReturn<T> {
  data: Ref<T | null>
  error: Ref<string | null>
  loading: Ref<boolean>
  isOffline: Ref<boolean>
  execute: () => Promise<void>
}

export function useNetworkAwareRequest<T>(
  request: () => Promise<T>
): UseNetworkAwareRequestReturn<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<string | null>(null)
  const loading = ref(false)
  const { isOnline } = useOnlineStatus()
  const isOffline = ref(!navigator.onLine)
  let pendingRetry = false

  async function execute() {
    if (!isOnline.value) {
      isOffline.value = true
      error.value = 'You are offline. This request will retry when connectivity is restored.'
      pendingRetry = true
      return
    }

    isOffline.value = false
    loading.value = true
    error.value = null

    try {
      data.value = await request()
      pendingRetry = false
    } catch (e) {
      if (e instanceof TypeError && e.message === 'Failed to fetch') {
        isOffline.value = true
        error.value = 'Network request failed. Please check your connection.'
        pendingRetry = true
      } else {
        error.value = e instanceof Error ? e.message : 'An unexpected error occurred.'
      }
    } finally {
      loading.value = false
    }
  }

  // Auto-retry when coming back online
  watch(isOnline, (online) => {
    if (online && pendingRetry) {
      execute()
    }
  })

  return { data, error, loading, isOffline, execute }
}
```

Usage:

```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { useNetworkAwareRequest } from '@/composables/useNetworkAwareRequest'
import { apiClient } from '@/services/apiClient'
import type { User } from '@/types'

const { data: users, error, loading, isOffline, execute } =
  useNetworkAwareRequest<User[]>(() => apiClient.get('/api/users'))

onMounted(() => {
  execute()
})
</script>

<template>
  <div v-if="isOffline" class="offline-banner" role="alert">
    You are offline. Data will refresh automatically when you reconnect.
  </div>

  <LoadingSpinner v-if="loading" />
  <ErrorDisplay v-else-if="error && !isOffline" :message="error" @retry="execute" />
  <UserList v-else-if="users" :users="users" />
</template>
```

## Toast and Notification System for Errors

Toasts are appropriate for transient errors and success confirmations that do not require user action. Use them for background operations, not for errors that block the user's workflow.

```ts
// composables/useToast.ts
import { ref } from 'vue'
import type { Ref } from 'vue'

export type ToastType = 'success' | 'error' | 'warning' | 'info'

export interface Toast {
  id: string
  type: ToastType
  message: string
  duration: number
}

interface UseToastReturn {
  toasts: Ref<Toast[]>
  addToast: (type: ToastType, message: string, duration?: number) => void
  removeToast: (id: string) => void
  success: (message: string) => void
  error: (message: string) => void
  warning: (message: string) => void
  info: (message: string) => void
}

// Shared state across all components using this composable
const toasts = ref<Toast[]>([])

export function useToast(): UseToastReturn {
  function addToast(type: ToastType, message: string, duration = 5000) {
    const id = crypto.randomUUID()
    toasts.value.push({ id, type, message, duration })

    if (duration > 0) {
      setTimeout(() => removeToast(id), duration)
    }
  }

  function removeToast(id: string) {
    toasts.value = toasts.value.filter((t) => t.id !== id)
  }

  return {
    toasts,
    addToast,
    removeToast,
    success: (msg: string) => addToast('success', msg),
    error: (msg: string) => addToast('error', msg, 8000),
    warning: (msg: string) => addToast('warning', msg),
    info: (msg: string) => addToast('info', msg),
  }
}
```

Toast container component:

```vue
<!-- components/ToastContainer.vue -->
<script setup lang="ts">
import { useToast } from '@/composables/useToast'
import type { Toast } from '@/composables/useToast'

const { toasts, removeToast } = useToast()

function iconForType(type: Toast['type']): string {
  const icons: Record<Toast['type'], string> = {
    success: 'check-circle',
    error: 'x-circle',
    warning: 'alert-triangle',
    info: 'info',
  }
  return icons[type]
}
</script>

<template>
  <Teleport to="body">
    <div class="toast-container" aria-live="polite">
      <TransitionGroup name="toast">
        <div
          v-for="toast in toasts"
          :key="toast.id"
          :class="['toast', `toast-${toast.type}`]"
          role="alert"
        >
          <span class="toast-icon">{{ iconForType(toast.type) }}</span>
          <p class="toast-message">{{ toast.message }}</p>
          <button
            class="toast-close"
            aria-label="Dismiss"
            @click="removeToast(toast.id)"
          >
            &times;
          </button>
        </div>
      </TransitionGroup>
    </div>
  </Teleport>
</template>

<style scoped>
.toast-container {
  position: fixed;
  top: 1rem;
  right: 1rem;
  z-index: 9999;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  max-width: 24rem;
}

.toast {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.75rem 1rem;
  border-radius: 0.5rem;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  font-size: 0.875rem;
}

.toast-success { background-color: #dcfce7; color: #166534; }
.toast-error { background-color: #fef2f2; color: #991b1b; }
.toast-warning { background-color: #fef9c3; color: #854d0e; }
.toast-info { background-color: #eff6ff; color: #1e40af; }

.toast-close {
  margin-left: auto;
  background: none;
  border: none;
  cursor: pointer;
  font-size: 1.25rem;
  line-height: 1;
  color: inherit;
  opacity: 0.7;
}

.toast-close:hover { opacity: 1; }

.toast-enter-active,
.toast-leave-active {
  transition: all 0.3s ease;
}

.toast-enter-from {
  opacity: 0;
  transform: translateX(2rem);
}

.toast-leave-to {
  opacity: 0;
  transform: translateX(2rem);
}
</style>
```

Using toasts for mutation feedback:

```vue
<script setup lang="ts">
import { useToast } from '@/composables/useToast'
import { apiClient } from '@/services/apiClient'
import { getUserMessage } from '@/utils/errorMessages'

const { success, error: showError } = useToast()

async function deleteUser(id: string) {
  try {
    await apiClient.post(`/api/users/${id}/delete`, {})
    success('User deleted successfully.')
  } catch (e) {
    showError(getUserMessage(e))
  }
}

async function saveSettings(data: Record<string, unknown>) {
  try {
    await apiClient.post('/api/settings', data)
    success('Settings saved.')
  } catch (e) {
    showError(getUserMessage(e))
  }
}
</script>

<template>
  <!-- ToastContainer must be mounted once in App.vue or a layout component -->
  <button @click="deleteUser('123')">Delete User</button>
  <button @click="saveSettings({ theme: 'dark' })">Save Settings</button>
</template>
```

When to use toasts vs inline errors:
- **Toasts:** Background operations, success confirmations, transient warnings
- **Inline errors:** Form validation, data loading failures, errors that need user action in context
- **Error boundaries:** Unexpected crashes in component subtrees
- **Full-page errors:** Navigation failures, 404s, unauthorized access

## Best Practices

**DO:**
- Set up `app.config.errorHandler` as the global safety net
- Use `onErrorCaptured` or `ErrorBoundary` for component-level isolation
- Always wrap async operations in try/catch inside composables
- Map technical errors to user-friendly messages before displaying
- Provide retry mechanisms for recoverable errors
- Log errors with context (component name, route, user action)
- Use `role="alert"` and `aria-invalid` for accessibility on error UI
- Validate form fields on blur and on submit
- Handle offline/network errors as a distinct category
- Use toasts for transient feedback, inline errors for actionable failures

**DON'T:**
- Show stack traces or raw error objects to users
- Swallow errors silently with empty catch blocks
- Let errors propagate without logging or reporting them
- Display generic "An error occurred" without actionable guidance
- Throw errors inside `app.config.errorHandler` (causes infinite loops)
- Use toasts for errors that require user action in the current context
- Forget to handle the loading and empty states alongside error states
- Mix form validation errors with system errors in the same display

## Guidelines

**Essential:**
- Global `app.config.errorHandler` registered in every application
- try/catch around every `await` in composables and event handlers
- Loading, error, and success states handled in every data-fetching component
- User-friendly error messages (never raw technical details)

**Recommended:**
- Reusable `ErrorBoundary` component wrapping independent page sections
- Centralized error reporting service with structured error data
- Result type pattern for operations where errors are expected
- Toast notification system for background operation feedback
- Network connectivity detection with automatic retry on reconnect

**Advanced:**
- Per-route error handling with router `onError` and navigation guards
- Typed `ApiRequestError` class with convenience properties (`isNotFound`, `isServerError`)
- Exponential backoff retry composable for transient failures
- Error boundary nesting for granular section-level recovery
- `unhandledrejection` listener for promise errors that escape Vue's tracking

## Benefits

Defense in depth. Multiple error handling layers ensure nothing reaches the user unhandled.

User trust. Clear, actionable error messages maintain confidence in the application.

Debuggability. Structured error reporting with context accelerates root cause analysis.

Resilience. Retry mechanisms and network-aware requests recover from transient failures automatically.

Accessibility. ARIA attributes on error states ensure screen readers communicate problems.

Isolation. Error boundaries prevent one broken widget from taking down the entire page.

## Related

- [vue-component-structure.md](./vue-component-structure.md) - Component organization and presentational vs container patterns
- [vue-composables.md](./vue-composables.md) - Composable patterns including async data composables
- [vue-router-patterns.md](./vue-router-patterns.md) - Router setup, guards, and navigation patterns
