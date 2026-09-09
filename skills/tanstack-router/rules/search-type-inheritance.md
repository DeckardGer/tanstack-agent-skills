# search-type-inheritance: Leverage Parent Search Param Types

## Priority: HIGH

## Explanation

Search params defined on a parent route are automatically inherited by all child routes. Children can read parent search params with full type safety. Use this for shared state like pagination, filters, or theme settings that apply across a route subtree. Define global search params on the root route.

## Bad Example

```tsx
// Duplicating search param validation in every child route
// routes/posts.tsx
export const Route = createFileRoute('/posts')({
  validateSearch: z.object({ view: z.enum(['grid', 'list']).catch('grid') }),
})

// routes/posts.$postId.tsx
export const Route = createFileRoute('/posts/$postId')({
  // Re-declaring parent's search params — redundant, can go out of sync
  validateSearch: z.object({
    view: z.enum(['grid', 'list']).catch('grid'),
    tab: z.enum(['content', 'comments']).catch('content'),
  }),
})
```

## Good Example

```tsx
// routes/posts.tsx — defines shared search params once
export const Route = createFileRoute('/posts')({
  validateSearch: z.object({
    view: z.enum(['grid', 'list']).catch('grid'),
  }),
})

// routes/posts.$postId.tsx — only defines its own search params
export const Route = createFileRoute('/posts/$postId')({
  validateSearch: z.object({
    tab: z.enum(['content', 'comments']).catch('content'),
  }),
  component: PostPage,
})

function PostPage() {
  // Child automatically inherits parent search params
  const { tab } = Route.useSearch()     // tab from this route
  const { view } = useSearch({ from: '/posts' }) // view from parent

  return <div>View: {view}, Tab: {tab}</div>
}
```

## Good Example: Global Search Params on Root

```tsx
// routes/__root.tsx — search params available everywhere
export const Route = createRootRoute({
  validateSearch: z.object({
    theme: z.enum(['light', 'dark']).catch('light'),
    lang: z.enum(['en', 'es', 'fr']).catch('en'),
  }),
})

// Any route can access theme and lang
function AnyComponent() {
  const { theme, lang } = useSearch({ from: '__root__' })
  return <div>Theme: {theme}</div>
}
```

## Preserving Inherited Params During Navigation

```tsx
function PostFilters() {
  return (
    // WRONG — loses parent search params (view, theme, etc.)
    // <Link to="." search={{ tab: 'comments' }}>Comments</Link>

    // CORRECT — preserves parent params with functional update
    <Link to="." search={(prev) => ({ ...prev, tab: 'comments' })}>
      Comments
    </Link>
  )
}
```

## Context

- Search params flow down the route tree — parent → child
- Children don't need to re-validate parent search params
- Use `useSearch({ from: '/parent-path' })` to read a specific parent's search params with type safety
- Always use functional `search` updates (`(prev) => ({ ...prev, ... })`) to preserve inherited params when navigating
- Use `strict: false` on `useSearch` in shared components that don't know their route context
