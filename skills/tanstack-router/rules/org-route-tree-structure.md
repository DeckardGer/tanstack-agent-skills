# org-route-tree-structure: Follow Hierarchical Route Tree Patterns

## Priority: CRITICAL

## Explanation

TanStack Router uses a nested route tree to match URLs to the correct component hierarchy. Each route in the tree can define its own layout, loader, error boundary, and search params. Child routes render inside their parent's `<Outlet />`. Understanding this nesting is essential for correct layouts, data loading, and code organization.

## Bad Example

```tsx
// Flat list of routes with no nesting — no shared layouts or data
routes/
├── __root.tsx
├── dashboard.tsx           # /dashboard
├── dashboard-stats.tsx     # /dashboard-stats (separate route, not nested!)
├── dashboard-settings.tsx  # /dashboard-settings (no shared layout)
├── dashboard-users.tsx     # /dashboard-users (auth check duplicated)
```

```tsx
// Each route must duplicate the dashboard layout and auth logic
// dashboard-stats.tsx
export const Route = createFileRoute('/dashboard-stats')({
  beforeLoad: () => { /* duplicate auth check */ },
  component: () => (
    <DashboardLayout>  {/* duplicate layout wrapper */}
      <Stats />
    </DashboardLayout>
  ),
})
```

## Good Example

```
# Nested tree — shared layout and auth handled once in parent
routes/
├── __root.tsx
├── dashboard.tsx           # /dashboard (layout + auth)
├── dashboard.index.tsx     # /dashboard (exact — default view)
├── dashboard.stats.tsx     # /dashboard/stats
├── dashboard.settings.tsx  # /dashboard/settings
└── dashboard.users.tsx     # /dashboard/users
```

```tsx
// dashboard.tsx — parent layout, runs once for all children
import { createFileRoute, Outlet } from '@tanstack/react-router'

export const Route = createFileRoute('/dashboard')({
  beforeLoad: ({ context }) => {
    if (!context.user) throw redirect({ to: '/login' })
  },
  component: DashboardLayout,
})

function DashboardLayout() {
  return (
    <div>
      <DashboardSidebar />
      <main>
        <Outlet /> {/* Child routes render here */}
      </main>
    </div>
  )
}

// dashboard.stats.tsx — inherits layout and auth from parent
export const Route = createFileRoute('/dashboard/stats')({
  loader: ({ context: { queryClient } }) =>
    queryClient.ensureQueryData(statsQueryOptions()),
  component: StatsPage,
})
```

## Directory Style Equivalent

```
routes/
├── __root.tsx
├── dashboard/
│   ├── route.tsx          # /dashboard layout (use route.tsx for the parent)
│   ├── index.tsx          # /dashboard (exact)
│   ├── stats.tsx          # /dashboard/stats
│   ├── settings.tsx       # /dashboard/settings
│   └── users.tsx          # /dashboard/users
```

## Context

- Parent route components must render `<Outlet />` for children to appear
- Layout routes run `beforeLoad` and `loader` before any child route
- Search params, context, and error boundaries are inherited by child routes
- Use flat (`.` separator) or directory style interchangeably — they produce the same tree
- Use `route.tsx` inside a directory to define the parent route's configuration
- Routes are automatically sorted by specificity: index > static > dynamic > splat
