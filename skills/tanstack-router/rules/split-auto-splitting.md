# split-auto-splitting: Enable autoCodeSplitting When Possible

## Priority: MEDIUM

## Explanation

The `autoCodeSplitting` option in the TanStack Router bundler plugin automatically splits route components into separate chunks without manual `.lazy.tsx` files. It separates `component`, `errorComponent`, and `notFoundComponent` into lazy-loaded virtual files per route. This is the simplest way to get code splitting and should be the default for new projects.

## Bad Example

```tsx
// Manual splitting everywhere — lots of boilerplate
// routes/posts.tsx
export const Route = createFileRoute('/posts')({
  loader: () => fetchPosts(),
})

// routes/posts.lazy.tsx
export const Route = createLazyFileRoute('/posts')({
  component: PostsPage,
})

// routes/posts.$postId.tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params }) => fetchPost(params.postId),
})

// routes/posts.$postId.lazy.tsx
export const Route = createLazyFileRoute('/posts/$postId')({
  component: PostPage,
})

// Every route needs two files — error-prone, tedious
```

## Good Example

```tsx
// vite.config.ts — one setting, all routes auto-split
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { tanstackRouter } from '@tanstack/router-plugin/vite'

export default defineConfig({
  plugins: [
    tanstackRouter({
      autoCodeSplitting: true,
    }),
    react(),
  ],
})

// routes/posts.tsx — single file, components auto-split
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts')({
  loader: () => fetchPosts(),
  component: PostsPage,       // Auto-split into a lazy chunk
  errorComponent: PostsError,  // Auto-split into a separate chunk
})

// Do NOT export component functions — they must stay internal
function PostsPage() {
  const posts = Route.useLoaderData()
  return <div>{/* ... */}</div>
}

function PostsError({ error }: { error: Error }) {
  return <div>Error: {error.message}</div>
}
```

## Customizing Split Groups

```tsx
// vite.config.ts — group components together
tanstackRouter({
  autoCodeSplitting: true,
  codeSplittingOptions: {
    // Put all component types in one chunk instead of separate chunks
    defaultBehavior: [
      ['component', 'pendingComponent', 'errorComponent', 'notFoundComponent'],
    ],
  },
})
```

## Per-Route Overrides

```tsx
// Split the loader too for this specific heavy route
export const Route = createFileRoute('/analytics')({
  codeSplitGroupings: [['loader', 'component']],
  loader: () => fetchHeavyAnalyticsData(),
  component: AnalyticsPage,
})
```

## Context

- `autoCodeSplitting` eliminates the need for `.lazy.tsx` files entirely
- Do NOT export component functions from route files — exported values are included in the main bundle and bypass code splitting
- Default split groups: `['component']`, `['errorComponent']`, `['notFoundComponent']` — each gets its own chunk
- Configuration precedence: per-route `codeSplitGroupings` > programmatic `splitBehavior` > global `defaultBehavior`
- The `__root.tsx` route does not support code splitting
- Works with Vite, Rspack/Rsbuild, Webpack, and Esbuild
