# nav-relative-paths: Understand Relative Path Navigation

## Priority: MEDIUM

## Explanation

TanStack Router navigation uses `from` (origin) and `to` (destination). When `from` is omitted, the router assumes the root `/` and only auto-completes absolute paths. Always pass `from` for relative navigation so TypeScript knows the current route and can validate the target.

## Bad Example

```tsx
// No 'from' — relative navigation has no context
function PostActions() {
  return (
    <div>
      {/* to=".." without from — TypeScript doesn't know the current route */}
      <Link to="..">Back to Posts</Link>

      {/* Interpolating params into path — loses type safety */}
      <Link to={`/posts/${postId}/edit`}>Edit</Link>
    </div>
  )
}
```

## Good Example

```tsx
function PostActions() {
  return (
    <div>
      {/* to="." reloads the current route / from path */}
      <Link from="/posts/$postId" to=".">
        Refresh
      </Link>

      {/* to=".." navigates to the first parent route */}
      <Link from="/posts/$postId" to="..">
        Back to Posts
      </Link>

      {/* Relative navigation to a sibling */}
      <Link from="/posts/$postId" to="./edit">
        Edit Post
      </Link>

      {/* Absolute navigation with type-safe params */}
      <Link to="/posts/$postId/edit" params={{ postId: '123' }}>
        Edit Post
      </Link>
    </div>
  )
}
```

## Good Example: Using Route.fullPath

```tsx
// Route.fullPath provides the correct 'from' for the current route
function PostPage() {
  return (
    <div>
      <Link from={Route.fullPath} to="..">
        Back to Posts
      </Link>

      <Link from={Route.fullPath} to="." search={(prev) => ({ ...prev, tab: 'comments' })}>
        Comments Tab
      </Link>
    </div>
  )
}
```

## Good Example: useNavigate with from

```tsx
function PostPage() {
  // 'from' set at hook level — all navigate calls are relative to this route
  const navigate = useNavigate({ from: '/posts/$postId' })

  const handleDelete = async () => {
    await deletePost()
    navigate({ to: '..' }) // Go to /posts
  }

  const handleDuplicate = async () => {
    const newPost = await duplicatePost()
    navigate({ to: '.', params: { postId: newPost.id } }) // Stay on same route, different param
  }
}
```

## Special Relative Paths

| Path | Meaning |
|---|---|
| `to="."` | Current route / `from` path |
| `to=".."` | First parent route of `from` |

## Context

- Always pass `from` for relative navigation — without it, TypeScript can't validate the target
- Use `Route.fullPath` or literal route path strings for `from`
- Do NOT interpolate path params, search, or hash into the `to` string — use `params`, `search`, `hash` options
- In pathless layout routes, `Route.useNavigate()` resolves relative paths from the layout's parent route
- `from` can also be a route object import: `from={postRoute.fullPath}`
