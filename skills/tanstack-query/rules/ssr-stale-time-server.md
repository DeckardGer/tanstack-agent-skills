# ssr-stale-time-server: Set Higher staleTime on Server

## Priority: HIGH

## Explanation

When using SSR, set a default `staleTime` above 0 for the server-side `QueryClient`. The default `staleTime: 0` means data fetched on the server is immediately considered stale, causing every query to refetch as soon as the client hydrates — wasting the server-side prefetch.

## Bad Example

```tsx
// Server-side prefetch
export async function getServerSideProps() {
  const queryClient = new QueryClient() // staleTime defaults to 0

  await queryClient.prefetchQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  })

  return { props: { dehydratedState: dehydrate(queryClient) } }
}

// Client — data from server is immediately stale
// Every query refetches on mount, doubling network requests
function Posts() {
  const { data } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
    // staleTime: 0 → instant refetch after hydration
  })
}
```

## Good Example

```tsx
// Set staleTime as a default for the SSR QueryClient
export async function getServerSideProps() {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000, // Data stays fresh for 1 minute
      },
    },
  })

  await queryClient.prefetchQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  })

  return { props: { dehydratedState: dehydrate(queryClient) } }
}

// Client-side QueryClient should match
const [queryClient] = React.useState(
  () =>
    new QueryClient({
      defaultOptions: {
        queries: {
          staleTime: 60 * 1000, // Same as server — no refetch within 1 minute
        },
      },
    })
)
```

## Good Example: Shared Config

```tsx
// Shared QueryClient factory for both server and client
function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,
      },
    },
  })
}

// Server: new client per request
// Client: singleton
```

## Context

- Without this, every SSR-prefetched query double-fetches on hydration
- Staleness is measured from `dataUpdatedAt` (when fetched on server) — ensure server clock is accurate
- Set `staleTime` on both server and client `QueryClient` for consistent behavior
- Use `queryOptions` to centralize per-query staleTime overrides
- `staleTime: Infinity` makes data permanently fresh — only manual invalidation triggers refetch
- This applies to all SSR approaches: Pages Router, App Router, Remix, TanStack Start
