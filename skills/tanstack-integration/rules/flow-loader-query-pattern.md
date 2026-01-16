# flow-loader-query-pattern: Use Loaders with ensureQueryData

## Priority: HIGH

## Explanation

The recommended pattern combines TanStack Router's loaders with TanStack Query's `ensureQueryData`. Loaders prefetch data during navigation, while Query manages caching and updates. Use `useSuspenseQuery` in components since data is guaranteed.

## Bad Example

```tsx
// Only using Query in component - loading waterfall
function PostsPage() {
  const { data, isLoading } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  })

  if (isLoading) return <Loading />
  return <PostList posts={data} />
}

// Only using loader - no cache management
export const Route = createFileRoute('/posts')({
  loader: async () => {
    const posts = await fetchPosts()  // Not cached
    return { posts }
  },
})
```

## Good Example

```tsx
// lib/queries/posts.ts - Define queryOptions
import { queryOptions } from '@tanstack/react-query'

export const postQueries = {
  all: () => queryOptions({
    queryKey: ['posts'],
    queryFn: fetchPosts,
    staleTime: 5 * 60 * 1000,
  }),
  detail: (id: string) => queryOptions({
    queryKey: ['posts', id],
    queryFn: () => fetchPost(id),
    staleTime: 5 * 60 * 1000,
  }),
}

// routes/posts.tsx
import { createFileRoute } from '@tanstack/react-router'
import { useSuspenseQuery } from '@tanstack/react-query'
import { postQueries } from '@/lib/queries/posts'

export const Route = createFileRoute('/posts')({
  loader: async ({ context: { queryClient } }) => {
    // Prefetch in loader - runs during navigation
    await queryClient.ensureQueryData(postQueries.all())
  },
  component: PostsPage,
})

function PostsPage() {
  // Data guaranteed by loader - no loading state needed
  const { data: posts } = useSuspenseQuery(postQueries.all())

  return <PostList posts={posts} />
}

// routes/posts/$postId.tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params, context: { queryClient } }) => {
    await queryClient.ensureQueryData(postQueries.detail(params.postId))
  },
  component: PostDetailPage,
})

function PostDetailPage() {
  const { postId } = Route.useParams()
  const { data: post } = useSuspenseQuery(postQueries.detail(postId))

  return <PostContent post={post} />
}
```

## Good Example: Parallel Data Loading

```tsx
export const Route = createFileRoute('/dashboard')({
  loader: async ({ context: { queryClient } }) => {
    // Load multiple queries in parallel
    await Promise.all([
      queryClient.ensureQueryData(statsQueries.overview()),
      queryClient.ensureQueryData(activityQueries.recent()),
      queryClient.ensureQueryData(userQueries.current()),
    ])
  },
  component: DashboardPage,
})

function DashboardPage() {
  // All data ready - no loading states
  const { data: stats } = useSuspenseQuery(statsQueries.overview())
  const { data: activity } = useSuspenseQuery(activityQueries.recent())
  const { data: user } = useSuspenseQuery(userQueries.current())

  return (
    <Dashboard
      stats={stats}
      activity={activity}
      user={user}
    />
  )
}
```

## Good Example: Optional Prefetch with Non-Critical Data

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params, context: { queryClient } }) => {
    // Critical data - await it
    await queryClient.ensureQueryData(postQueries.detail(params.postId))

    // Non-critical - prefetch but don't await
    queryClient.prefetchQuery(postQueries.comments(params.postId))
    queryClient.prefetchQuery(postQueries.related(params.postId))
  },
  component: PostPage,
})

function PostPage() {
  const { postId } = Route.useParams()

  // Critical - guaranteed by loader
  const { data: post } = useSuspenseQuery(postQueries.detail(postId))

  // Non-critical - may still be loading
  const { data: comments, isLoading: commentsLoading } = useQuery(
    postQueries.comments(postId)
  )

  return (
    <article>
      <PostContent post={post} />
      {commentsLoading ? <CommentsSkeleton /> : <Comments data={comments} />}
    </article>
  )
}
```

## Data Flow Summary

```
Navigation Starts
       ↓
Router matches route
       ↓
loader() executes
       ↓
ensureQueryData() checks cache
       ↓
Fresh cache? → Return cached     Stale/missing? → Fetch and cache
       ↓                                    ↓
Route renders                         Route renders
       ↓                                    ↓
useSuspenseQuery returns data     useSuspenseQuery returns data
```

## Context

- `ensureQueryData` respects staleTime - won't refetch fresh data
- `useSuspenseQuery` throws promise to Suspense if data missing
- Loaders enable preloading on link hover
- This pattern eliminates loading waterfalls
- Use `useQuery` for non-critical data that can load after render
- Query invalidation and background updates still work normally
