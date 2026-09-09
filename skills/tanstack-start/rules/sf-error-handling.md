# sf-error-handling: Handle Errors in Server Functions

## Priority: CRITICAL

## Explanation

Server functions can throw `redirect()` and `notFound()` from `@tanstack/react-router` for control flow, and regular errors for unexpected failures. These are automatically serialized across the server/client boundary. Proper error handling prevents unhandled rejections and provides good UX.

## Bad Example

```tsx
// Returning error objects instead of throwing - loses router integration
export const getPost = createServerFn()
  .inputValidator((data: { id: string }) => data)
  .handler(async ({ data }) => {
    const post = await db.posts.findUnique({ where: { id: data.id } })
    if (!post) {
      return { error: 'Not found' }  // Client must check manually
    }
    return { data: post }
  })

// Throwing generic errors instead of redirect
export const requireAuth = createServerFn()
  .handler(async () => {
    const user = await getCurrentUser()
    if (!user) {
      throw new Error('Not authenticated')  // No redirect, bad UX
    }
    return user
  })
```

## Good Example

```tsx
import { createServerFn } from '@tanstack/react-start'
import { redirect, notFound } from '@tanstack/react-router'

// Use notFound() for missing resources - triggers notFoundComponent
export const getPost = createServerFn()
  .inputValidator((data: { id: string }) => data)
  .handler(async ({ data }) => {
    const post = await db.posts.findUnique({ where: { id: data.id } })
    if (!post) {
      throw notFound()
    }
    return post
  })

// Use redirect() for auth failures - navigates to login
export const requireAuth = createServerFn()
  .handler(async () => {
    const user = await getCurrentUser()
    if (!user) {
      throw redirect({ to: '/login' })
    }
    return user
  })

// Use redirect() with search params to preserve return URL
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
```

## Good Example: Error Handling with isRedirect

```tsx
import { redirect, isRedirect } from '@tanstack/react-router'

export const performAction = createServerFn({ method: 'POST' })
  .inputValidator((data: { action: string }) => data)
  .handler(async ({ data }) => {
    try {
      const result = await riskyOperation(data.action)
      return result
    } catch (error) {
      // Re-throw redirects and notFound - they are control flow, not errors
      if (isRedirect(error)) throw error
      // Handle actual errors
      console.error('Action failed:', error)
      throw new Error('Failed to perform action')
    }
  })
```

## Context

- `throw redirect()` triggers client-side navigation
- `throw notFound()` triggers the nearest `notFoundComponent`
- Regular thrown errors trigger the nearest `errorComponent`
- Use `isRedirect()` to distinguish redirects from errors in try/catch blocks
- Validation errors from `.inputValidator()` are automatically serialized
- All imports: `redirect`, `notFound`, `isRedirect` from `@tanstack/react-router`
