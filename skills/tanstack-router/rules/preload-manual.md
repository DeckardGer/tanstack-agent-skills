# preload-manual: Use Manual Preloading Strategically

## Priority: MEDIUM

## Explanation

Beyond automatic intent-based preloading on `<Link>` hover, you can manually preload routes programmatically with `router.preloadRoute()`. Use this for predictive preloading — loading the next likely route based on user behavior, scroll position, or application logic. You can also preload only the JS chunks without data using `router.loadRouteChunk()`.

## Bad Example

```tsx
// Preloading everything eagerly on mount — wastes bandwidth
function App() {
  const router = useRouter()

  useEffect(() => {
    // Loading ALL routes on app init is wasteful
    router.preloadRoute({ to: '/dashboard' })
    router.preloadRoute({ to: '/settings' })
    router.preloadRoute({ to: '/profile' })
    router.preloadRoute({ to: '/posts' })
    router.preloadRoute({ to: '/analytics' })
    // ...every route in the app
  }, [])
}
```

## Good Example: Predictive Preloading

```tsx
function WizardStep2() {
  const router = useRouter()

  useEffect(() => {
    // User is on step 2 — preload step 3 (most likely next action)
    router.preloadRoute({ to: '/wizard/step-3' })
  }, [router])

  return <div>{/* Step 2 content */}</div>
}
```

## Good Example: After Data Fetch

```tsx
function PostList() {
  const router = useRouter()
  const posts = Route.useLoaderData()

  useEffect(() => {
    // Preload the first few posts that the user is likely to click
    posts.slice(0, 3).forEach((post) => {
      router.preloadRoute({
        to: '/posts/$postId',
        params: { postId: post.id },
      })
    })
  }, [router, posts])

  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>
          <Link to="/posts/$postId" params={{ postId: post.id }}>
            {post.title}
          </Link>
        </li>
      ))}
    </ul>
  )
}
```

## Good Example: Preload Only JS Chunks

```tsx
function App() {
  const router = useRouter()

  useEffect(() => {
    // Preload only the code chunks (no data fetching)
    // Useful for routes behind auth or feature flags
    Promise.all([
      router.loadRouteChunk(router.routesByPath['/dashboard']),
      router.loadRouteChunk(router.routesByPath['/settings']),
    ]).catch(() => {
      // Failed to preload chunks — non-critical, ignore
    })
  }, [router])
}
```

## Preloading API

```tsx
// Preload full route (data + code chunks)
const matches = await router.preloadRoute({
  to: '/posts/$postId',
  params: { postId: '123' },
})

// Preload only the JS chunk (no loader execution)
await router.loadRouteChunk(router.routesByPath['/posts'])
```

## Context

- `router.preloadRoute()` loads both JS chunks and route data (runs the loader)
- `router.loadRouteChunk()` loads only the JS chunk — useful for warming the code cache without fetching data
- Manual preloading respects `preloadStaleTime` — it won't refetch if data is still fresh
- Wrap in try/catch — preloading failures are non-critical and shouldn't break the app
- Use for wizard flows, multi-step forms, or when you can predict the user's next action
- Combine with automatic `defaultPreload: 'intent'` — manual preloading supplements intent-based preloading
