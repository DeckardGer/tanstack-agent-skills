# pf-route-prefetch: Prefetch Data During Route Transitions

## Priority: MEDIUM

## Explanation

Prefetch query data in route loaders so it's available instantly when the page renders. Use `ensureQueryData` for critical data that should block navigation, and `prefetchQuery` for non-critical data that can load in the background.

## Bad Example

```tsx
// No prefetching — data only loads after route renders
export const Route = createFileRoute('/posts/$postId')({
  component: PostPage,
})

function PostPage() {
  const { postId } = Route.useParams()
  // Fetch starts only after component mounts — shows loading spinner
  const { data, isPending } = useQuery({
    queryKey: ['post', postId],
    queryFn: () => fetchPost(postId),
  })

  if (isPending) return <Skeleton />
  return <Post data={data} />
}
```

## Good Example

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params, context: { queryClient } }) => {
    // Critical data — block navigation until loaded
    await queryClient.ensureQueryData({
      queryKey: ['post', params.postId],
      queryFn: () => fetchPost(params.postId),
    })

    // Non-critical — prefetch but don't block navigation
    queryClient.prefetchQuery({
      queryKey: ['post-comments', params.postId],
      queryFn: () => fetchComments(params.postId),
    })
  },
  component: PostPage,
})

function PostPage() {
  const { postId } = Route.useParams()

  // Data is already in cache — renders instantly
  const { data: post } = useSuspenseQuery({
    queryKey: ['post', postId],
    queryFn: () => fetchPost(postId),
  })

  // May still be loading, but won't block the page
  const { data: comments, isPending } = useQuery({
    queryKey: ['post-comments', postId],
    queryFn: () => fetchComments(postId),
  })

  return (
    <article>
      <PostContent post={post} />
      {isPending ? <CommentsSkeleton /> : <Comments data={comments} />}
    </article>
  )
}
```

## Good Example: Parallel Prefetching

```tsx
loader: async ({ params, context: { queryClient } }) => {
  // Prefetch multiple independent queries in parallel
  await Promise.all([
    queryClient.ensureQueryData({
      queryKey: ['post', params.postId],
      queryFn: () => fetchPost(params.postId),
    }),
    queryClient.ensureQueryData({
      queryKey: ['author', params.postId],
      queryFn: () => fetchAuthor(params.postId),
    }),
  ])
}
```

## Context

- `ensureQueryData` returns cached data if available, only fetches on cache miss
- `prefetchQuery` never throws — failed prefetches silently retry when the component mounts
- Without a `staleTime` above 0, prefetched data is immediately stale and refetches on mount
- Set `staleTime` on the corresponding `useQuery` to prevent double-fetching
- Use `queryOptions` factory to share configuration between loader and component
- For SSR, see the `ssr-dehydration` rule for the full hydration pattern
