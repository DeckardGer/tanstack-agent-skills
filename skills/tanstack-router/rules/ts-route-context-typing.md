# ts-route-context-typing: Type Route Context with createRootRouteWithContext

## Priority: CRITICAL

## Explanation

Use `createRootRouteWithContext<YourContext>()` instead of `createRootRoute()` to define a typed router context. This makes context available with full type safety in every route's `beforeLoad`, `loader`, and components. Without it, context is `{}` and dependency injection through the router is untyped.

## Bad Example

```tsx
// routes/__root.tsx - No typed context
import { createRootRoute, Outlet } from '@tanstack/react-router'

export const Route = createRootRoute({
  component: () => <Outlet />,
})

// routes/posts.tsx
export const Route = createFileRoute('/posts')({
  loader: ({ context }) => {
    // context is {} — no type safety, no access to queryClient
    const posts = context.queryClient.ensureQueryData(postsQuery) // Type error
  },
})
```

## Good Example

```tsx
// routes/__root.tsx
import { createRootRouteWithContext, Outlet } from '@tanstack/react-router'
import type { QueryClient } from '@tanstack/react-query'

interface RouterContext {
  queryClient: QueryClient
}

export const Route = createRootRouteWithContext<RouterContext>()({
  component: () => <Outlet />,
})

// router.tsx
import { createRouter } from '@tanstack/react-router'
import { QueryClient } from '@tanstack/react-query'
import { routeTree } from './routeTree.gen'

const queryClient = new QueryClient()

const router = createRouter({
  routeTree,
  context: { queryClient }, // TypeScript enforces this matches RouterContext
})

// routes/posts.tsx
export const Route = createFileRoute('/posts')({
  loader: ({ context: { queryClient } }) => {
    // queryClient is fully typed — autocomplete and type checking work
    return queryClient.ensureQueryData(postsQueryOptions())
  },
})
```

## Context

- `createRootRouteWithContext` is a factory — note the double call `()()`
- Only define properties in the context type that are passed directly to `createRouter`
- Context added via `beforeLoad` in child routes is automatically inferred — no need to include it in the root type
- TypeScript will error if required context properties are missing from `createRouter`
- If all context properties are optional, passing context to `createRouter` is optional too
