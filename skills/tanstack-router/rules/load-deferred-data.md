# load-deferred-data: Split Critical and Non-Critical Data

## Priority: HIGH

## Explanation

Not all data is equally important for the initial render. Await critical data in the loader to prevent layout shift, but return non-critical data as unawaited promises to stream in after the initial paint. This shows content faster while slower data loads in the background with Suspense boundaries.

## Bad Example

```tsx
// Awaiting everything — page blocked until slowest request finishes
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => {
    const post = await fetchPost(params.postId)           // 100ms
    const comments = await fetchComments(params.postId)   // 2000ms
    const recommendations = await fetchRecommendations()  // 3000ms
    return { post, comments, recommendations }
    // User waits 3+ seconds before seeing anything
  },
})
```

## Good Example: Built-In Deferred Loading

```tsx
import { createFileRoute, Await } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => {
    // Kick off slow requests — do NOT await
    const commentsPromise = fetchComments(params.postId)
    const recommendationsPromise = fetchRecommendations()

    // Await only critical data
    const post = await fetchPost(params.postId)

    return {
      post,
      deferredComments: commentsPromise,
      deferredRecommendations: recommendationsPromise,
    }
  },
  component: PostPage,
})

function PostPage() {
  const { post, deferredComments, deferredRecommendations } = Route.useLoaderData()

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>

      <Suspense fallback={<CommentsSkeleton />}>
        <Await promise={deferredComments}>
          {(comments) => <CommentsList comments={comments} />}
        </Await>
      </Suspense>

      <Suspense fallback={<RecommendationsSkeleton />}>
        <Await promise={deferredRecommendations}>
          {(recs) => <Recommendations items={recs} />}
        </Await>
      </Suspense>
    </article>
  )
}
```

## Good Example: With TanStack Query

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params, context: { queryClient } }) => {
    // Non-critical — prefetch without awaiting
    queryClient.prefetchQuery(commentsQueryOptions(params.postId))
    queryClient.prefetchQuery(recommendationsQueryOptions())

    // Critical — await to prevent loading state
    await queryClient.ensureQueryData(postQueryOptions(params.postId))
  },
  component: PostPage,
})

function PostPage() {
  const { postId } = Route.useParams()
  const { data: post } = useSuspenseQuery(postQueryOptions(postId))

  return (
    <article>
      <h1>{post.title}</h1>
      <Suspense fallback={<CommentsSkeleton />}>
        <CommentsSection postId={postId} />
      </Suspense>
    </article>
  )
}

function CommentsSection({ postId }: { postId: string }) {
  const { data: comments } = useSuspenseQuery(commentsQueryOptions(postId))
  return <CommentsList comments={comments} />
}
```

## Context

- Critical data: content above the fold, page title, primary layout data — `await` these
- Non-critical data: comments, recommendations, analytics, sidebar content — defer these
- Deferred promises follow the same cache lifecycle as their loader
- Works with SSR streaming — deferred data streams in as inline `<script>` tags
- With TanStack Query, use `prefetchQuery` (not awaited) for deferred and `ensureQueryData` (awaited) for critical
- Wrap each deferred section in its own `<Suspense>` boundary for independent loading states
