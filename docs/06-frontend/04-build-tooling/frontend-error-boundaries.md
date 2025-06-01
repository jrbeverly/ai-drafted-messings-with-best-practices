# Frontend Error Boundaries

`keywords: error-boundary, Vue, error-handling, fallback-UI, error-recovery, resilience, onErrorCaptured`

## Principle

**Contain runtime errors within boundaries so that a failure in one part of the UI does not crash the entire application.** Error boundaries catch errors in their child component tree, display fallback UI, and optionally report the error for monitoring.

---

## Vue ErrorBoundary Component Pattern

Vue provides the `onErrorCaptured` lifecycle hook which catches errors from descendant components. Wrap this in a reusable ErrorBoundary component.

### Basic ErrorBoundary Component

```vue
<!-- components/ErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue';

interface Props {
  fallbackMessage?: string;
}

const props = withDefaults(defineProps<Props>(), {
  fallbackMessage: 'Something went wrong. Please try again.',
});

const emit = defineEmits<{
  error: [error: Error, info: string];
}>();

const error = ref<Error | null>(null);
const errorInfo = ref<string>('');

onErrorCaptured((err: Error, instance, info: string) => {
  error.value = err;
  errorInfo.value = info;
  emit('error', err, info);

  // Return false to stop the error from propagating further
  return false;
});

function reset() {
  error.value = null;
  errorInfo.value = '';
}
</script>

<template>
  <slot v-if="!error" />
  <slot v-else name="fallback" :error="error" :reset="reset">
    <div role="alert" class="error-boundary-fallback">
      <p>{{ fallbackMessage }}</p>
      <button @click="reset">Try again</button>
    </div>
  </slot>
</template>

<style scoped>
.error-boundary-fallback {
  padding: 1rem;
  border: 1px solid var(--color-error, #dc3545);
  border-radius: 4px;
  background: var(--color-error-bg, #fdf0f0);
  text-align: center;
}
</style>
```

### Usage

```vue
<script setup lang="ts">
import ErrorBoundary from '@/components/ErrorBoundary.vue';
import UserProfile from '@/components/UserProfile.vue';

function handleError(error: Error, info: string) {
  console.error('ErrorBoundary caught:', error, info);
}
</script>

<template>
  <ErrorBoundary @error="handleError">
    <UserProfile :user-id="userId" />

    <template #fallback="{ error, reset }">
      <div class="error-state">
        <p>Failed to load user profile.</p>
        <details>
          <summary>Error details</summary>
          <pre>{{ error.message }}</pre>
        </details>
        <button @click="reset">Retry</button>
      </div>
    </template>
  </ErrorBoundary>
</template>
```

---

## Error Boundaries for Route Segments

Wrap route-level components in error boundaries so that a crash in one page does not break navigation or the application shell.

### Router-Level Error Boundaries

```vue
<!-- layouts/DefaultLayout.vue -->
<script setup lang="ts">
import ErrorBoundary from '@/components/ErrorBoundary.vue';
import AppHeader from '@/components/AppHeader.vue';
import AppSidebar from '@/components/AppSidebar.vue';

function handleRouteError(error: Error) {
  // Report to error tracking service
  reportError(error);
}
</script>

<template>
  <div class="app-layout">
    <!-- Header and sidebar are outside the boundary - always visible -->
    <AppHeader />
    <AppSidebar />

    <main class="app-content">
      <!-- Only the route content is wrapped - navigation stays functional -->
      <ErrorBoundary
        fallback-message="This page encountered an error."
        @error="handleRouteError"
      >
        <RouterView />

        <template #fallback="{ reset }">
          <div class="route-error">
            <h2>Page Error</h2>
            <p>This page could not be displayed.</p>
            <div class="route-error-actions">
              <button @click="reset">Retry</button>
              <RouterLink to="/">Go to Dashboard</RouterLink>
            </div>
          </div>
        </template>
      </ErrorBoundary>
    </main>
  </div>
</template>
```

---

## Granular Error Recovery

Different parts of a page can fail independently. Wrap each section in its own boundary so that a failure in the sidebar does not take down the main content.

```vue
<!-- pages/DashboardPage.vue -->
<script setup lang="ts">
import ErrorBoundary from '@/components/ErrorBoundary.vue';
import ActivityFeed from '@/components/ActivityFeed.vue';
import StatsPanel from '@/components/StatsPanel.vue';
import NotificationsWidget from '@/components/NotificationsWidget.vue';
</script>

<template>
  <div class="dashboard">
    <ErrorBoundary fallback-message="Stats unavailable.">
      <StatsPanel />
    </ErrorBoundary>

    <ErrorBoundary fallback-message="Activity feed unavailable.">
      <ActivityFeed />
    </ErrorBoundary>

    <ErrorBoundary fallback-message="Notifications unavailable.">
      <NotificationsWidget />
    </ErrorBoundary>
  </div>
</template>
```

