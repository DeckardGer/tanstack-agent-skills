# mw-composability: Compose Middleware Effectively

## Priority: HIGH

## Explanation

Middleware can depend on other middleware via `.middleware([...])`, forming a composition chain. This lets you build layered concerns (logging → auth → permissions) without coupling them. Global middleware is configured in `src/start.ts` using `createStart()`.

## Bad Example

```tsx
// Monolithic middleware doing too many things
const doEverythingMiddleware = createMiddleware({ type: 'function' })
  .server(async ({ next, request }) => {
    // Logging
    const start = Date.now()
    console.log(`${request.method} ${request.url}`)

    // Auth
    const session = await useAppSession()
    if (!session.data.userId) throw redirect({ to: '/login' })
    const user = await db.users.findUnique({ where: { id: session.data.userId } })

    // Permissions
    const perms = await getPermissions(user.id)

    // Rate limiting
    const ip = request.headers.get('x-forwarded-for')
    if (await isRateLimited(ip)) throw new Error('Rate limited')

    console.log(`Completed in ${Date.now() - start}ms`)
    return next({ context: { user, perms } })
  })
```

## Good Example: Composable Middleware Chain

```tsx
import { createMiddleware } from '@tanstack/react-start'

// Each middleware has a single responsibility
const loggingMiddleware = createMiddleware({ type: 'function' })
  .server(async ({ next }) => {
    const start = Date.now()
    const result = await next()
    console.log(`Completed in ${Date.now() - start}ms`)
    return result
  })

const authMiddleware = createMiddleware({ type: 'function' })
  .middleware([loggingMiddleware])  // Logging wraps auth
  .server(async ({ next }) => {
    const session = await useAppSession()
    if (!session.data.userId) throw redirect({ to: '/login' })
    const user = await db.users.findUnique({
      where: { id: session.data.userId },
    })
    return next({ context: { user } })
  })

const adminMiddleware = createMiddleware({ type: 'function' })
  .middleware([authMiddleware])  // Auth wraps admin check
  .server(async ({ next, context }) => {
    if (context.user.role !== 'admin') {
      throw redirect({ to: '/unauthorized' })
    }
    return next()
  })

// Use the composed chain — logging + auth + admin all applied
export const deleteUser = createServerFn({ method: 'POST' })
  .middleware([adminMiddleware])
  .inputValidator(z.object({ userId: z.string() }))
  .handler(async ({ data, context }) => {
    await db.users.delete({ where: { id: data.userId } })
  })
```

## Good Example: Global Middleware in start.ts

```tsx
// src/start.ts
import { createStart, createMiddleware } from '@tanstack/react-start'

const requestLogger = createMiddleware().server(async ({ next, request }) => {
  console.log(`${request.method} ${request.url}`)
  return next()
})

const functionLogger = createMiddleware({ type: 'function' })
  .server(async ({ next }) => {
    const start = Date.now()
    const result = await next()
    console.log(`Server fn completed in ${Date.now() - start}ms`)
    return result
  })

export const startInstance = createStart(() => ({
  // Runs on ALL server requests (SSR, server routes, server functions)
  requestMiddleware: [requestLogger],
  // Runs on ALL server function calls
  functionMiddleware: [functionLogger],
}))
```

## Good Example: Middleware for Server Routes

```tsx
// Middleware on all handlers in a server route
export const Route = createFileRoute('/api/users')({
  server: {
    middleware: [authMiddleware],  // Applies to GET and POST
    handlers: {
      GET: async ({ context }) => {
        return Response.json(await db.users.findMany())
      },
      POST: async ({ request, context }) => {
        const body = await request.json()
        return Response.json(await db.users.create({ data: body }))
      },
    },
  },
})

// Middleware on specific handlers only
export const Route = createFileRoute('/api/posts')({
  server: {
    handlers: ({ createHandlers }) =>
      createHandlers({
        GET: async ({ request }) => {
          return Response.json(await db.posts.findMany())  // Public
        },
        POST: {
          middleware: [authMiddleware],  // Only POST requires auth
          handler: async ({ request, context }) => {
            const body = await request.json()
            return Response.json(await db.posts.create({ data: body }))
          },
        },
      }),
  },
})
```

## Context

- `.middleware([...])` on `createMiddleware` creates a dependency chain
- Execution order: outermost middleware wraps inner middleware wraps handler
- Global middleware in `src/start.ts`: `requestMiddleware` (all requests) and `functionMiddleware` (server functions)
- Route-level server middleware via `server.middleware` applies to all handlers
- Per-handler middleware via `createHandlers` applies to specific HTTP methods
- Keep each middleware focused on a single concern for reusability
