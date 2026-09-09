# split-critical-path: Keep Critical Config in Main Route File

## Priority: MEDIUM

## Explanation

When code splitting, the main route file must contain everything needed for route matching and data loading. This includes path config, search validation, `beforeLoad`, `loader`, `loaderDeps`, context, and static data. Only UI components (`component`, `errorComponent`, `pendingComponent`, `notFoundComponent`) go in the lazy file. Putting loaders in lazy files adds a network round-trip before data can load.

## Bad Example

```tsx
// routes/dashboard.tsx — empty, everything in lazy file
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/dashboard')({
  // Nothing here — loader is in the lazy file
})

// routes/dashboard.lazy.tsx
import { createLazyFileRoute } from '@tanstack/react-router'

export const Route = createLazyFileRoute('/dashboard')({
  // BAD: loader can't go in lazy files — it needs to run before rendering
  // loader: () => fetchDashboardData(),  // This won't work!
  component: DashboardPage,
})
```

```tsx
// Exporting component functions — prevents automatic code splitting
// routes/posts.tsx
export const Route = createFileRoute('/posts')({
  component: PostsComponent,
})

// BAD: exported functions are included in the main bundle
export function PostsComponent() {
  return <div>Posts</div>
}
```

## Good Example

```tsx
// routes/dashboard.tsx — all critical config here
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/dashboard')({
  validateSearch: dashboardSearchSchema,
  beforeLoad: ({ context }) => {
    if (!context.user) throw redirect({ to: '/login' })
  },
  loaderDeps: ({ search: { period } }) => ({ period }),
  loader: ({ deps, context: { queryClient } }) =>
    queryClient.ensureQueryData(dashboardQueryOptions(deps)),
  // No component — it's in the lazy file
})

// routes/dashboard.lazy.tsx — only UI
import { createLazyFileRoute } from '@tanstack/react-router'

export const Route = createLazyFileRoute('/dashboard')({
  component: DashboardPage,
  pendingComponent: DashboardSkeleton,
  errorComponent: DashboardError,
  notFoundComponent: DashboardNotFound,
})

// NOT exported — stays code-split
function DashboardPage() {
  const data = Route.useLoaderData()
  return <div>{/* heavy UI */}</div>
}
```

## What Goes Where

| Main Route File | Lazy File |
|---|---|
| `validateSearch` | `component` |
| `beforeLoad` | `errorComponent` |
| `loader` | `pendingComponent` |
| `loaderDeps` | `notFoundComponent` |
| `context` | |
| `staticData` | |
| `head` | |

## Context

- The loader is already an async boundary — splitting it adds a second network round-trip (fetch chunk + execute loader)
- Loaders typically contribute less to bundle size than components with heavy UI libraries
- The loader is a key preloadable asset (e.g. on link hover) — it should be available without async overhead
- Do NOT export component functions from route files — exported values bypass code splitting
- `createLazyFileRoute` only accepts `component`, `errorComponent`, `pendingComponent`, `notFoundComponent`
- Use `getRouteApi()` in lazy files for type-safe access to loader data, params, and search without importing the main route file
