# pf-stale-time-config: Set staleTime When Prefetching

## Priority: MEDIUM

## Explanation

When prefetching data, you must set a `staleTime` above 0 on the corresponding `useQuery` — otherwise the default `staleTime: 0` makes prefetched data immediately stale, triggering a refetch as soon as the component mounts. This defeats the purpose of prefetching.

## Bad Example

```tsx
// Prefetch in route loader
loader: async ({ context: { queryClient } }) => {
  await queryClient.prefetchQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  })
}

// Component — no staleTime set
function Posts() {
  const { data } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
    // staleTime defaults to 0
    // Data is immediately stale → refetches on mount
    // The prefetch was wasted!
  })
}
```

## Good Example: staleTime on useQuery

```tsx
const postsQueryOptions = queryOptions({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  staleTime: 60 * 1000, // 1 minute — prevents refetch after prefetch
})

// Loader uses the same options
loader: async ({ context: { queryClient } }) => {
  await queryClient.prefetchQuery(postsQueryOptions)
}

// Component uses the same options — staleTime prevents double-fetch
function Posts() {
  const { data } = useQuery(postsQueryOptions)
}
```

## Good Example: Global Default for SSR/Prefetch Apps

```tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000, // Prevent refetch within 1 minute of prefetch
    },
  },
})
```

## The Problem Visualized

```
Without staleTime:
  Server/Loader: |--prefetch--| ✓ cached
  Client mount:  |--refetch---| ← wasted request (data was "stale" at 0ms)

With staleTime: 60s:
  Server/Loader: |--prefetch--| ✓ cached (fresh for 60s)
  Client mount:  ✓ uses cache  ← no refetch needed
```

## Context

- This is the most common prefetching mistake — always pair prefetch with appropriate `staleTime`
- Use `queryOptions` to share the same config between prefetch and `useQuery`
- `ensureQueryData` returns cached data regardless of staleness (with `revalidateIfStale: false`), but `useQuery` still refetches if stale
- For SSR, set a global default `staleTime` above 0 to avoid double-fetching all queries
- The `staleTime` passed to `prefetchQuery` only controls whether the prefetch itself runs — it does not affect the `useQuery` subscriber
