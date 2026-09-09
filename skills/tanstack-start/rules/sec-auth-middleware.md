# sec-auth-middleware: Protect Routes with Auth Middleware

## Priority: CRITICAL

## Explanation

Use middleware to enforce authentication and authorization consistently across routes and server functions. Centralizing auth checks in middleware prevents accidentally leaving endpoints unprotected.

## Bad Example

```tsx
// Checking auth in every handler - easy to forget one
export const getProfile = createServerFn()
  .handler(async () => {
    const session = await useAppSession()
    if (!session.data.userId) throw redirect({ to: '/login' })
    return db.users.findUnique({ where: { id: session.data.userId } })
  })

export const updateProfile = createServerFn({ method: 'POST' })
  .handler(async ({ data }) => {
    const session = await useAppSession()
    if (!session.data.userId) throw redirect({ to: '/login' })
    // ... duplicate check in every function
  })

export const getSettings = createServerFn()
  .handler(async () => {
    // Forgot the auth check here! Endpoint is unprotected
    return db.settings.findFirst()
  })
```

## Good Example: Auth Middleware for Server Functions

```tsx
import { createMiddleware, createServerFn } from '@tanstack/react-start'
import { useSession } from '@tanstack/react-start/server'
import { redirect } from '@tanstack/react-router'

// Reusable auth middleware
const authMiddleware = createMiddleware({ type: 'function' })
  .server(async ({ next }) => {
    const session = await useAppSession()
    if (!session.data.userId) {
      throw redirect({ to: '/login' })
    }

    const user = await db.users.findUnique({
      where: { id: session.data.userId },
    })
    if (!user) {
      throw redirect({ to: '/login' })
    }

    return next({
      context: { user },
    })
  })

// All protected functions use the middleware - auth is guaranteed
export const getProfile = createServerFn()
  .middleware([authMiddleware])
  .handler(async ({ context }) => {
    return context.user  // User is guaranteed to exist
  })

export const updateProfile = createServerFn({ method: 'POST' })
  .middleware([authMiddleware])
  .inputValidator(updateProfileSchema)
  .handler(async ({ data, context }) => {
    return db.users.update({
      where: { id: context.user.id },
      data,
    })
  })
```

## Good Example: Route Protection with beforeLoad

```tsx
// routes/_authed.tsx — pathless layout route protects all children
import { createFileRoute, redirect } from '@tanstack/react-router'

export const Route = createFileRoute('/_authed')({
  beforeLoad: async ({ location }) => {
    const user = await getCurrentUser()
    if (!user) {
      throw redirect({
        to: '/login',
        search: { redirect: location.href },
      })
    }
    return { user }
  },
})

// routes/_authed/dashboard.tsx — automatically protected
export const Route = createFileRoute('/_authed/dashboard')({
  component: DashboardComponent,
})

function DashboardComponent() {
  const { user } = Route.useRouteContext()
  return <h1>Welcome, {user.email}!</h1>
}
```

## Good Example: Role-Based Middleware

```tsx
const adminMiddleware = createMiddleware({ type: 'function' })
  .middleware([authMiddleware])  // Chains on auth middleware
  .server(async ({ next, context }) => {
    if (context.user.role !== 'admin') {
      throw redirect({ to: '/unauthorized' })
    }
    return next()
  })

export const deleteUser = createServerFn({ method: 'POST' })
  .middleware([adminMiddleware])
  .inputValidator(z.object({ userId: z.string() }))
  .handler(async ({ data }) => {
    await db.users.delete({ where: { id: data.userId } })
  })
```

## Context

- Use `createMiddleware({ type: 'function' })` for server function middleware
- Use `beforeLoad` on pathless layout routes (e.g., `_authed.tsx`) for route protection
- Middleware can compose: admin middleware can chain on auth middleware
- Context from middleware is type-safe in handlers
- `beforeLoad` runs serially (parent before children) so parent auth protects all children
