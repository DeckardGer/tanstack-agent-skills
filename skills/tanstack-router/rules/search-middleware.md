# search-middleware: Use Search Param Middleware

## Priority: HIGH

## Explanation

Search middlewares transform search params when generating link hrefs and upon navigation. Use the built-in `retainSearchParams` middleware to persist params across all navigations, and `stripSearchParams` to keep URLs clean by removing params that match default values. You can also write custom middlewares for advanced transformations.

## Bad Example

```tsx
// Manually retaining search params in every Link — tedious, error-prone
function Navigation() {
  const { locale } = Route.useSearch()

  return (
    <nav>
      {/* Must manually pass locale to every single link */}
      <Link to="/dashboard" search={{ locale }}>Dashboard</Link>
      <Link to="/settings" search={{ locale }}>Settings</Link>
      <Link to="/profile" search={{ locale }}>Profile</Link>
      {/* Forget one and locale is lost */}
    </nav>
  )
}
```

## Good Example: retainSearchParams

```tsx
import { createRootRoute } from '@tanstack/react-router'
import { retainSearchParams } from '@tanstack/react-router'
import { z } from 'zod'

export const Route = createRootRoute({
  validateSearch: z.object({
    locale: z.enum(['en', 'es', 'fr']).catch('en'),
  }),
  search: {
    middlewares: [retainSearchParams(['locale'])],
  },
})

// Now locale is automatically retained on every link and navigation
// No need to manually pass it
function Navigation() {
  return (
    <nav>
      <Link to="/dashboard">Dashboard</Link>  {/* locale auto-retained */}
      <Link to="/settings">Settings</Link>    {/* locale auto-retained */}
    </nav>
  )
}
```

## Good Example: stripSearchParams

```tsx
import { stripSearchParams } from '@tanstack/react-router'

const defaultSearch = {
  page: 1,
  sort: 'newest',
  filter: '',
}

export const Route = createFileRoute('/products')({
  validateSearch: z.object({
    page: z.number().catch(1),
    sort: z.enum(['newest', 'oldest', 'price']).catch('newest'),
    filter: z.string().catch(''),
  }),
  search: {
    middlewares: [stripSearchParams(defaultSearch)],
  },
})

// URL: /products (when page=1, sort=newest, filter='')
// URL: /products?page=2 (only non-default params shown)
// URL: /products?page=2&sort=price (two non-defaults)
```

## Good Example: Combining Middlewares

```tsx
export const Route = createFileRoute('/products')({
  validateSearch: productSearchSchema,
  search: {
    middlewares: [
      retainSearchParams(['locale']),
      stripSearchParams({ page: 1, sort: 'newest' }),
    ],
  },
})
```

## Good Example: Custom Middleware

```tsx
export const Route = createRootRoute({
  validateSearch: searchSchema,
  search: {
    middlewares: [
      ({ search, next }) => {
        const result = next(search)
        return {
          rootValue: search.rootValue, // Always preserve rootValue
          ...result,
        }
      },
    ],
  },
})
```

## Context

- Middlewares run when generating link hrefs and during navigation after search validation
- `retainSearchParams` — automatically carries specified params across navigations without manual spreading
- `stripSearchParams` — removes params from the URL when they match default values, keeping URLs clean
- Middlewares can be chained in the `middlewares` array — they run in order
- Custom middlewares receive `{ search, next }` — call `next(search)` to continue the chain
- Define on the root route for global params (locale, theme), or on specific routes for scoped params
