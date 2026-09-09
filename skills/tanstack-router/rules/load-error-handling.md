# load-error-handling: Handle Loader Errors Appropriately

## Priority: HIGH

## Explanation

Loader errors are caught by the route's `errorComponent` (or the router's `defaultErrorComponent`). Use `router.invalidate()` to retry after loader errors — not the `reset` function, which only resets React render errors. For missing resources, throw `notFound()` instead of returning null. When using TanStack Query, also reset the query error boundary state.

## Bad Example

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => {
    const post = await fetchPost(params.postId)
    return post // Could be null — component must handle it
  },
  errorComponent: ({ error, reset }) => (
    <div>
      <p>Error: {error.message}</p>
      {/* reset() only works for render errors, not loader errors */}
      <button onClick={reset}>Retry</button>
    </div>
  ),
})
```

## Good Example

```tsx
import { createFileRoute, notFound, ErrorComponent, useRouter } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params: { postId } }) => {
    const post = await fetchPost(postId)
    if (!post) throw notFound() // Handled by notFoundComponent
    return { post }
  },
  errorComponent: PostError,
  notFoundComponent: () => {
    const { postId } = Route.useParams()
    return <p>Post {postId} was not found.</p>
  },
})

function PostError({ error, reset }: ErrorComponentProps) {
  const router = useRouter()

  // For known error types, render custom UI
  if (error instanceof FetchError && error.status === 403) {
    return <p>You don't have permission to view this post.</p>
  }

  // For unknown errors, fall back to default error component
  return (
    <div>
      <ErrorComponent error={error} />
      <button
        onClick={() => {
          // router.invalidate() reloads loaders AND resets the error boundary
          router.invalidate()
        }}
      >
        Retry
      </button>
    </div>
  )
}
```

## Good Example: With TanStack Query

```tsx
import { useQueryErrorResetBoundary } from '@tanstack/react-query'

export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params, context: { queryClient } }) =>
    queryClient.ensureQueryData(postQueryOptions(params.postId)),
  errorComponent: PostError,
})

function PostError({ error, reset }: ErrorComponentProps) {
  const router = useRouter()
  const queryErrorResetBoundary = useQueryErrorResetBoundary()

  useEffect(() => {
    // Reset TanStack Query's error state when error component mounts
    queryErrorResetBoundary.reset()
  }, [queryErrorResetBoundary])

  return (
    <div>
      <ErrorComponent error={error} />
      <button onClick={() => router.invalidate()}>Retry</button>
    </div>
  )
}
```

## Error Handling Callbacks

```tsx
export const Route = createFileRoute('/posts')({
  loader: () => fetchPosts(),
  onError: ({ error }) => {
    // Called when loader throws — use for logging/reporting
    reportError(error)
  },
  onCatch: ({ error, errorInfo }) => {
    // Called when error caught by router's CatchBoundary
    logToService(error, errorInfo)
  },
})
```

## Context

- Use `router.invalidate()` for loader errors — it reloads all active loaders and resets error boundaries
- Use `reset()` only for React render errors (component-level throws)
- Throw `notFound()` for missing resources — don't return null and check in the component
- `onError` fires when the loader throws — use it for logging, not recovery
- Router-wide defaults: set `defaultErrorComponent` on the router for a fallback
- `ErrorComponent` from `@tanstack/react-router` provides a styled default you can use as a fallback
