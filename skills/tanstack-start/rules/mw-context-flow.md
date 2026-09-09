# mw-context-flow: Properly Pass Context Through Middleware

## Priority: HIGH

## Explanation

Middleware communicates with downstream handlers and other middleware through context. Use `next({ context: {...} })` to pass data down the chain. Client-to-server context requires explicit `sendContext`. Context is fully type-safe — TypeScript infers what's available in each handler.

## Bad Example

```tsx
// Using global state instead of context
let currentUser: User | null = null

const authMiddleware = createMiddleware({ type: 'function' })
  .server(async ({ next }) => {
    currentUser = await getUser()  // Global mutable state - race conditions!
    return next()
  })

export const getProfile = createServerFn()
  .middleware([authMiddleware])
  .handler(async () => {
    return currentUser  // May be wrong user in concurrent requests
  })
```

## Good Example: Server Context

```tsx
import { createMiddleware, createServerFn } from '@tanstack/react-start'

const authMiddleware = createMiddleware({ type: 'function' })
  .server(async ({ next }) => {
    const user = await getCurrentUser()
    // Pass through context — type-safe and request-scoped
    return next({
      context: { user },
    })
  })

const permissionMiddleware = createMiddleware({ type: 'function' })
  .middleware([authMiddleware])
  .server(async ({ next, context }) => {
    // context.user is available from authMiddleware
    const permissions = await getPermissions(context.user.id)
    return next({
      context: { permissions },  // Adds to existing context
    })
  })

export const getAdminData = createServerFn()
  .middleware([permissionMiddleware])
  .handler(async ({ context }) => {
    // Both user and permissions are available and typed
    if (!context.permissions.includes('admin')) {
      throw new Error('Forbidden')
    }
    return fetchAdminData()
  })
```

## Good Example: Client-to-Server Context with sendContext

```tsx
// Client context is NOT sent to server by default — use sendContext
const workspaceMiddleware = createMiddleware({ type: 'function' })
  .client(async ({ next, context }) => {
    // Send specific data from client to server
    return next({
      sendContext: {
        workspaceId: context.workspaceId,
        locale: navigator.language,
      },
    })
  })
  .server(async ({ next, context }) => {
    // context.workspaceId is available here from sendContext
    console.log('Workspace:', context.workspaceId)
    console.log('Locale:', context.locale)
    return next()
  })
```

## Good Example: Server-to-Client Context with sendContext

```tsx
const timerMiddleware = createMiddleware({ type: 'function' })
  .server(async ({ next }) => {
    const start = Date.now()
    const result = await next({
      sendContext: {
        serverTimestamp: new Date().toISOString(),
      },
    })
    return result
  })
  .client(async ({ next, context }) => {
    const result = await next()
    // context.serverTimestamp is available after server responds
    console.log('Server time:', context.serverTimestamp)
    return result
  })
```

## Context

- `next({ context: {...} })` passes data to downstream middleware/handlers (server-side only)
- `sendContext` is needed to cross the client/server boundary in either direction
- Client context is NOT sent to the server by default — only `sendContext` fields are transmitted
- `sendContext` data is type-safe but NOT runtime-validated on the server — validate if it contains user input
- Context accumulates through the middleware chain — each middleware can add to it
- Context is request-scoped, avoiding race conditions from global state
