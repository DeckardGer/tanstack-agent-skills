# pf-ensure-query-data: Use ensureQueryData for Conditional Prefetching

## Priority: MEDIUM

## Explanation

`ensureQueryData` checks the cache before fetching — it only makes a network request if the query doesn't exist in the cache. Unlike `prefetchQuery`, it returns the data and throws on error. Use it in route loaders where you need to guarantee data exists.

## Bad Example

```tsx
// Always fetches, even when cache already has fresh data
loader: async ({ params, context: { queryClient } }) => {
  await queryClient.fetchQuery({
    queryKey: ['post', params.postId],
    queryFn: () => fetchPost(params.postId),
  })
}

// prefetchQuery also refetches when staleTime is 0 (the default)
loader: async ({ params, context: { queryClient } }) => {
  await queryClient.prefetchQuery({
    queryKey: ['post', params.postId],
    queryFn: () => fetchPost(params.postId),
  })
}
```

## Good Example

```tsx
loader: async ({ params, context: { queryClient } }) => {
  // Only fetches if cache is empty — returns cached data otherwise
  await queryClient.ensureQueryData({
    queryKey: ['post', params.postId],
    queryFn: () => fetchPost(params.postId),
  })
}
```

## Good Example: With revalidateIfStale

```tsx
loader: async ({ params, context: { queryClient } }) => {
  // Returns stale data immediately, triggers background refetch
  await queryClient.ensureQueryData({
    queryKey: ['post', params.postId],
    queryFn: () => fetchPost(params.postId),
    revalidateIfStale: true,
  })
}
```

## API Comparison

| Method | Returns | On error | Cache hit (fresh) | Cache hit (stale) | Cache miss |
|--------|---------|----------|-------------------|-------------------|------------|
| `prefetchQuery` | `void` | Swallows | Skips fetch | Refetches | Fetches |
| `fetchQuery` | `TData` | Throws | Skips fetch | Refetches | Fetches |
| `ensureQueryData` | `TData` | Throws | Returns cached | Returns cached* | Fetches |

*With `revalidateIfStale: true`, returns stale data immediately and refetches in background.

## Context

- `ensureQueryData` is ideal for route loaders — fast cache hits, only fetches when needed
- Default `revalidateIfStale` is `false` — stale data is returned without refetching
- Use `prefetchQuery` for fire-and-forget prefetching where you don't need the data immediately
- Use `fetchQuery` when you need the data and want to handle errors (e.g., responding with 404)
- `ensureQueryData` internally calls `fetchQuery` on cache miss
- Pair with `queryOptions` factory to share config between loader and component
