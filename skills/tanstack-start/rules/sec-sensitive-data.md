# sec-sensitive-data: Keep Secrets Server-Side Only

## Priority: CRITICAL

## Explanation

Route loaders are isomorphic — they run on both the server (SSR) and client (navigation). Code in loaders is included in the client bundle. Never access secrets, database connections, or private API keys in loaders. Use `createServerFn()` to keep sensitive logic server-only.

## Bad Example

```tsx
// DANGEROUS: loader runs on both server and client
// process.env.SECRET is bundled into client JavaScript
export const Route = createFileRoute('/users')({
  loader: () => {
    const secret = process.env.DATABASE_URL  // EXPOSED in client bundle!
    return fetch(`${process.env.API_URL}/users`, {
      headers: { Authorization: `Bearer ${process.env.API_SECRET}` },
    })
  },
})

// DANGEROUS: importing server modules in route files
import { db } from './db.server'  // Database driver bundled into client!

export const Route = createFileRoute('/posts')({
  loader: () => db.posts.findMany(),
})
```

## Good Example

```tsx
import { createServerFn } from '@tanstack/react-start'

// Server function - code only runs on server, replaced with RPC stub on client
const getUsers = createServerFn().handler(async () => {
  // Safe: this code never reaches the client bundle
  const secret = process.env.API_SECRET
  const response = await fetch(`${process.env.API_URL}/users`, {
    headers: { Authorization: `Bearer ${secret}` },
  })
  return response.json()
})

export const Route = createFileRoute('/users')({
  loader: () => getUsers(),  // Client calls this as an RPC
})
```

## Good Example: Environment Variable Boundaries

```tsx
// Client-safe: only VITE_ prefixed vars are available
function AppHeader() {
  return <h1>{import.meta.env.VITE_APP_NAME}</h1>
}

// Server-only: access any env var inside server functions
const getConfig = createServerFn().handler(async () => {
  return {
    // Only return what the client needs — never return secrets
    appName: process.env.VITE_APP_NAME,
    featureFlags: JSON.parse(process.env.FEATURE_FLAGS ?? '{}'),
  }
})

// For truly server-only utility functions
import { createServerOnlyFn } from '@tanstack/react-start'

const getDbConnection = createServerOnlyFn(() => {
  return connect(process.env.DATABASE_URL)
})
// Calling getDbConnection() on the client throws an error
```

## Good Example: File Organization for Server/Client Separation

```
src/utils/
  users.functions.ts   # createServerFn wrappers — safe to import anywhere
  users.server.ts      # Direct DB access, internal logic — server-only
  schemas.ts           # Zod schemas, types — safe for both
```

```tsx
// users.functions.ts — safe to import in components
import { createServerFn } from '@tanstack/react-start'

export const getUsers = createServerFn().handler(async () => {
  // Import server-only code inside the handler
  const { db } = await import('./users.server')
  return db.users.findMany()
})
```

## Context

- Route `loader`, `beforeLoad`, and `component` code is isomorphic (runs on server AND client)
- `createServerFn()` handlers only run on the server — client gets an RPC stub
- `createServerOnlyFn(fn)` throws if accidentally called on the client
- `createClientOnlyFn(fn)` throws if accidentally called on the server
- Only `VITE_`-prefixed environment variables are available on the client via `import.meta.env`
- All `process.env` variables are available inside server functions
- Static imports of server function files are safe — the bundler handles code splitting
