# org-file-based-routing: Prefer File-Based Routing for Conventions

## Priority: CRITICAL

## Explanation

File-based routing generates route configuration automatically from your file structure via the TanStack Router bundler plugin or CLI. It requires less code than code-based routing, enforces consistent conventions, provides automatic code-splitting, and generates type linkages. Use it as the default unless you have a specific reason not to.

## Bad Example

```tsx
// Code-based routing — verbose, manual, error-prone
import { createRouter, createRoute, createRootRoute } from '@tanstack/react-router'

const rootRoute = createRootRoute({ component: Root })
const indexRoute = createRoute({ getParentRoute: () => rootRoute, path: '/', component: Home })
const aboutRoute = createRoute({ getParentRoute: () => rootRoute, path: '/about', component: About })
const postsRoute = createRoute({ getParentRoute: () => rootRoute, path: '/posts', component: Posts })
const postRoute = createRoute({ getParentRoute: () => postsRoute, path: '/$postId', component: Post })

// Manual tree assembly — must keep in sync as routes are added
const routeTree = rootRoute.addChildren([
  indexRoute,
  aboutRoute,
  postsRoute.addChildren([postRoute]),
])

const router = createRouter({ routeTree })
```

## Good Example

```
# File structure generates the same route tree automatically
routes/
├── __root.tsx          # Root layout
├── index.tsx           # / (exact)
├── about.tsx           # /about
├── posts.tsx           # /posts (layout)
├── posts.index.tsx     # /posts (exact)
└── posts.$postId.tsx   # /posts/$postId
```

```tsx
// routes/__root.tsx
import { createRootRoute, Outlet } from '@tanstack/react-router'

export const Route = createRootRoute({
  component: () => (
    <div>
      <nav>{/* ... */}</nav>
      <Outlet />
    </div>
  ),
})

// routes/posts.$postId.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params }) => fetchPost(params.postId),
  component: PostPage,
})

function PostPage() {
  const post = Route.useLoaderData()
  return <article>{post.title}</article>
}
```

## Bundler Setup

```tsx
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { tanstackRouter } from '@tanstack/router-plugin/vite'

export default defineConfig({
  plugins: [
    tanstackRouter(), // Generates routeTree.gen.ts automatically
    react(),
  ],
})
```

## File Naming Conventions

| Convention | Example | Meaning |
|---|---|---|
| `__root.tsx` | `__root.tsx` | Root route |
| `.` separator | `posts.index.tsx` | Nesting (flat style) |
| `$` prefix | `$postId` | Dynamic segment |
| `_` prefix | `_layout.tsx` | Pathless layout |
| `_` suffix | `posts_.$id.edit.tsx` | Non-nested route |
| `-` prefix | `-components/` | Excluded from routing |
| `(folder)` | `(auth)/login.tsx` | Organizational group |

## Context

- File-based and code-based routing support the same features — file-based just requires less code
- The path string in `createFileRoute('/posts/$postId')` is auto-managed by the plugin — don't edit it manually
- Use flat style (`posts.index.tsx`) or directory style (`posts/index.tsx`) or mix both
- Supports Vite, Rspack/Rsbuild, Webpack, and Esbuild
- Route tree is regenerated on file changes during development
