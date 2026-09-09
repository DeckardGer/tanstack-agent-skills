# ssr-data-loading: Load Data Appropriately for SSR

## Priority: MEDIUM

## Explanation

Route loaders are isomorphic — they run on the server during SSR and on the client during navigation. Use server functions inside loaders for data that requires server-side access (databases, secrets). Understand the loading lifecycle to avoid common SSR pitfalls.

## Bad Example

```tsx
// Loader accessing server-only resources directly - breaks on client
export const Route = createFileRoute('/posts')({
  loader: async () => {
    // db import gets bundled into client code!
    const posts = await db.posts.findMany()
    return posts
  },
})

// Returning the entire search object from loaderDeps
export const Route = createFileRoute('/posts')({
  loaderDeps: ({ search }) => search,  // Reloads on ANY search change
  loader: async ({ deps }) => {
    return fetchPosts(deps)
  },
})

// Calling browser APIs during SSR
export const Route = createFileRoute('/dashboard')({
  loader: async () => {
    const width = window.innerWidth  // Crashes on server!
    return fetchLayout(width)
  },
})
```

## Good Example

```tsx
import { createServerFn } from '@tanstack/react-start'
import { createFileRoute } from '@tanstack/react-router'

// Server function handles server-only logic
const getPosts = createServerFn()
  .inputValidator((data: { offset: number; limit: number }) => data)
  .handler(async ({ data }) => {
    return db.posts.findMany({
      skip: data.offset,
      take: data.limit,
      orderBy: { createdAt: 'desc' },
    })
  })

export const Route = createFileRoute('/posts')({
  validateSearch: z.object({
    offset: z.number().default(0),
    limit: z.number().default(20),
  }),
  // Only extract the deps the loader actually uses
  loaderDeps: ({ search: { offset, limit } }) => ({ offset, limit }),
  loader: async ({ deps }) => {
    return getPosts({ data: deps })
  },
})
```

## Good Example: Using beforeLoad for Auth Context

```tsx
// beforeLoad runs serially (parent → child), before loaders
// Use it for auth checks that gate data loading
export const Route = createFileRoute('/_authed')({
  beforeLoad: async ({ location }) => {
    const user = await getCurrentUser()
    if (!user) {
      throw redirect({
        to: '/login',
        search: { redirect: location.href },
      })
    }
    return { user }  // Available to child routes via context
  },
})

// Child route can use context from parent's beforeLoad
export const Route = createFileRoute('/_authed/dashboard')({
  loader: async ({ context }) => {
    // context.user is guaranteed by parent's beforeLoad
    return getDashboardData({ data: { userId: context.user.id } })
  },
})
```

## Loading Lifecycle

```
1. Route Matching (top-down): params.parse → validateSearch
2. Pre-Loading (serial):      beforeLoad → onError
3. Loading (parallel):        component.preload? + loader
```

## Context

- Loaders run on both server (SSR) and client (navigation) — they are NOT server-only
- Use `createServerFn()` inside loaders to access databases, secrets, and server resources
- `loaderDeps` controls when the loader re-runs — only extract what's needed
- `beforeLoad` runs before `loader` and is serial (parent → child)
- Loader results are serialized and dehydrated during SSR, then hydrated on the client
- Default `staleTime` is `0` (always reload on navigation); `preloadStaleTime` is `30` seconds
- Use `router.invalidate()` to force all active loaders to re-run
