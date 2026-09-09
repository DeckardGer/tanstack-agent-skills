# org-pathless-layouts: Use Pathless Routes for Shared Layouts

## Priority: CRITICAL

## Explanation

Pathless layout routes (prefixed with `_`) wrap child routes with shared UI without adding a path segment to the URL. Use them for shared layouts, authentication boundaries, or visual groupings where the URL shouldn't reflect the layout structure.

## Bad Example

```tsx
// Adding a path segment just for layout purposes
// URL becomes /app/dashboard instead of /dashboard
routes/
├── app.tsx                 # /app — unnecessary path segment
├── app.dashboard.tsx       # /app/dashboard
├── app.settings.tsx        # /app/settings

// Or duplicating layout in every route
// routes/dashboard.tsx
export const Route = createFileRoute('/dashboard')({
  component: () => (
    <AppShell>           {/* Duplicated in every sibling route */}
      <DashboardPage />
    </AppShell>
  ),
})
```

## Good Example

```
# Pathless layout — no URL impact, shared layout
routes/
├── __root.tsx
├── _app.tsx                # Pathless layout (no URL segment)
├── _app.dashboard.tsx      # /dashboard
├── _app.settings.tsx       # /settings
├── login.tsx               # /login (outside the _app layout)
```

```tsx
// routes/_app.tsx — wraps children without affecting URL
import { createFileRoute, Outlet } from '@tanstack/react-router'

export const Route = createFileRoute('/_app')({
  beforeLoad: ({ context }) => {
    if (!context.user) throw redirect({ to: '/login' })
  },
  component: AppLayout,
})

function AppLayout() {
  return (
    <div className="app-shell">
      <Sidebar />
      <main>
        <Outlet />
      </main>
    </div>
  )
}

// routes/_app.dashboard.tsx — URL is /dashboard, not /_app/dashboard
export const Route = createFileRoute('/_app/dashboard')({
  component: DashboardPage,
})
```

## Directory Style

```
routes/
├── _app/
│   ├── route.tsx          # Pathless layout definition
│   ├── dashboard.tsx      # /dashboard
│   └── settings.tsx       # /settings
├── login.tsx              # /login (outside layout)
```

## Multiple Pathless Layouts

```
routes/
├── _auth.tsx              # Auth layout (login/register styling)
├── _auth.login.tsx        # /login
├── _auth.register.tsx     # /register
├── _app.tsx               # App layout (sidebar, nav)
├── _app.dashboard.tsx     # /dashboard
├── _app.settings.tsx      # /settings
```

## Context

- The `_` prefix marks a route as pathless — it doesn't contribute to the URL
- Child routes of pathless layouts are matched by their own path, not the layout's
- Pathless layouts can define `beforeLoad`, `loader`, search validation, error boundaries — all inherited by children
- Pathless layouts cannot use dynamic segments (`_$param/` is invalid)
- Use for auth boundaries, visual chrome, or grouping routes that share UI but not URL structure
