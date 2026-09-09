# err-retry-config: Configure Retry Logic Appropriately

## Priority: HIGH

## Explanation

TanStack Query retries failed queries 3 times by default with exponential backoff (1s, 2s, 4s, capped at 30s). This is too aggressive for some APIs and wastes time for errors that won't resolve by retrying (like 404s). Configure retry logic based on error type and context.

## Bad Example

```tsx
// Relies on default retry: 3 for everything
const { data } = useQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
  // Default: retries 3 times even for 404 Not Found
  // User waits ~7 seconds before seeing the error
})

// Infinite retries — never shows error to user
const { data } = useQuery({
  queryKey: ['data'],
  queryFn: fetchData,
  retry: true, // Retries forever
})
```

## Good Example

```tsx
// Skip retries for client errors (4xx), retry server errors (5xx)
const { data } = useQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
  retry: (failureCount, error) => {
    if (error.status === 404) return false  // Not found — don't retry
    if (error.status === 403) return false  // Forbidden — don't retry
    if (error.status >= 400 && error.status < 500) return false
    return failureCount < 2  // Retry server errors up to 2 times
  },
})
```

## Good Example: Global Retry Defaults

```tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error) => {
        // Never retry client errors
        if (error.status >= 400 && error.status < 500) return false
        return failureCount < 2
      },
      retryDelay: (attemptIndex) =>
        Math.min(1000 * 2 ** attemptIndex, 30000), // Exponential backoff
    },
    mutations: {
      retry: 0, // Never retry mutations by default
    },
  },
})
```

## Retry Option Reference

| Value | Behavior |
|-------|----------|
| `false` or `0` | No retries |
| `true` | Infinite retries (use with caution) |
| `3` (number) | Retry up to 3 times |
| `(failureCount, error) => boolean` | Custom logic per error |

## Context

- Default retry is `3` on client, `0` on server (SSR)
- Default `retryDelay` uses exponential backoff: 1s → 2s → 4s → 8s, capped at 30s
- During retries, the error appears in `failureReason`; only after final failure does it move to `error`
- Mutations default to `retry: 0` — they don't retry unless configured
- For offline-first apps, consider `networkMode: 'offlineFirst'` instead of high retry counts
- `retryOnMount: false` prevents retrying failed queries when a new component mounts
