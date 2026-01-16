# ssr-dehydrate-hydrate: Configure SSR Query Integration

## Priority: CRITICAL

## Explanation

Use `@tanstack/react-router-ssr-query` to automatically handle SSR dehydration/hydration between TanStack Router and TanStack Query. This package automates cache transfer, streaming, and redirect handling.

## Bad Example

```tsx
// Manual dehydration - verbose and error-prone
import { dehydrate, hydrate } from '@tanstack/react-query'

const router = createRouter({
  routeTree,
  context: { queryClient },

  // Manual approach - lots of boilerplate
  dehydrate: () => ({
    queryClientState: dehydrate(queryClient),
  }),

  hydrate: (dehydrated) => {
    hydrate(queryClient, dehydrated.queryClientState)
  },

  Wrap: ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  ),
})
```

## Good Example: Modern SSR Integration

```tsx
// router.tsx
import { QueryClient } from '@tanstack/react-query'
import { createRouter } from '@tanstack/react-router'
import { setupRouterSsrQueryIntegration } from '@tanstack/react-router-ssr-query'
import { routeTree } from './routeTree.gen'

export function getRouter() {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        refetchOnWindowFocus: false,
        staleTime: 1000 * 60 * 2, // 2 minutes
      },
    },
  })

  const router = createRouter({
    routeTree,
    context: { queryClient },
    defaultPreload: 'intent',
    defaultPreloadStaleTime: 0, // Let Query manage cache freshness
    scrollRestoration: true,
    defaultStructuralSharing: true,
  })

  // Automatic SSR dehydration/hydration
  setupRouterSsrQueryIntegration({
    router,
    queryClient,
    handleRedirects: true,  // Intercept redirects from queries/mutations
    wrapQueryClient: true,  // Auto-wrap with QueryClientProvider
  })

  return router
}
```

## Good Example: With Error and NotFound Components

```tsx
import { DefaultCatchBoundary } from '@/components/DefaultCatchBoundary'
import { DefaultNotFound } from '@/components/DefaultNotFound'

export function getRouter() {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        refetchOnWindowFocus: false,
        staleTime: 1000 * 60 * 2,
      },
    },
  })

  const router = createRouter({
    routeTree,
    context: { queryClient, user: null },
    defaultPreload: 'intent',
    defaultPreloadStaleTime: 0,
    defaultErrorComponent: DefaultCatchBoundary,
    defaultNotFoundComponent: DefaultNotFound,
    scrollRestoration: true,
    defaultStructuralSharing: true,
  })

  setupRouterSsrQueryIntegration({
    router,
    queryClient,
    handleRedirects: true,
    wrapQueryClient: true,
  })

  return router
}
```

## Good Example: Custom QueryClientProvider

```tsx
// If you need custom provider setup (e.g., for DevTools)
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'

export function getRouter() {
  const queryClient = new QueryClient()

  const router = createRouter({
    routeTree,
    context: { queryClient },
    defaultPreload: 'intent',
    defaultPreloadStaleTime: 0,
    scrollRestoration: true,
  })

  setupRouterSsrQueryIntegration({
    router,
    queryClient,
    handleRedirects: true,
    wrapQueryClient: false, // We'll provide our own
  })

  // Custom wrapper with DevTools
  router.options.Wrap = ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
      {process.env.NODE_ENV === 'development' && (
        <ReactQueryDevtools initialIsOpen={false} />
      )}
    </QueryClientProvider>
  )

  return router
}
```

## Good Example: Vite Configuration

```ts
// vite.config.ts
import { tanstackStart } from "@tanstack/start/plugin/vite"
import { defineConfig } from "vite"
import react from "@vitejs/plugin-react"

export default defineConfig({
  plugins: [
    tanstackStart(),  // Handles SSR entry points automatically
    react(),
  ],
})
```

TanStack Start handles client hydration and SSR automatically via the Vite plugin. No separate `client.tsx` or `ssr.tsx` files are needed.

## setupRouterSsrQueryIntegration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `router` | Router | Required | Your router instance |
| `queryClient` | QueryClient | Required | Your QueryClient instance |
| `handleRedirects` | boolean | `true` | Intercept and handle redirects from queries/mutations |
| `wrapQueryClient` | boolean | `true` | Wrap router with QueryClientProvider automatically |

## SSR Data Flow

```
Server:
  1. Request received
  2. getRouter() creates fresh QueryClient + Router
  3. setupRouterSsrQueryIntegration connects them
  4. Router matches routes, runs loaders
  5. Loaders call ensureQueryData → data cached
  6. Integration auto-dehydrates QueryClient state
  7. HTML + serialized state streamed to client

Client:
  1. HTML rendered (React hydrates)
  2. getRouter() creates fresh QueryClient + Router
  3. Integration auto-hydrates state from server
  4. useSuspenseQuery finds data in cache - no refetch!
  5. App is interactive with data already loaded
```

## Context

- Install: `npm install @tanstack/react-router-ssr-query`
- Creates fresh QueryClient per request (required for SSR)
- Handles streaming of queries that resolve during render
- Set `defaultPreloadStaleTime: 0` to let Query manage freshness
- Each SSR request needs its own router instance via `getRouter()`
- The integration handles all dehydration/hydration automatically
