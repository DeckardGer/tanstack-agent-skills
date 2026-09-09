# err-redirects: Use Redirects Appropriately

## Priority: MEDIUM

## Explanation

TanStack Start uses `throw redirect()` for server-side navigation control. Redirects work in server functions, `beforeLoad`, and loaders. They trigger client-side navigation and are distinct from errors — use `isRedirect()` to distinguish them in try/catch blocks.

## Bad Example

```tsx
// Returning a redirect URL instead of throwing - client must handle manually
export const checkAuth = createServerFn()
  .handler(async () => {
    const user = await getCurrentUser()
    if (!user) {
      return { redirect: '/login' }  // Client must check and navigate
    }
    return { user }
  })

// Catching redirects as errors
export const Route = createFileRoute('/_authed')({
  beforeLoad: async () => {
    try {
      return await checkAuth()
    } catch (error) {
      // This catches the redirect too! It never navigates
      console.error('Auth failed:', error)
      return { user: null }
    }
  },
})
```

## Good Example

```tsx
import { redirect, isRedirect } from '@tanstack/react-router'
import { createServerFn } from '@tanstack/react-start'

// Throw redirect for navigation control
export const requireAuth = createServerFn()
  .handler(async () => {
    const user = await getCurrentUser()
    if (!user) {
      throw redirect({ to: '/login' })
    }
    return user
  })

// Redirect with search params to preserve return URL
export const requireAuthWithReturn = createServerFn()
  .handler(async () => {
    const request = getRequest()
    const user = await getCurrentUser()
    if (!user) {
      throw redirect({
        to: '/login',
        search: { redirect: new URL(request.url).pathname },
      })
    }
    return user
  })

// Redirect after mutation
export const createPost = createServerFn({ method: 'POST' })
  .inputValidator(createPostSchema)
  .handler(async ({ data }) => {
    const post = await db.posts.create({ data })
    throw redirect({ to: '/posts/$postId', params: { postId: post.id } })
  })
```

## Good Example: Handling Redirects in try/catch

```tsx
import { redirect, isRedirect } from '@tanstack/react-router'

export const Route = createFileRoute('/_authed')({
  beforeLoad: async ({ location }) => {
    try {
      const user = await verifySession()
      if (!user) {
        throw redirect({
          to: '/login',
          search: { redirect: location.href },
        })
      }
      return { user }
    } catch (error) {
      // Always re-throw redirects — they are control flow, not errors
      if (isRedirect(error)) throw error
      // Handle actual errors
      throw redirect({
        to: '/login',
        search: { redirect: location.href },
      })
    }
  },
})
```

## Good Example: External Redirects

```tsx
// Use href for external URLs (not to)
export const startOAuth = createServerFn({ method: 'POST' })
  .handler(async () => {
    const authUrl = buildOAuthUrl()
    throw redirect({ href: authUrl })  // href for full URLs
  })
```

## Context

- `throw redirect({ to: '/path' })` for internal navigation (type-safe route paths)
- `throw redirect({ href: 'https://...' })` for external URLs
- `throw redirect({ to: '/path', search: { key: 'value' } })` to include search params
- Redirects work in: server functions, `beforeLoad`, loaders
- Always use `isRedirect(error)` in try/catch to re-throw redirects
- Redirect is from `@tanstack/react-router`, not `@tanstack/react-start`
- After a `throw redirect()`, no further code in the handler executes
