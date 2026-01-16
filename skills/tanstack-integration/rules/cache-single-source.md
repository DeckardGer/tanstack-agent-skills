# cache-single-source: Let TanStack Query Manage Caching

## Priority: MEDIUM

## Explanation

When using TanStack Router with TanStack Query, let Query be the single source of truth for caching. Disable or minimize Router's built-in cache to avoid confusion about which cache is authoritative.

## Bad Example

```tsx
// Both Router and Query caching enabled - confusing
const router = createRouter({
  routeTree,
  context: { queryClient },
  // Default router caching enabled
  // defaultPreloadStaleTime: 30000 (default)
})

export const Route = createFileRoute('/posts')({
  loader: async () => {
    // Fetches directly - cached by Router
    const posts = await fetchPosts()
    return { posts }
  },
  component: PostsPage,
})

function PostsPage() {
  // Also uses Query cache - which is authoritative?
  const { data } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  })

  // Now there are TWO caches with potentially different data
}
```

## Good Example

```tsx
// router.tsx - Disable router cache when using Query
const router = createRouter({
  routeTree,
  context: { queryClient },

  // Let Query manage caching
  defaultPreloadStaleTime: 0,  // Router doesn't cache

  // SSR integration
  dehydrate: () => ({
    queryClientState: dehydrate(queryClient),
  }),
  hydrate: (dehydrated) => {
    hydrate(queryClient, dehydrated.queryClientState)
  },
})

// routes/posts.tsx
export const Route = createFileRoute('/posts')({
  loader: async ({ context: { queryClient } }) => {
    // Query is the single cache source
    await queryClient.ensureQueryData(postQueries.all())
    // No return needed - data lives in Query cache
  },
  component: PostsPage,
})

function PostsPage() {
  // Single source of truth
  const { data: posts } = useSuspenseQuery(postQueries.all())

  return <PostList posts={posts} />
}
```

## Cache Comparison

| Feature | Router Cache | Query Cache |
|---------|-------------|-------------|
| Invalidation | Manual/time-based | Query keys, patterns |
| Background refetch | No | Yes |
| Optimistic updates | No | Yes |
| Mutations | No built-in | Full support |
| DevTools | Limited | Rich debugging |
| Cross-route sharing | Limited | Full |

## Good Example: Coordinated Caching Config

```tsx
// lib/query-client.ts
export function createQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,       // Fresh for 1 minute
        gcTime: 10 * 60 * 1000,     // Cache for 10 minutes
        refetchOnWindowFocus: true, // Refetch when tab focused
        retry: 1,
      },
    },
  })
}

// router.tsx
export function createAppRouter() {
  const queryClient = createQueryClient()

  return createRouter({
    routeTree,
    context: { queryClient },

    // Router defers to Query for all caching decisions
    defaultPreloadStaleTime: 0,

    Wrap: ({ children }) => (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    ),
  })
}
```

## Good Example: Preload Still Works

```tsx
// Preloading still works - it just uses Query's cache
const router = createRouter({
  routeTree,
  context: { queryClient },
  defaultPreload: 'intent',       // Preload on hover
  defaultPreloadStaleTime: 0,     // Query decides if data is stale
})

// When user hovers a Link:
// 1. Router triggers preload
// 2. Loader runs ensureQueryData
// 3. Query checks its cache - fresh? skip fetch. stale? refetch.
// 4. User clicks - data already in Query cache
```

## Mutation Invalidation

```tsx
// Mutations properly invalidate the single cache
const createPost = useMutation({
  mutationFn: submitPost,
  onSuccess: () => {
    // Invalidate Query cache - the single source
    queryClient.invalidateQueries({ queryKey: ['posts'] })

    // Router automatically uses updated cache on next navigation
    navigate({ to: '/posts' })
  },
})
```

## Context

- `defaultPreloadStaleTime: 0` means "always ask Query"
- Query's staleTime/gcTime controls caching behavior
- Preloading still works - just uses Query's cache
- Mutations, optimistic updates, invalidation all work normally
- DevTools show the single authoritative cache state
- SSR hydration uses Query's dehydrate/hydrate
