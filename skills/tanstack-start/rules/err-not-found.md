# err-not-found: Handle Not-Found Scenarios

## Priority: MEDIUM

## Explanation

Use `throw notFound()` when a requested resource doesn't exist. This triggers the nearest `notFoundComponent` in the route tree, providing a consistent not-found experience. The router also automatically throws not-found for unmatched URL paths.

## Bad Example

```tsx
// Returning null/undefined instead of throwing notFound
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => {
    const post = await getPost(params.postId)
    return post  // Returns undefined if not found - component crashes
  },
  component: PostComponent,
})

function PostComponent() {
  const post = Route.useLoaderData()
  // post might be undefined - renders broken UI instead of 404
  return <h1>{post.title}</h1>
}

// Using error boundaries for not-found - wrong semantics
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => {
    const post = await getPost(params.postId)
    if (!post) throw new Error('Post not found')  // Shows error UI, not 404
    return post
  },
})
```

## Good Example

```tsx
import { createFileRoute, notFound } from '@tanstack/react-router'
import { createServerFn } from '@tanstack/react-start'

// Throw notFound in server functions
const getPost = createServerFn()
  .inputValidator((data: { id: string }) => data)
  .handler(async ({ data }) => {
    const post = await db.posts.findUnique({ where: { id: data.id } })
    if (!post) {
      throw notFound()
    }
    return post
  })

// Route with notFoundComponent
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => {
    return getPost({ data: { id: params.postId } })
  },
  notFoundComponent: () => {
    return (
      <div>
        <h2>Post not found</h2>
        <p>The post you're looking for doesn't exist or has been removed.</p>
        <Link to="/posts">Browse all posts</Link>
      </div>
    )
  },
  component: PostComponent,
})

function PostComponent() {
  const post = Route.useLoaderData()
  // post is guaranteed to exist — notFound was thrown otherwise
  return <h1>{post.title}</h1>
}
```

## Good Example: Not-Found in beforeLoad

```tsx
// Throwing notFound in beforeLoad targets the root notFoundComponent
export const Route = createFileRoute('/users/$username')({
  beforeLoad: async ({ params }) => {
    const user = await getUserByUsername(params.username)
    if (!user) throw notFound()
    return { user }
  },
  component: UserProfile,
})

// Root not-found handler
export const Route = createRootRoute({
  notFoundComponent: () => (
    <div>
      <h1>404 — Page not found</h1>
      <Link to="/">Go home</Link>
    </div>
  ),
})
```

## Good Example: Targeting a Specific Route

```tsx
import { notFound, rootRouteId } from '@tanstack/react-router'

export const Route = createFileRoute('/org/$orgId/project/$projectId')({
  loader: async ({ params }) => {
    const project = await getProject(params.projectId)
    if (!project) {
      // Target the org layout's notFoundComponent instead of this route's
      throw notFound({ routeId: '/org/$orgId' })
    }
    return project
  },
})
```

## Context

- `throw notFound()` triggers the nearest `notFoundComponent` up the route tree
- Throwing in `beforeLoad` always targets the root `notFoundComponent`
- Throwing in `loader` targets the current route's `notFoundComponent`
- Use `notFound({ routeId })` to target a specific route's not-found handler
- Router `notFoundMode: 'fuzzy'` (default) finds the closest parent with `notFoundComponent`
- Router `notFoundMode: 'root'` sends all not-founds to the root route
- `NotFoundRoute` is deprecated — use `notFoundComponent` on routes instead
- `notFound` is from `@tanstack/react-router`
