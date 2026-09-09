# ssr-hydration-boundary: Wrap with HydrationBoundary

## Priority: HIGH

## Explanation

Use the `dehydrate` / `HydrationBoundary` pattern to transfer server-prefetched data to the client cache. This is the recommended SSR approach — it properly populates the cache with correct timestamps and works with any data-fetching component in the tree.

## Bad Example

```tsx
// Using initialData prop drilling — fragile and limited
export async function getServerSideProps() {
  const posts = await fetchPosts()
  return { props: { posts } }
}

function PostsPage({ posts }) {
  const { data } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
    initialData: posts,
    // Problem: no dataUpdatedAt — staleness based on page load, not fetch time
    // Problem: requires prop drilling to every component that needs data
  })
}
```

## Good Example: Pages Router / Remix

```tsx
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query'

export async function getServerSideProps() {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { staleTime: 60 * 1000 } },
  })

  // Prefetch multiple queries in parallel
  await Promise.all([
    queryClient.prefetchQuery({ queryKey: ['posts'], queryFn: fetchPosts }),
    queryClient.prefetchQuery({ queryKey: ['tags'], queryFn: fetchTags }),
  ])

  return { props: { dehydratedState: dehydrate(queryClient) } }
}

export default function PostsRoute({ dehydratedState }) {
  return (
    <HydrationBoundary state={dehydratedState}>
      <Posts />
    </HydrationBoundary>
  )
}

function Posts() {
  // Data is already in cache — no loading spinner
  const { data } = useQuery({ queryKey: ['posts'], queryFn: fetchPosts })
  return <PostList posts={data} />
}
```

## Good Example: Server Components (App Router)

```tsx
// app/posts/page.tsx — Server Component
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query'

export default async function PostsPage() {
  const queryClient = new QueryClient()

  await queryClient.prefetchQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  })

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <Posts />
    </HydrationBoundary>
  )
}

// app/posts/posts.tsx — Client Component
'use client'

export default function Posts() {
  const { data } = useQuery({ queryKey: ['posts'], queryFn: fetchPosts })
  return <PostList posts={data} />
}
```

## Good Example: Remove Boilerplate in _app.tsx

```tsx
// Wrap at the app level to avoid repeating HydrationBoundary per page
export default function App({ Component, pageProps }: AppProps) {
  const [queryClient] = React.useState(() => new QueryClient())

  return (
    <QueryClientProvider client={queryClient}>
      <HydrationBoundary state={pageProps.dehydratedState}>
        <Component {...pageProps} />
      </HydrationBoundary>
    </QueryClientProvider>
  )
}
```

## Context

- `dehydrate` only includes successful queries by default — failed queries are excluded
- Multiple `HydrationBoundary` components can be nested (e.g., per Server Component)
- Each can use its own `QueryClient` for prefetching — they all hydrate into the nearest provider
- Treat Server Components as a place to prefetch — don't render query data directly in them
- `dehydrate` serializes the cache — avoid non-JSON-serializable data or use custom serializers
- Avoid `gcTime: 0` on the server — data may be garbage collected before hydration completes (minimum: 2 seconds)
