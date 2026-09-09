# ts-query-options-loader: Use queryOptions in Loaders for Type Inference

## Priority: CRITICAL

## Explanation

Define query configurations with `queryOptions()` and share them between route loaders and components. This ensures the query key, fetch function, and return type stay in sync. When loaders return `void` (using `await`) instead of returning data directly, TypeScript has less to infer per route, improving editor performance in large apps.

## Bad Example

```tsx
// Inline query config — duplicated between loader and component, easy to desync
export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params, context: { queryClient } }) =>
    queryClient.ensureQueryData({
      queryKey: ['posts', params.postId],
      queryFn: () => fetchPost(params.postId),
    }),
  component: PostPage,
})

function PostPage() {
  const { postId } = Route.useParams()
  // Different query key or staleTime here would cause a cache miss
  const { data } = useSuspenseQuery({
    queryKey: ['post', postId], // Bug: 'post' vs 'posts'
    queryFn: () => fetchPost(postId),
  })
  return <div>{data.title}</div>
}
```

## Good Example

```tsx
// Shared queryOptions — single source of truth
import { queryOptions } from '@tanstack/react-query'

const postQueryOptions = (postId: string) =>
  queryOptions({
    queryKey: ['posts', postId],
    queryFn: () => fetchPost(postId),
    staleTime: 5 * 60 * 1000,
  })

export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params, context: { queryClient } }) => {
    // Return void — avoids unnecessary type inference on loader return
    await queryClient.ensureQueryData(postQueryOptions(params.postId))
  },
  component: PostPage,
})

function PostPage() {
  const { postId } = Route.useParams()
  // Same queryOptions — guaranteed cache hit, fully typed
  const { data: post } = useSuspenseQuery(postQueryOptions(postId))
  return <div>{post.title}</div>
}
```

## Good Example: Organizing Query Options

```tsx
// api/posts.ts — centralized query option factories
import { queryOptions } from '@tanstack/react-query'

export const postsQueryOptions = () =>
  queryOptions({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  })

export const postQueryOptions = (postId: string) =>
  queryOptions({
    queryKey: ['posts', postId],
    queryFn: () => fetchPost(postId),
  })

export const postCommentsQueryOptions = (postId: string) =>
  queryOptions({
    queryKey: ['posts', postId, 'comments'],
    queryFn: () => fetchPostComments(postId),
  })
```

## Context

- `queryOptions()` returns a typed config object reusable across `ensureQueryData`, `useSuspenseQuery`, `useQuery`, and `prefetchQuery`
- Use `async`/`await` in loaders to return `Promise<void>` — this reduces TypeScript inference burden
- Keep query option factories in separate files to avoid circular imports between routes
- `useSuspenseQuery` is preferred over `useQuery` when the loader guarantees data exists
