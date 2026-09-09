# nav-use-navigate: Use useNavigate for Programmatic Navigation

## Priority: MEDIUM

## Explanation

Use `useNavigate()` for programmatic navigation triggered by side effects — form submissions, successful mutations, or conditional logic. Prefer `<Link>` for anything the user directly clicks. Pass `from` to `useNavigate()` at the hook level for type-safe relative navigation.

## Bad Example

```tsx
// Using window.location or manual history manipulation
function CreatePostForm() {
  const handleSubmit = async (data: PostData) => {
    const post = await createPost(data)
    window.location.href = `/posts/${post.id}` // Full page reload, loses state
  }
  return <form onSubmit={handleSubmit}>{/* ... */}</form>
}

// Using Link for side-effect-driven navigation
function CreatePostForm() {
  const [newPostId, setNewPostId] = useState<string | null>(null)
  // Awkward: rendering a hidden Link and clicking it programmatically
  return newPostId ? <Navigate to="/posts/$postId" params={{ postId: newPostId }} /> : null
}
```

## Good Example

```tsx
import { useNavigate } from '@tanstack/react-router'

function CreatePostForm() {
  // Pass 'from' at the hook level for type safety
  const navigate = useNavigate({ from: '/posts/new' })

  const handleSubmit = async (data: PostData) => {
    const post = await createPost(data)

    navigate({
      to: '/posts/$postId',
      params: { postId: post.id },
    })
  }

  return <form onSubmit={handleSubmit}>{/* ... */}</form>
}
```

## Good Example: With Search Params and Replace

```tsx
function LoginForm() {
  const navigate = useNavigate()

  const handleLogin = async (credentials: Credentials) => {
    await authenticate(credentials)

    navigate({
      to: '/dashboard',
      replace: true,        // Replace login page in history
      resetScroll: true,    // Scroll to top
    })
  }

  return <form onSubmit={handleLogin}>{/* ... */}</form>
}

function FilterButton({ sort }: { sort: string }) {
  const navigate = useNavigate({ from: '/products' })

  return (
    <button
      onClick={() =>
        navigate({
          search: (prev) => ({ ...prev, sort, page: 1 }),
        })
      }
    >
      Sort by {sort}
    </button>
  )
}
```

## Good Example: Route-Scoped useNavigate

```tsx
// Route.useNavigate() has 'from' pre-set to the current route
function PostActions() {
  const navigate = Route.useNavigate()

  const handleDelete = async () => {
    await deletePost()
    navigate({ to: '..', })  // Navigate to parent route
  }

  return <button onClick={handleDelete}>Delete</button>
}
```

## NavigateOptions Reference

| Option | Description |
|---|---|
| `to` | Destination path (absolute or relative from `from`) |
| `params` | Path params object or updater function |
| `search` | Search params object or updater function |
| `hash` | Hash string or updater function |
| `replace` | Replace current history entry instead of pushing |
| `resetScroll` | Reset scroll position to 0,0 |
| `viewTransition` | Use View Transitions API |

## Context

- Prefer `<Link>` for user-clickable elements — it renders a real `<a>` with `href`, supports cmd/ctrl+click, and tracks active state
- Use `useNavigate` for side effects: form submissions, auth flows, conditional redirects
- Pass `from` to the hook (not each `navigate()` call) for type-safe relative paths
- `Route.useNavigate()` automatically sets `from` to the current route
- Do not use `router.navigate()` inside React components — use the hook instead
- `<Navigate>` component is for immediate redirects on mount (renders nothing)
