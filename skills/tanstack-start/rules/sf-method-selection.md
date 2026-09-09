# sf-method-selection: Choose Appropriate HTTP Method

## Priority: CRITICAL

## Explanation

Server functions support `GET` (default) and `POST` methods. Use `GET` for data fetching (idempotent, cacheable) and `POST` for mutations that change data. Choosing the wrong method can cause caching issues, duplicate mutations, or unexpected behavior during prefetching.

## Bad Example

```tsx
// Using POST for a read operation - prevents caching and prefetching
export const getUsers = createServerFn({ method: 'POST' })
  .handler(async () => {
    return db.users.findMany()
  })

// Using GET for a mutation - can be replayed by browser/cache
export const deleteUser = createServerFn({ method: 'GET' })
  .inputValidator((data: { id: string }) => data)
  .handler(async ({ data }) => {
    await db.users.delete({ where: { id: data.id } })
    return { success: true }
  })
```

## Good Example

```tsx
import { createServerFn } from '@tanstack/react-start'

// GET for reads - idempotent, cacheable, safe for prefetching
export const getUsers = createServerFn()  // GET is default
  .handler(async () => {
    return db.users.findMany({ orderBy: { createdAt: 'desc' } })
  })

export const getUser = createServerFn()
  .inputValidator((data: { id: string }) => data)
  .handler(async ({ data }) => {
    return db.users.findUnique({ where: { id: data.id } })
  })

// POST for mutations - not cached, not replayed
export const createUser = createServerFn({ method: 'POST' })
  .inputValidator((data: { name: string; email: string }) => data)
  .handler(async ({ data }) => {
    return db.users.create({ data })
  })

export const updateUser = createServerFn({ method: 'POST' })
  .inputValidator((data: { id: string; name: string }) => data)
  .handler(async ({ data }) => {
    return db.users.update({
      where: { id: data.id },
      data: { name: data.name },
    })
  })

export const deleteUser = createServerFn({ method: 'POST' })
  .inputValidator((data: { id: string }) => data)
  .handler(async ({ data }) => {
    await db.users.delete({ where: { id: data.id } })
    return { success: true }
  })
```

## Context

- `GET` is the default when no method is specified
- `GET` functions can be called during route preloading and prefetching
- `POST` functions are never automatically called by the router
- Use `GET` for route loaders, `POST` for form submissions and mutations
- Both methods support `.inputValidator()` and `.middleware()`
