# ssr-prerender: Configure Static Prerendering and ISR

## Priority: MEDIUM

## Explanation

Static prerendering generates HTML at build time for pages that don't require request-time data. Incremental Static Regeneration (ISR) extends this by revalidating cached pages on a schedule. Use these for better performance and lower server costs.

## Bad Example

```tsx
// SSR for completely static content - wasteful
export const Route = createFileRoute('/about')({
  loader: async () => {
    // Fetching static content on every request
    const content = await fetchAboutPageContent()
    return { content }
  },
})

// Or no caching headers for semi-static content
export const Route = createFileRoute('/blog/$slug')({
  loader: async ({ params }) => {
    const post = await fetchPost(params.slug)
    return { post }
    // Every request hits the database
  },
})
```

## Good Example: Static Prerendering

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import viteReact from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [
    tanstackStart({
      prerender: {
        enabled: true,
        crawlLinks: true,
        autoStaticPathsDiscovery: true,
        concurrency: 14,
        filter: ({ path }) => !path.startsWith('/app'),
      },
    }),
    viteReact(),
  ],
})
```

```tsx
// routes/about.tsx - Will be prerendered
export const Route = createFileRoute('/about')({
  loader: async () => {
    // Runs at BUILD time, not request time
    const content = await fetchAboutPageContent()
    return { content }
  },
  component: AboutPage,
})
```

## Good Example: Per-Page Prerender Config

```ts
// vite.config.ts
tanstackStart({
  prerender: {
    enabled: true,
    crawlLinks: true,
  },
  pages: [
    {
      path: '/landing',
      prerender: { enabled: true, outputPath: '/landing/index.html' },
    },
  ],
})
```

## Good Example: ISR with Cache Headers

```tsx
// routes/blog/$slug.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/blog/$slug')({
  loader: async ({ params }) => fetchPost(params.slug),
  // ISR via standard HTTP cache headers
  headers: () => ({
    'Cache-Control': 'public, max-age=3600, s-maxage=3600, stale-while-revalidate=86400',
  }),
  staleTime: 60_000,      // Client-side stale time (1 minute)
  gcTime: 5 * 60_000,     // Client-side GC time (5 minutes)
  component: BlogPost,
})

// First request: SSR and cache at CDN
// Next 3600 seconds: Serve cached version
// After 3600 seconds: Serve stale, revalidate in background
// After 86400 seconds: Full SSR again
```

## Good Example: Hybrid Static/Dynamic

```tsx
// routes/products.tsx - Prerendered
export const Route = createFileRoute('/products')({
  loader: async () => {
    // Featured products - prerendered at build
    const featured = await fetchFeaturedProducts()
    return { featured }
  },
})

// routes/products/$productId.tsx - ISR
export const Route = createFileRoute('/products/$productId')({
  loader: async ({ params }) => {
    const product = await fetchProduct(params.productId)
    if (!product) throw notFound()
    return { product }
  },
  // Cache product pages for 5 minutes
  headers: () => ({
    'Cache-Control': 'public, s-maxage=300, stale-while-revalidate=600',
  }),
})

// routes/cart.tsx - Always SSR (user-specific)
export const Route = createFileRoute('/cart')({
  loader: async ({ context }) => {
    const cart = await fetchUserCart(context.user.id)
    return { cart }
  },
  headers: () => ({
    'Cache-Control': 'private, no-store',
  }),
})
```

## Good Example: On-Demand Revalidation via Server Route

```tsx
// routes/api/revalidate.ts
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/api/revalidate')({
  server: {
    handlers: {
      POST: async ({ request }) => {
        const { secret, path } = await request.json()

        if (secret !== process.env.REVALIDATE_SECRET) {
          return Response.json({ error: 'Invalid secret' }, { status: 401 })
        }

        // Trigger revalidation (implementation depends on hosting)
        await revalidatePath(path)
        return Response.json({ revalidated: true, path })
      },
    },
  },
})

// Usage: POST /api/revalidate { "secret": "...", "path": "/blog/my-post" }
```

## Cache-Control Directives

| Directive | Meaning |
|-----------|---------|
| `s-maxage=N` | CDN cache duration (seconds) |
| `max-age=N` | Browser cache duration |
| `stale-while-revalidate=N` | Serve stale while fetching fresh |
| `private` | Don't cache on CDN (user-specific) |
| `no-store` | Never cache |

## Context

- Prerendering happens at build time - no request context
- ISR requires CDN/edge support (Vercel, Cloudflare, etc.)
- Use prerendering for truly static pages (about, pricing)
- Use ISR for content that changes but not per-request
- Always SSR for user-specific or real-time data
- Test with production builds - dev server is always SSR
