# ssr-client-per-request: Create QueryClient Per Request

## Priority: CRITICAL

## Explanation

On the server, create a new `QueryClient` for every request. A shared `QueryClient` at module scope leaks cached data between requests and users — this is a security vulnerability and a performance problem.

## Bad Example

```tsx
// NEVER do this — shared across all requests/users
const queryClient = new QueryClient()

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Page />
    </QueryClientProvider>
  )
}

// User A's private data is cached and served to User B
```

## Good Example: React State (Pages Router / Remix)

```tsx
export default function App({ Component, pageProps }: AppProps) {
  // useState ensures one client per component instance (per request on server)
  const [queryClient] = React.useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000, // Prevent immediate refetch after hydration
          },
        },
      })
  )

  return (
    <QueryClientProvider client={queryClient}>
      <HydrationBoundary state={pageProps.dehydratedState}>
        <Component {...pageProps} />
      </HydrationBoundary>
    </QueryClientProvider>
  )
}
```

## Good Example: Server Components (App Router)

```tsx
// app/providers.tsx
'use client'

import { isServer, QueryClient, QueryClientProvider } from '@tanstack/react-query'

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,
      },
    },
  })
}

let browserQueryClient: QueryClient | undefined = undefined

function getQueryClient() {
  if (isServer) {
    // Server: always create a new client (one per request)
    return makeQueryClient()
  } else {
    // Browser: reuse the same client across renders
    if (!browserQueryClient) browserQueryClient = makeQueryClient()
    return browserQueryClient
  }
}

export default function Providers({ children }: { children: React.ReactNode }) {
  const queryClient = getQueryClient()

  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}
```

## Context

- Module-scope `QueryClient` leaks data between users — this is a security issue
- On the server, `gcTime` defaults to `Infinity` — memory is cleaned up when the request ends
- On the browser, reuse a singleton client to persist cache across navigations
- Avoid `useState` for the client when there is no Suspense boundary between the provider and suspending content — React discards state on initial render if it suspends
- Use `isServer` from `@tanstack/react-query` to detect environment
- For server-side loaders (getServerSideProps, Remix loaders), create a `new QueryClient()` per loader call