If `NotificationsWidget` throws, the stats and activity feed continue working. The user sees a contained error message only in the notifications area.

---

## Fallback UI Patterns

Design fallback UIs that match the context and severity of the error.

### Inline Fallback (Non-Critical Section)

```vue
<template #fallback="{ reset }">
  <div class="inline-error">
    <span>Could not load this section.</span>
    <button @click="reset">Retry</button>
  </div>
</template>
```

### Card-Level Fallback (Widget or Panel)

```vue
<template #fallback="{ error, reset }">
  <div class="card card--error">
    <div class="card-body">
      <p class="card-title">Unable to display</p>
      <p class="card-text">{{ error.message }}</p>
      <button class="btn btn-outline" @click="reset">Reload</button>
    </div>
  </div>
</template>
```

### Full-Page Fallback (Route-Level Error)

```vue
<template #fallback="{ error, reset }">
  <div class="full-page-error">
    <h1>Something went wrong</h1>
    <p>We encountered an unexpected error while loading this page.</p>
    <div class="error-actions">
      <button @click="reset">Try Again</button>
      <RouterLink to="/">Return to Dashboard</RouterLink>
    </div>
    <details v-if="isDevelopment">
      <summary>Debug information</summary>
      <pre>{{ error.stack }}</pre>
    </details>
  </div>
</template>
```

---

## Error Reporting Integration

Connect error boundaries to your error reporting service for visibility into production errors.

```typescript
// composables/useErrorReporting.ts
interface ErrorReport {
  error: Error;
  componentInfo: string;
  route: string;
  userId?: string;
  timestamp: string;
}

export function useErrorReporting() {
  function reportError(error: Error, info: string = '') {
    const report: ErrorReport = {
      error,
      componentInfo: info,
      route: window.location.pathname,
      userId: getCurrentUserId(),
      timestamp: new Date().toISOString(),
    };

    // Send to your error reporting endpoint
    fetch('/api/errors', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        message: report.error.message,
        stack: report.error.stack,
        component: report.componentInfo,
        route: report.route,
        userId: report.userId,
        timestamp: report.timestamp,
      }),
    }).catch(() => {
      // Silently fail - do not throw from the error reporter
      console.error('Failed to report error:', report);
    });
  }

  return { reportError };
}
```

### ErrorBoundary with Reporting

```vue
<!-- components/ReportingErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue';
import { useErrorReporting } from '@/composables/useErrorReporting';

const { reportError } = useErrorReporting();
const error = ref<Error | null>(null);

onErrorCaptured((err: Error, instance, info: string) => {
  error.value = err;
  reportError(err, info);
  return false;
});

function reset() {
  error.value = null;
}
</script>

<template>
  <slot v-if="!error" />
  <slot v-else name="fallback" :error="error" :reset="reset">
    <div role="alert">
      <p>An error occurred. Our team has been notified.</p>
      <button @click="reset">Try again</button>
    </div>
  </slot>
</template>
```

---

## Error Boundary Composition

Compose error boundaries at multiple levels for defense in depth.

```
App
 +-- ErrorBoundary (global - catches anything uncaught)
      +-- Layout
           +-- Header (no boundary needed - static content)
           +-- ErrorBoundary (route-level - catches page errors)
           |    +-- RouterView
           |         +-- DashboardPage
           |              +-- ErrorBoundary (widget - catches section errors)
           |              |    +-- StatsPanel
           |              +-- ErrorBoundary (widget)
           |              |    +-- ActivityFeed
           |              +-- ErrorBoundary (widget)
           |                   +-- NotificationsWidget
           +-- Footer (no boundary needed - static content)
```

Errors bubble up through the tree. The innermost boundary that can handle the error catches it. If a widget boundary fails to recover, the route-level boundary catches the next error. The global boundary is the last line of defense.

---

## Global vs. Local Error Handling

### Global Error Handler

Vue's global error handler catches errors that escape all error boundaries:

```typescript
// main.ts
import { createApp } from 'vue';
import App from './App.vue';

const app = createApp(App);

app.config.errorHandler = (err, instance, info) => {
  // Last-resort error handling
  console.error('Unhandled error:', err);
  console.error('Component:', instance);
  console.error('Info:', info);

  // Report to monitoring
  reportCriticalError(err as Error, info);
};

app.mount('#app');
```

