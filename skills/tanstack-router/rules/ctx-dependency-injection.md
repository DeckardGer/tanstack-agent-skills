# ctx-dependency-injection: Use Context for Dependency Injection

## Priority: LOW

## Explanation

Router context enables dependency injection — passing services, clients, and configuration to routes without direct imports. This makes routes testable, decouples them from specific implementations, and centralizes shared dependencies like `QueryClient`, auth state, or API clients. Pass dependencies via `createRouter({ context })` and access them in loaders and `beforeLoad`.

## Bad Example

```tsx
// Direct imports in every route — tightly coupled, hard to test
// routes/posts.tsx
import { queryClient } from '~/lib/query-client' // Singleton import
import { apiClient } from '~/lib/api'            // Hard to mock in tests

export const Route = createFileRoute('/posts')({
  loader: async () => {
    const posts = await apiClient.getPosts()      // Can't swap implementation
    await queryClient.ensureQueryData(postsQuery)  // Tight coupling to singleton
    return posts
  },
})
```

## Good Example

```tsx
// routes/__root.tsx — define the context shape
import { createRootRouteWithContext } from '@tanstack/react-router'
import type { QueryClient } from '@tanstack/react-query'

interface RouterContext {
  queryClient: QueryClient
  apiClient: ApiClient
}

export const Route = createRootRouteWithContext<RouterContext>()({
  component: RootLayout,
})

// router.tsx — inject dependencies
import { createRouter } from '@tanstack/react-router'
import { QueryClient } from '@tanstack/react-query'
import { ApiClient } from '~/lib/api'

const queryClient = new QueryClient()
const apiClient = new ApiClient({ baseUrl: '/api' })

const router = createRouter({
  routeTree,
  context: { queryClient, apiClient },
})

// routes/posts.tsx — use injected dependencies
export const Route = createFileRoute('/posts')({
  loader: ({ context: { queryClient, apiClient } }) => {
    // Dependencies come from context — no direct imports
    return queryClient.ensureQueryData({
      queryKey: ['posts'],
      queryFn: () => apiClient.getPosts(),
    })
  },
})
```

## Good Example: Passing React Hook Values

```tsx
// You can't call hooks in loaders — pass hook values through context
// router.tsx
export const router = createRouter({
  routeTree,
  context: {
    queryClient,
    networkStrength: undefined!, // Will be set by React
  },
})

// main.tsx — provide live hook values
function App() {
  const networkStrength = useNetworkStrength()
  return <RouterProvider router={router} context={{ networkStrength }} />
}

// routes/posts.tsx — access hook value in loader
export const Route = createFileRoute('/posts')({
  loader: ({ context: { networkStrength } }) => {
    if (networkStrength === 'STRONG') {
      return fetchPostsWithImages()
    }
    return fetchPostsLite()
  },
})
```

## Good Example: Invalidating on Context Change

```tsx
function AuthProvider({ children }: { children: React.ReactNode }) {
  const router = useRouter()
  const [user, setUser] = useState<User | null>(null)

  useEffect(() => {
    const unsubscribe = auth.onAuthStateChanged((user) => {
      setUser(user)
      // Invalidate router when auth state changes
      // Forces re-evaluation of beforeLoad and loaders
      router.invalidate()
    })
    return unsubscribe
  }, [router])

  return <RouterProvider router={router} context={{ user }} />
}
```

## Context

- Only include properties in the root context type that are passed directly to `createRouter`
- Context added via `beforeLoad` in child routes is automatically inferred — no need to include it in the root type
- Call `router.invalidate()` when context values change (auth state, preferences) to force re-evaluation
- `RouterProvider`'s `context` prop overrides values from `createRouter`'s `context` — use this for live React values
- TypeScript enforces that all required context properties are provided
- For SSR, create dependencies inside `createRouter` factory to ensure per-request isolation
