# mw-function-middleware: Use Function Middleware for Server Functions

## Priority: HIGH

## Explanation

Server function middleware (`type: 'function'`) runs specifically around server function calls and supports both client-side and server-side hooks. Unlike request middleware, function middleware can intercept calls on the client before they reach the server, add custom headers, validate inputs, and transform responses.

## Bad Example

```tsx
// Using request middleware for function-specific concerns
// This runs on ALL requests, not just the server functions that need it
const loggingMiddleware = createMiddleware()
  .server(async ({ next }) => {
    console.log('Request started')
    return next()
  })

// Or duplicating logic in every handler
export const getData = createServerFn()
  .handler(async () => {
    console.log('Function called')  // Repeated everywhere
    const start = Date.now()
    const result = await fetchData()
    console.log(`Took ${Date.now() - start}ms`)
    return result
  })
```

## Good Example

```tsx
import { createMiddleware, createServerFn } from '@tanstack/react-start'

// Function middleware with both client and server hooks
const loggingMiddleware = createMiddleware({ type: 'function' })
  .client(async ({ next }) => {
    console.log('Client: calling server function')
    const start = Date.now()
    const result = await next()
    console.log(`Client: response in ${Date.now() - start}ms`)
    return result
  })
  .server(async ({ next }) => {
    console.log('Server: handling function call')
    const result = await next()
    console.log('Server: function complete')
    return result
  })

export const getData = createServerFn()
  .middleware([loggingMiddleware])
  .handler(async () => {
    return fetchData()
  })
```

## Good Example: Auth Token Middleware

```tsx
// Attach auth headers from client-side token
const authTokenMiddleware = createMiddleware({ type: 'function' })
  .client(async ({ next }) => {
    const token = getAuthToken()  // From client-side auth provider
    return next({
      headers: {
        Authorization: `Bearer ${token}`,
      },
    })
  })
  .server(async ({ next }) => {
    const authHeader = getRequestHeader('Authorization')
    const user = await verifyToken(authHeader)
    return next({ context: { user } })
  })

export const getProfile = createServerFn()
  .middleware([authTokenMiddleware])
  .handler(async ({ context }) => {
    return db.users.findUnique({ where: { id: context.user.id } })
  })
```

## Good Example: Input Validation in Middleware

```tsx
import { zodValidator } from '@tanstack/zod-adapter'
import { z } from 'zod'

// Middleware that validates common input patterns
const workspaceMiddleware = createMiddleware({ type: 'function' })
  .inputValidator(zodValidator(z.object({
    workspaceId: z.string().uuid(),
  })))
  .server(async ({ next, data }) => {
    const workspace = await db.workspaces.findUnique({
      where: { id: data.workspaceId },
    })
    if (!workspace) throw new Error('Workspace not found')
    return next({ context: { workspace } })
  })

export const getWorkspaceMembers = createServerFn()
  .middleware([workspaceMiddleware])
  .handler(async ({ context }) => {
    return db.members.findMany({
      where: { workspaceId: context.workspace.id },
    })
  })
```

## Context

- `type: 'function'` distinguishes function middleware from request middleware
- `.client()` runs on the client before the RPC call — use for headers, tokens, logging
- `.server()` runs on the server when handling the call — use for auth, context, validation
- `.inputValidator()` is only available on function middleware, not request middleware
- Use `@tanstack/zod-adapter`'s `zodValidator()` for middleware input validation
- Function middleware can send custom headers via `next({ headers: {...} })`
- Request middleware (`createMiddleware()` without `type`) runs on ALL server requests
