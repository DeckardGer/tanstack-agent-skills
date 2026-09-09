# search-defaults: Provide Sensible Defaults for Search Params

## Priority: HIGH

## Explanation

Always provide default values for search params so the app works when users visit a URL with no query string. Use `.catch()` in Zod for graceful fallbacks that never show errors, or use `fallback()` from `@tanstack/zod-adapter` when you need defaults that also make the params optional in `<Link>` navigation types.

## Bad Example

```tsx
// No defaults — crashes or shows errors on bare URL visit
const searchSchema = z.object({
  page: z.number(),           // Required — /products without ?page= throws
  sort: z.enum(['name', 'price']),  // Required — no fallback
})

export const Route = createFileRoute('/products')({
  validateSearch: searchSchema,
  // Visiting /products without search params triggers errorComponent
})
```

```tsx
// Using .default() without adapter — makes search params required in Link types
const searchSchema = z.object({
  page: z.number().default(1),
  sort: z.enum(['name', 'price']).default('name'),
})

export const Route = createFileRoute('/products')({
  validateSearch: searchSchema,
})

// Error: search is required on Link because .default() doesn't affect input type
<Link to="/products">Products</Link> // Type error!
<Link to="/products" search={{ page: 1, sort: 'name' }}>Products</Link> // Required
```

## Good Example: With .catch()

```tsx
import { z } from 'zod'

// .catch() provides fallbacks AND makes validation never throw
const searchSchema = z.object({
  page: z.number().min(1).catch(1),
  sort: z.enum(['name', 'price', 'date']).catch('name'),
  order: z.enum(['asc', 'desc']).catch('asc'),
  query: z.string().catch(''),
})

export const Route = createFileRoute('/products')({
  validateSearch: (search) => searchSchema.parse(search),
})

// /products → { page: 1, sort: 'name', order: 'asc', query: '' }
// /products?page=abc → { page: 1, sort: 'name', order: 'asc', query: '' }
// /products?page=3&sort=price → { page: 3, sort: 'price', order: 'asc', query: '' }
```

## Good Example: With Zod Adapter (Recommended)

```tsx
import { z } from 'zod'
import { fallback, zodValidator } from '@tanstack/zod-adapter'

// fallback() retains types while providing defaults
// .default() makes the param optional in Link search types
const searchSchema = z.object({
  page: fallback(z.number().min(1), 1).default(1),
  sort: fallback(z.enum(['name', 'price', 'date']), 'name').default('name'),
  order: fallback(z.enum(['asc', 'desc']), 'asc').default('asc'),
  query: fallback(z.string(), '').default(''),
})

export const Route = createFileRoute('/products')({
  validateSearch: zodValidator(searchSchema),
})

// Links don't require search params — defaults are used
<Link to="/products">Products</Link> // Works — no type error
<Link to="/products" search={{ page: 2 }}>Page 2</Link> // Also works
```

## Good Example: With Valibot (Standard Schema — No Adapter Needed)

```tsx
import * as v from 'valibot'

const searchSchema = v.object({
  page: v.optional(v.fallback(v.number(), 1), 1),
  sort: v.optional(v.fallback(v.picklist(['name', 'price', 'date']), 'name'), 'name'),
  query: v.optional(v.fallback(v.string(), ''), ''),
})

export const Route = createFileRoute('/products')({
  validateSearch: searchSchema, // Valibot implements Standard Schema
})
```

## Context

- `.catch()` makes Zod never throw on invalid input — returns the fallback value instead
- `.default()` provides a default when the value is `undefined` but doesn't affect the TypeScript input type
- `fallback()` from `@tanstack/zod-adapter` combines both: catches invalid values AND makes the param optional in navigation types
- Use `.catch()` when you want malformed URLs to silently use defaults
- Use the Zod adapter with `fallback()` + `.default()` for the best DX in production apps
- Valibot and ArkType implement Standard Schema and work without adapters
- When `validateSearch` throws, the route's `errorComponent` renders instead of `component`
