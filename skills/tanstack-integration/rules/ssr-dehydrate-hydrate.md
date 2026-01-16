# ssr-dehydrate-hydrate: Configure Dehydration/Hydration

## Priority: LOW

## Explanation

For SSR with TanStack Start, configure the router to dehydrate Query's cache on the server and hydrate it on the client. This transfers prefetched data to the client, preventing duplicate requests.

## Bad Example

```tsx
// No SSR configuration - data refetches on client
const router = createRouter({
  routeTree,
  context: { queryClient },
  // Missing dehydrate/hydrate configuration
})

// Server prefetches data
export const Route = createFileRoute('/posts')({
  loader: async ({ context: { queryClient } }) => {
    await queryClient.ensureQueryData(postQueries.all())
  },
})

// Client doesn't receive prefetched data
// useSuspenseQuery refetches on mount
```

## Good Example: Full SSR Configuration

```tsx
// router.tsx
import { createRouter } from '@tanstack/react-router'
import {
  QueryClient,
  QueryClientProvider,
  dehydrate,
  hydrate,
} from '@tanstack/react-query'

export function createAppRouter() {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,
        // Higher gcTime on server to survive serialization
        gcTime: Infinity,
      },
    },
  })

  return createRouter({
    routeTree,
    context: { queryClient },

    // Disable router caching - Query handles it
    defaultPreloadStaleTime: 0,

    // SSR: Serialize Query cache to send to client
    dehydrate: () => ({
      queryClientState: dehydrate(queryClient, {
        shouldDehydrateQuery: (query) => {
          // Only dehydrate successful queries
          return query.state.status === 'success'
        },
      }),
    }),

    // SSR: Restore Query cache on client
    hydrate: (dehydrated) => {
      hydrate(queryClient, dehydrated.queryClientState)
    },

    Wrap: ({ children }) => (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    ),
  })
}
```

## Good Example: TanStack Start Entry Files

```tsx
// app/ssr.tsx
import { createRouter } from './router'
import { getRouterManifest } from '@tanstack/react-start/router-manifest'
import {
  createStartHandler,
  defaultStreamHandler,
} from '@tanstack/react-start/server'

export default createStartHandler({
  createRouter,
  getRouterManifest,
})(defaultStreamHandler)

// app/client.tsx
import { createRouter } from './router'
import { StartClient } from '@tanstack/react-start'
import { hydrateRoot } from 'react-dom/client'

const router = createRouter()

hydrateRoot(
  document,
  <StartClient router={router} />
)
```

## Good Example: Selective Dehydration

```tsx
const router = createRouter({
  routeTree,
  context: { queryClient },

  dehydrate: () => ({
    queryClientState: dehydrate(queryClient, {
      shouldDehydrateQuery: (query) => {
        // Don't dehydrate failed queries
        if (query.state.status !== 'success') return false

        // Don't dehydrate user-specific data in shared cache
        if (query.queryKey[0] === 'user-private') return false

        // Don't dehydrate large payloads
        if (query.state.dataUpdateCount > 0) {
          const dataSize = JSON.stringify(query.state.data).length
          if (dataSize > 100_000) return false  // Skip if > 100KB
        }

        return true
      },
    }),
  }),

  hydrate: (dehydrated) => {
    hydrate(queryClient, dehydrated.queryClientState)
  },
})
```

## Good Example: With React Query DevTools

```tsx
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'

export function createAppRouter() {
  const queryClient = new QueryClient(/* ... */)

  return createRouter({
    routeTree,
    context: { queryClient },

    dehydrate: () => ({
      queryClientState: dehydrate(queryClient),
    }),

    hydrate: (dehydrated) => {
      hydrate(queryClient, dehydrated.queryClientState)
    },

    Wrap: ({ children }) => (
      <QueryClientProvider client={queryClient}>
        {children}
        {process.env.NODE_ENV === 'development' && (
          <ReactQueryDevtools initialIsOpen={false} />
        )}
      </QueryClientProvider>
    ),
  })
}
```

## SSR Data Flow

```
Server:
  1. Request received
  2. createRouter() creates fresh QueryClient
  3. Router matches routes, runs loaders
  4. Loaders call ensureQueryData → data cached in QueryClient
  5. dehydrate() serializes QueryClient state
  6. HTML + serialized state sent to client

Client:
  1. HTML rendered (React hydrates)
  2. createRouter() creates fresh QueryClient
  3. hydrate() restores state from server
  4. useSuspenseQuery finds data in cache - no refetch!
  5. App is interactive with data already loaded
```

## Context

- `dehydrate()` extracts serializable state from QueryClient
- `hydrate()` restores state into a QueryClient
- Only successful queries are dehydrated by default
- Set `staleTime > 0` to prevent immediate client refetch
- Each SSR request needs its own QueryClient instance
- DevTools only show in development builds
