# perf-placeholder-data: Use placeholderData for Instant UI

## Priority: LOW

## Explanation

`placeholderData` provides temporary data to display immediately while the real data loads. Unlike `initialData`, it is not persisted to the cache. Use it to show preview content from other cache entries, keep previous page data during pagination, or display skeleton-like content without a loading state.

## Bad Example

```tsx
// Full loading state for paginated data — content disappears on page change
function ProductList({ page }: { page: number }) {
  const { data, isPending } = useQuery({
    queryKey: ['products', page],
    queryFn: () => fetchProducts(page),
  })

  if (isPending) return <Skeleton /> // Entire list disappears while loading
  return <ProductGrid products={data} />
}

// Detail page shows nothing while loading despite having list data
function UserProfile({ userId }: { userId: string }) {
  const { data, isPending } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  })

  if (isPending) return <Skeleton /> // Could show name/avatar from list
  return <Profile user={data} />
}
```

## Good Example: keepPreviousData for Pagination

```tsx
import { keepPreviousData } from '@tanstack/react-query'

function ProductList({ page }: { page: number }) {
  const { data, isPlaceholderData } = useQuery({
    queryKey: ['products', page],
    queryFn: () => fetchProducts(page),
    placeholderData: keepPreviousData,
  })

  return (
    <div style={{ opacity: isPlaceholderData ? 0.5 : 1 }}>
      <ProductGrid products={data} />
      <button
        onClick={() => setPage((p) => p + 1)}
        disabled={isPlaceholderData} // Prevent rapid page flipping
      >
        Next Page
      </button>
    </div>
  )
}
```

## Good Example: Placeholder from Another Cache Entry

```tsx
function UserProfile({ userId }: { userId: string }) {
  const queryClient = useQueryClient()

  const { data, isPlaceholderData } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    placeholderData: () => {
      // Pull partial data from the users list cache
      return queryClient
        .getQueryData<User[]>(['users'])
        ?.find((user) => user.id === userId)
    },
  })

  return (
    <div>
      <h1>{data?.name}</h1>
      {isPlaceholderData ? (
        <p>Loading full profile...</p>
      ) : (
        <ProfileDetails user={data} />
      )}
    </div>
  )
}
```

## Context

- `placeholderData` is never persisted to the cache — it's observer-level only
- `isPlaceholderData` is `true` while showing placeholder, transitions to `false` when real data arrives
- Always triggers a background fetch regardless of `staleTime`
- `keepPreviousData` is a built-in helper — import from `@tanstack/react-query`
- Use `isPlaceholderData` to visually indicate partial/stale content (opacity, skeleton overlays)
- For SSR-fetched data, use `initialData` or the hydration pattern instead
