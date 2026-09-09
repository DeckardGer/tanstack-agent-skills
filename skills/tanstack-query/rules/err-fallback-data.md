# err-fallback-data: Provide Fallback Data When Appropriate

## Priority: MEDIUM

## Explanation

Use `placeholderData` or `initialData` to show meaningful content while queries load or recover from errors. Choose based on whether the data should persist in the cache (`initialData`) or serve as temporary UI scaffolding (`placeholderData`).

## Bad Example

```tsx
// Detail page shows blank loading state despite having data from the list
function PostDetail({ postId }: { postId: string }) {
  const { data, isPending } = useQuery({
    queryKey: ['post', postId],
    queryFn: () => fetchPost(postId),
  })

  if (isPending) return <Skeleton /> // Unnecessary full skeleton
  return <Post data={data} />
}
```

## Good Example: placeholderData from Cache

```tsx
function PostDetail({ postId }: { postId: string }) {
  const queryClient = useQueryClient()

  const { data, isPlaceholderData } = useQuery({
    queryKey: ['post', postId],
    queryFn: () => fetchPost(postId),
    placeholderData: () => {
      // Use partial data from the list cache as placeholder
      return queryClient
        .getQueryData<Post[]>(['posts'])
        ?.find((post) => post.id === postId)
    },
  })

  return (
    <article style={{ opacity: isPlaceholderData ? 0.7 : 1 }}>
      <h1>{data?.title}</h1>
      {isPlaceholderData ? (
        <p>Loading full content...</p>
      ) : (
        <div>{data?.content}</div>
      )}
    </article>
  )
}
```

## Good Example: keepPreviousData for Pagination

```tsx
import { keepPreviousData } from '@tanstack/react-query'

function ProductList({ page }: { page: number }) {
  const { data, isPlaceholderData } = useQuery({
    queryKey: ['products', page],
    queryFn: () => fetchProducts(page),
    placeholderData: keepPreviousData, // Show previous page while loading next
  })

  return (
    <div style={{ opacity: isPlaceholderData ? 0.5 : 1 }}>
      {data?.map((product) => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  )
}
```

## placeholderData vs initialData

| Behavior | `initialData` | `placeholderData` |
|----------|---------------|-------------------|
| Persisted to cache | Yes | No |
| `staleTime` applies | Yes | No (always fetches) |
| `isPlaceholderData` | `false` | `true` |
| Visible to other components | Yes | No |
| On fetch failure | Data preserved | Data becomes `undefined` |
| Best for | SSR data, complete known data | Preview data, pagination |

## Context

- `placeholderData` never persists to cache — it's observer-level only
- `isPlaceholderData` flag lets you style placeholder state differently
- `initialData` affects `dataUpdatedAt` — use `initialDataUpdatedAt` to control staleness
- `keepPreviousData` is a built-in helper that preserves the previous query's data
- For SSR, prefer `initialData` or the hydration API over `placeholderData`
- Use `placeholderData` as a function for lazy evaluation from other cache entries
