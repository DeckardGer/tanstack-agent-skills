# auth-server-functions: Verify Auth in Server Functions

## Priority: HIGH

## Explanation

Every server function that accesses user-specific data or performs mutations must verify authentication. Use middleware to centralize auth checks, or call your session utility directly. Never trust client-side auth state alone — always verify on the server.

## Bad Example

```tsx
// Trusting a userId passed from the client
export const getUserData = createServerFn()
  .inputValidator((data: { userId: string }) => data)
  .handler(async ({ data }) => {
    // Anyone can pass any userId! No server-side auth check
    return db.users.findUnique({ where: { id: data.userId } })
  })

// Checking auth on the client only
function ProfilePage() {
  const { user } = useAuth()  // Client-side only
  if (!user) return <Navigate to="/login" />
  // If someone calls getProfile() directly, there's no server check
  return <Profile />
}
```

## Good Example: Auth via Middleware

```tsx
import { createMiddleware, createServerFn } from '@tanstack/react-start'
import { useSession } from '@tanstack/react-start/server'
import { redirect } from '@tanstack/react-router'

const authMiddleware = createMiddleware({ type: 'function' })
  .server(async ({ next }) => {
    const session = await useAppSession()
    if (!session.data.userId) {
      throw redirect({ to: '/login' })
    }
    return next({
      context: { userId: session.data.userId },
    })
  })

// User can only access their own data - userId comes from session, not client
export const getMyProfile = createServerFn()
  .middleware([authMiddleware])
  .handler(async ({ context }) => {
    return db.users.findUnique({
      where: { id: context.userId },
      select: { id: true, name: true, email: true, avatar: true },
    })
  })

export const updateMyProfile = createServerFn({ method: 'POST' })
  .middleware([authMiddleware])
  .inputValidator(z.object({
    name: z.string().min(1).max(100),
  }))
  .handler(async ({ data, context }) => {
    return db.users.update({
      where: { id: context.userId },
      data: { name: data.name },
    })
  })
```

## Good Example: Direct Session Check

```tsx
import { createServerFn } from '@tanstack/react-start'
import { redirect } from '@tanstack/react-router'

export const getCurrentUser = createServerFn()
  .handler(async () => {
    const session = await useAppSession()
    const userId = session.data.userId

    if (!userId) return null

    return db.users.findUnique({
      where: { id: userId },
      select: {
        id: true,
        email: true,
        name: true,
        // Never include passwordHash or secrets
      },
    })
  })

// Use in root route to make user available everywhere
export const Route = createRootRoute({
  beforeLoad: async () => {
    const user = await getCurrentUser()
    return { user }
  },
})
```

## Context

- Always derive the current user from the server-side session, never from client input
- Use middleware for consistent auth across multiple server functions
- Use `beforeLoad` on layout routes to protect entire route trees
- Return `null` (not throw) from `getCurrentUser` when auth is optional
- Throw `redirect()` when auth is required and the user is missing
- Never return sensitive fields (password hashes, tokens) from auth server functions
