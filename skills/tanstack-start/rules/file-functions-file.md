# file-functions-file: Use .functions.ts Pattern for Server Functions

## Priority: LOW

## Explanation

Organize server functions in `.functions.ts` files to clearly separate RPC wrappers from server-only implementation details. Static imports of server function files are safe anywhere — the bundler replaces handler implementations with RPC stubs in client bundles.

## Bad Example

```tsx
// Mixing server functions with component code
// routes/posts.tsx
import { createServerFn } from '@tanstack/react-start'
import { db } from '../lib/db.server'  // Server import in route file

const getPosts = createServerFn().handler(async () => {
  return db.posts.findMany()
})

const createPost = createServerFn({ method: 'POST' })
  .inputValidator(createPostSchema)
  .handler(async ({ data }) => {
    return db.posts.create({ data })
  })

// 200 lines of server functions mixed with component code...

export const Route = createFileRoute('/posts')({
  loader: () => getPosts(),
  component: PostsPage,
})
```

## Good Example

```
src/
├── utils/
│   ├── posts.functions.ts   # Server function wrappers (createServerFn)
│   ├── posts.server.ts      # Server-only helpers (direct DB, internal logic)
│   └── posts.schema.ts      # Shared validation schemas (client-safe)
├── routes/
│   └── posts.tsx             # Route definition — clean, focused
```

```tsx
// utils/posts.schema.ts — shared between client and server
import { z } from 'zod'

export const createPostSchema = z.object({
  title: z.string().min(1).max(200).trim(),
  content: z.string().min(1).max(50000),
  published: z.boolean().default(false),
})

// utils/posts.server.ts — server-only implementation
import { db } from '../lib/db'

export async function findPosts(offset: number, limit: number) {
  return db.posts.findMany({
    skip: offset,
    take: limit,
    orderBy: { createdAt: 'desc' },
  })
}

export async function insertPost(data: { title: string; content: string }) {
  return db.posts.create({ data })
}

// utils/posts.functions.ts — server function wrappers
import { createServerFn } from '@tanstack/react-start'
import { createPostSchema } from './posts.schema'

export const getPosts = createServerFn()
  .inputValidator((data: { offset: number; limit: number }) => data)
  .handler(async ({ data }) => {
    const { findPosts } = await import('./posts.server')
    return findPosts(data.offset, data.limit)
  })

export const createPost = createServerFn({ method: 'POST' })
  .inputValidator(createPostSchema)
  .handler(async ({ data }) => {
    const { insertPost } = await import('./posts.server')
    return insertPost(data)
  })

// routes/posts.tsx — clean route file
import { createFileRoute } from '@tanstack/react-router'
import { getPosts } from '../utils/posts.functions'

export const Route = createFileRoute('/posts')({
  loader: () => getPosts({ data: { offset: 0, limit: 20 } }),
  component: PostsPage,
})
```

## Context

- `.functions.ts` — `createServerFn` wrappers, safe to import anywhere
- `.server.ts` — server-only helpers (database queries, internal logic)
- `.schema.ts` or `schemas.ts` — validation schemas shared between client and server
- Static imports of `.functions.ts` files are safe — the bundler replaces handlers with RPC stubs
- Do NOT use dynamic imports for server functions themselves — only for server-only helpers inside handlers
- This pattern keeps route files focused on routing concerns
- Group by domain (posts, users, auth) rather than by layer
