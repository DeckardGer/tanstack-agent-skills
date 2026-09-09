# org-index-routes: Understand Index vs Layout Routes

## Priority: CRITICAL

## Explanation

Index routes match when their parent route's URL is matched exactly and no child route matches. They render inside the parent's `<Outlet />` as the default content. Without an index route, navigating to a parent URL shows nothing in the outlet. Confusing index routes with layout routes leads to empty pages or incorrect component hierarchy.

## Bad Example

```tsx
// Missing index route — /posts shows empty outlet
routes/
├── posts.tsx              # /posts layout with sidebar
├── posts.$postId.tsx      # /posts/:postId

// When user visits /posts, the layout renders but Outlet is empty
// No default content like "Select a post" is shown
```

```tsx
// Putting default content in the layout instead of an index route
// routes/posts.tsx
export const Route = createFileRoute('/posts')({
  component: PostsLayout,
})

function PostsLayout() {
  const params = Route.useParams()
  return (
    <div>
      <PostsSidebar />
      {/* Mixing layout and index concerns — breaks when children render */}
      {!params.postId && <p>Select a post</p>}
      <Outlet />
    </div>
  )
}
```

## Good Example

```
routes/
├── posts.tsx              # /posts layout (always renders)
├── posts.index.tsx        # /posts exact (default view)
├── posts.$postId.tsx      # /posts/:postId
```

```tsx
// routes/posts.tsx — pure layout, always renders
import { createFileRoute, Outlet } from '@tanstack/react-router'

export const Route = createFileRoute('/posts')({
  component: PostsLayout,
})

function PostsLayout() {
  return (
    <div className="flex">
      <PostsSidebar />
      <main>
        <Outlet /> {/* Index or child route renders here */}
      </main>
    </div>
  )
}

// routes/posts.index.tsx — shows when at /posts exactly
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/')({
  component: PostsIndex,
})

function PostsIndex() {
  return <p>Select a post from the sidebar to get started.</p>
}
```

## File Naming

```
# Flat style
posts.index.tsx          # Index route for /posts

# Directory style
posts/index.tsx          # Same thing

# In createFileRoute, use a trailing slash for index routes
createFileRoute('/posts/')
```

## Context

- Index routes use the `index` filename token (or trailing `/` in `createFileRoute`)
- An index route only matches when the parent URL is matched exactly with no further path segments
- Layout routes always render and provide `<Outlet />` — index routes fill that outlet at the exact parent path
- Every layout route with child routes should have a corresponding index route unless the parent path is never visited directly
- The root index route (`routes/index.tsx`) matches `/` exactly