### When to Use Each

| Scope | Mechanism | Use Case |
|---|---|---|
| Widget/Section | ErrorBoundary component | Non-critical UI sections that can fail independently |
| Route/Page | ErrorBoundary around RouterView | Page-level errors, keep navigation working |
| Application | `app.config.errorHandler` | Last resort, log critical failures |
| Async operations | try/catch in composables | API calls, async logic outside render |

### Async Errors Need Explicit Handling

Error boundaries only catch errors during rendering and lifecycle hooks. Async errors must be caught explicitly:

```typescript
// composables/useUsers.ts
export function useUsers() {
  const error = ref<Error | null>(null);
  const users = ref<User[]>([]);

  async function fetchUsers() {
    try {
      error.value = null;
      const response = await api.get('/users');
      users.value = response.data;
    } catch (err) {
      error.value = err instanceof Error ? err : new Error(String(err));
    }
  }

  return { users, error, fetchUsers };
}
```

---

## User-Facing Error Messages

Error messages shown to users must be helpful without exposing implementation details.

```typescript
// utils/userFacingErrors.ts

const USER_MESSAGES: Record<string, string> = {
  NETWORK_ERROR: 'Unable to connect to the server. Please check your connection.',
  NOT_FOUND: 'The requested resource was not found.',
  UNAUTHORIZED: 'Your session has expired. Please log in again.',
  FORBIDDEN: 'You do not have permission to access this resource.',
  VALIDATION_ERROR: 'Please check your input and try again.',
  SERVER_ERROR: 'An unexpected error occurred. Please try again later.',
};

export function getUserMessage(error: Error): string {
  // Map known error types to user-friendly messages
  if (error.message.includes('NetworkError') || error.message.includes('fetch')) {
    return USER_MESSAGES.NETWORK_ERROR;
  }

  if ('status' in error) {
    const status = (error as any).status;
    if (status === 401) return USER_MESSAGES.UNAUTHORIZED;
    if (status === 403) return USER_MESSAGES.FORBIDDEN;
    if (status === 404) return USER_MESSAGES.NOT_FOUND;
    if (status === 422) return USER_MESSAGES.VALIDATION_ERROR;
  }

  return USER_MESSAGES.SERVER_ERROR;
}
```

```vue
<template #fallback="{ error, reset }">
  <div role="alert" class="error-state">
    <!-- User sees a friendly message -->
    <p>{{ getUserMessage(error) }}</p>
    <button @click="reset">Try again</button>

    <!-- Developers see the actual error in dev mode -->
    <pre v-if="isDev">{{ error.stack }}</pre>
  </div>
</template>
```

---

## Best Practices

### DO

- Wrap route-level content in error boundaries to keep navigation working
- Wrap independent UI sections in separate boundaries for granular recovery
- Provide a reset mechanism so users can retry without refreshing the page
- Report caught errors to a monitoring service
- Show user-friendly error messages that suggest an action
- Use `role="alert"` on fallback UI for screen reader accessibility

### DON'T

- Let a single component error crash the entire application
- Show raw error messages, stack traces, or technical details to end users
- Silently swallow errors without logging or reporting them
- Wrap every single component in an error boundary (wrap logical sections)
- Rely on error boundaries to catch async errors (use try/catch for those)
- Forget to handle the case where the error boundary itself might have issues

---

## Guidelines

### Essential

- Global error handler configured in `app.config.errorHandler`
- Route-level error boundary wrapping `<RouterView>`
- Fallback UI for all error boundaries with retry capability

### Recommended

- Section-level error boundaries for independent dashboard widgets and panels
- Error reporting integration sending caught errors to monitoring service
- User-facing error messages mapped from error types, no raw technical details

### Advanced

- Error boundary composition at multiple levels (global, route, section)
- Automatic retry logic with exponential backoff for transient errors
- Error boundary analytics tracking which boundaries trigger most often

---

## Benefits

- **Application resilience** - one broken widget does not crash the whole page
- **Better user experience** - clear error messages with retry actions
- **Operational visibility** - errors reported and tracked in monitoring
- **Graceful degradation** - unaffected sections continue working normally
- **Faster debugging** - error context (component, route, user) captured at boundary

---

## Related

- [vue-error-handling.md](../vue-error-handling.md) - General Vue error handling patterns
- [vue-component-structure.md](../vue-component-structure.md) - Component organization and composition
