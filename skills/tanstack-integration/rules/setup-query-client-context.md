# setup-query-client-context: Pass QueryClient Through Router Context

## Priority: CRITICAL

## Explanation

Pass the QueryClient instance through TanStack Router's context system rather than using a global. This enables proper SSR with per-request clients, testability, and type-safe access in loaders.

## Bad Example

```tsx
// lib/query-client.ts - Global singleton
export const queryClient = new QueryClient()

// routes/posts.tsx - Importing global
import { queryClient } from '@/lib/query-client'

export const Route = createFileRoute('/posts')({
  loader: async () => {
    // Using global - breaks SSR, harder to test
    return queryClient.fetchQuery(postQueries.list())
  },
})
```

## Good Example

```tsx
// routes/__root.tsx
import { createRootRouteWithContext } from '@tanstack/react-router'
import { QueryClient } from '@tanstack/react-query'

interface RouterContext {
  queryClient: QueryClient
}

export const Route = createRootRouteWithContext<RouterContext>()({
  component: RootComponent,
})

// router.tsx
import { createRouter } from '@tanstack/react-router'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { routeTree } from './routeTree.gen'

export function createAppRouter() {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,
      },
    },
  })

  const router = createRouter({
    routeTree,
    context: {
      queryClient,
    },
    // Wrap entire app with QueryClientProvider
    Wrap: ({ children }) => (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    ),
  })

  return router
}

declare module '@tanstack/react-router' {
  interface Register {
    router: ReturnType<typeof createAppRouter>
  }
}

// routes/posts.tsx - Access from context
export const Route = createFileRoute('/posts')({
  loader: async ({ context: { queryClient } }) => {
    // Type-safe access to queryClient from context
    await queryClient.ensureQueryData(postQueries.list())
  },
})
```

## Good Example: SSR with Per-Request Client

```tsx
// entry-server.tsx
import { createAppRouter } from './router'

export async function render(req: Request) {
  // Create fresh QueryClient for each request
  const router = createAppRouter()

  // Wait for critical data to load
  await router.load()

  const html = renderToString(
    <RouterProvider router={router} />
  )

  return html
}

// entry-client.tsx
import { createAppRouter } from './router'

const router = createAppRouter()

hydrateRoot(
  document.getElementById('app')!,
  <RouterProvider router={router} />
)
```

## Good Example: Testing with Mock QueryClient

```tsx
// tests/posts.test.tsx
import { createRouter, RouterProvider } from '@tanstack/react-router'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { render } from '@testing-library/react'

function renderWithProviders(route: string) {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
    },
  })

  const router = createRouter({
    routeTree,
    context: { queryClient },
    Wrap: ({ children }) => (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    ),
  })

  return {
    ...render(<RouterProvider router={router} />),
    queryClient,
  }
}

test('loads posts', async () => {
  const { queryClient } = renderWithProviders('/posts')

  // Pre-populate cache for testing
  queryClient.setQueryData(['posts'], mockPosts)

  // ... assertions
})
```

## Context

- Router context flows to all loaders and beforeLoad hooks
- Creating QueryClient per request is essential for SSR
- Use `Wrap` option for provider wrapping
- Access queryClient via `context` parameter in loaders
- This pattern enables clean dependency injection for testing
