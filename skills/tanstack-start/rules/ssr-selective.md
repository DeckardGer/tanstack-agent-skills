# ssr-selective: Apply Selective SSR When Beneficial

## Priority: MEDIUM

## Explanation

Not every route needs full server-side rendering. TanStack Start provides per-route SSR control via the `ssr` option: `true` (default, full SSR), `false` (client-only rendering), or `'data-only'` (server data loading but client rendering). This optimizes server load while preserving SEO and initial load performance where needed.

## Bad Example

```tsx
// SSR for a heavy interactive dashboard - wastes server resources
// and the server-rendered HTML is immediately replaced by client state
export const Route = createFileRoute('/app/dashboard')({
  // ssr: true is default - server renders the full component tree
  loader: () => getDashboardData(),
  component: HeavyInteractiveDashboard,
})

// No SSR for a marketing landing page - bad for SEO
export const Route = createFileRoute('/')({
  ssr: false,  // Google sees a blank page
  component: LandingPage,
})
```

## Good Example

```tsx
import { createFileRoute } from '@tanstack/react-router'

// Full SSR for public/SEO-critical pages (default)
export const Route = createFileRoute('/blog/$slug')({
  // ssr: true is default — full server rendering
  loader: ({ params }) => getPost({ data: { slug: params.slug } }),
  component: BlogPost,
})

// Client-only for heavy interactive apps — no server rendering overhead
export const Route = createFileRoute('/app/dashboard')({
  ssr: false,  // No beforeLoad/loader on server, no server rendering
  loader: () => getDashboardData(),
  component: Dashboard,
})

// Data-only for pages where data matters but rendering doesn't
export const Route = createFileRoute('/app/settings')({
  ssr: 'data-only',  // Runs loader on server, renders on client only
  loader: () => getUserSettings(),
  component: SettingsPage,
})
```

## Good Example: Dynamic SSR with Function Form

```tsx
import { z } from 'zod'

export const Route = createFileRoute('/docs/$docType/$docId')({
  validateSearch: z.object({ details: z.boolean().optional() }),
  ssr: ({ params, search }) => {
    // Disable SSR for heavy document types
    if (params.status === 'success' && params.value.docType === 'spreadsheet') {
      return false
    }
    // Data-only for detail views
    if (search.status === 'success' && search.value.details) {
      return 'data-only'
    }
    // Default: full SSR
  },
  loader: ({ params }) => getDocument({ data: params }),
  component: DocumentViewer,
})
```

## Good Example: Global Default and Root Shell

```tsx
// src/start.ts — disable SSR globally for a SPA-like app
import { createStart } from '@tanstack/react-start'

export const startInstance = createStart(() => ({
  defaultSsr: false,
}))

// When root route has ssr: false, you need a shellComponent
// routes/__root.tsx
export const Route = createRootRoute({
  ssr: false,
  shellComponent: RootShell,
  component: RootComponent,
})

function RootShell({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head><HeadContent /></head>
      <body>{children}<Scripts /></body>
    </html>
  )
}
```

## SSR Modes Reference

| Mode | `beforeLoad`/`loader` on server | Server renders component | Use case |
|------|------|------|------|
| `true` | Yes | Yes | SEO pages, landing pages, blogs |
| `'data-only'` | Yes | No | Data-heavy apps, authenticated pages |
| `false` | No | No | Interactive dashboards, admin panels |

## Context

- Child routes inherit parent SSR settings and can only make them MORE restrictive
- Inheritance hierarchy: `true` → `'data-only'` → `false` (child can only go right)
- When `ssr: false` on the root route, provide a `shellComponent` for the HTML shell
- The `ssr` function form receives `params` and `search` for dynamic decisions
- Global default SSR is set in `src/start.ts` via `createStart(() => ({ defaultSsr }))`
- `ssr: false` routes still work with server functions — they just don't SSR the component
