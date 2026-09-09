# ctx-before-load: Extend Context in beforeLoad

## Priority: LOW

## Explanation

Use `beforeLoad` to extend or modify the route context before the loader runs. Context returned from `beforeLoad` is merged with the parent context and made available to the route's `loader` and all child routes. This is the right place for auth checks, redirects, and injecting route-specific dependencies.

## Bad Example

```tsx
// Duplicating auth checks in every loader
export const Route = createFileRoute('/dashboard')({
  loader: async ({ context }) => {
    // Auth check in loader — runs after beforeLoad, too late for redirects
    if (!context.user) {
      throw redirect({ to: '/login' })
    }
    return fetchDashboard(context.user.id)
  },
})

export const Route = createFileRoute('/dashboard/settings')({
  loader: async ({ context }) => {
    // Same auth check duplicated — if parent didn't check, child must
    if (!context.user) {
      throw redirect({ to: '/login' })
    }
    return fetchSettings(context.user.id)
  },
})
```

## Good Example

```tsx
// routes/dashboard.tsx — auth check once in beforeLoad, inherited by children
export const Route = createFileRoute('/dashboard')({
  beforeLoad: async ({ context }) => {
    if (!context.user) {
      throw redirect({ to: '/login' })
    }

    // Return additional context for this route and all children
    return {
      dashboardApi: createDashboardApi(context.user.token),
    }
  },
  loader: ({ context: { dashboardApi } }) => {
    // dashboardApi is available — typed and guaranteed by beforeLoad
    return dashboardApi.getOverview()
  },
})

// routes/dashboard.settings.tsx — inherits auth + dashboardApi
export const Route = createFileRoute('/dashboard/settings')({
  loader: ({ context: { dashboardApi } }) => {
    // No auth check needed — parent's beforeLoad already handled it
    // dashboardApi is available from parent context
    return dashboardApi.getSettings()
  },
})
```

## Good Example: Building Context Incrementally

```tsx
// routes/__root.tsx — base context
export const Route = createRootRouteWithContext<{ queryClient: QueryClient }>()({
  component: RootLayout,
})

// routes/_auth.tsx — adds user context
export const Route = createFileRoute('/_auth')({
  beforeLoad: async ({ context }) => {
    const user = await context.queryClient.ensureQueryData(currentUserQuery())
    if (!user) throw redirect({ to: '/login' })
    return { user } // Now all _auth children have context.user
  },
})

// routes/_auth.dashboard.tsx — adds dashboard-specific context
export const Route = createFileRoute('/_auth/dashboard')({
  beforeLoad: ({ context }) => {
    // context.queryClient from root + context.user from _auth
    return {
      canManageUsers: context.user.role === 'admin',
    }
  },
  loader: ({ context }) => {
    // Has: queryClient, user, canManageUsers
    context.queryClient  // from root
    context.user         // from _auth beforeLoad
    context.canManageUsers // from this route's beforeLoad
  },
})
```

## Execution Order

```
beforeLoad runs SERIALLY top-down through the route tree:
1. __root.beforeLoad
2. _auth.beforeLoad        (can throw redirect here)
3. _auth.dashboard.beforeLoad

Then loaders run IN PARALLEL:
1. _auth.loader + _auth.dashboard.loader (parallel)
```

## Context

- `beforeLoad` runs serially top-down before any loaders — use it for auth guards and redirects
- Context returned from `beforeLoad` is merged with parent context and passed to `loader` and children
- `beforeLoad` receives `context`, `params`, `search`, `location`, and `cause`
- You cannot use React hooks in `beforeLoad` — pass hook values through router context via `RouterProvider`
- Throwing `redirect()` in `beforeLoad` prevents the route and all children from loading
- Throwing `notFound()` in `beforeLoad` always triggers the root `notFoundComponent`
- TypeScript automatically infers the merged context — no manual typing needed
