# api-routes: Create API Routes for External Consumers

## Priority: MEDIUM

## Explanation

While server functions are ideal for internal RPC, API routes provide traditional REST endpoints for external consumers, webhooks, and integrations. Use API routes when you need standard HTTP semantics, custom response formats, or third-party compatibility.

## Bad Example

```tsx
// Using server functions for webhook endpoints
export const stripeWebhook = createServerFn({ method: 'POST' })
  .handler(async ({ request }) => {
    // Server functions aren't designed for raw request handling
    // No easy access to raw body for signature verification
    // Response format is JSON by default
  })

// Or exposing internal functions to external consumers
export const getUsers = createServerFn()
  .handler(async () => {
    return db.users.findMany()
  })
// No versioning, no standard REST semantics
```

## Good Example: Basic API Route

```tsx
// app/routes/api/users.ts
import { json } from '@tanstack/react-start'
import { createAPIFileRoute } from '@tanstack/react-start/api'

export const APIRoute = createAPIFileRoute('/api/users')({
  GET: async ({ request }) => {
    const users = await db.users.findMany({
      select: { id: true, name: true, email: true },
    })

    return json(users, {
      headers: {
        'Cache-Control': 'public, max-age=60',
      },
    })
  },

  POST: async ({ request }) => {
    const body = await request.json()

    // Validate input
    const parsed = createUserSchema.safeParse(body)
    if (!parsed.success) {
      return json({ error: parsed.error.flatten() }, { status: 400 })
    }

    const user = await db.users.create({ data: parsed.data })
    return json(user, { status: 201 })
  },
})
```

## Good Example: Webhook Handler

```tsx
// app/routes/api/webhooks/stripe.ts
import { createAPIFileRoute } from '@tanstack/react-start/api'
import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!)

export const APIRoute = createAPIFileRoute('/api/webhooks/stripe')({
  POST: async ({ request }) => {
    const signature = request.headers.get('stripe-signature')
    if (!signature) {
      return new Response('Missing signature', { status: 400 })
    }

    // Get raw body for signature verification
    const rawBody = await request.text()

    let event: Stripe.Event
    try {
      event = stripe.webhooks.constructEvent(
        rawBody,
        signature,
        process.env.STRIPE_WEBHOOK_SECRET!
      )
    } catch (err) {
      console.error('Webhook signature verification failed:', err)
      return new Response('Invalid signature', { status: 400 })
    }

    // Handle the event
    switch (event.type) {
      case 'checkout.session.completed':
        await handleCheckoutComplete(event.data.object)
        break
      case 'customer.subscription.updated':
        await handleSubscriptionUpdate(event.data.object)
        break
      default:
        console.log(`Unhandled event type: ${event.type}`)
    }

    return new Response('OK', { status: 200 })
  },
})
```

## Good Example: RESTful Resource

```tsx
// app/routes/api/posts/$postId.ts
import { json } from '@tanstack/react-start'
import { createAPIFileRoute } from '@tanstack/react-start/api'

export const APIRoute = createAPIFileRoute('/api/posts/$postId')({
  GET: async ({ params }) => {
    const post = await db.posts.findUnique({
      where: { id: params.postId },
    })

    if (!post) {
      return json({ error: 'Post not found' }, { status: 404 })
    }

    return json(post)
  },

  PUT: async ({ request, params }) => {
    const body = await request.json()
    const parsed = updatePostSchema.safeParse(body)

    if (!parsed.success) {
      return json({ error: parsed.error.flatten() }, { status: 400 })
    }

    const post = await db.posts.update({
      where: { id: params.postId },
      data: parsed.data,
    })

    return json(post)
  },

  DELETE: async ({ params }) => {
    await db.posts.delete({ where: { id: params.postId } })
    return new Response(null, { status: 204 })
  },
})
```

## Good Example: With Authentication

```tsx
// app/routes/api/protected/data.ts
import { json } from '@tanstack/react-start'
import { createAPIFileRoute } from '@tanstack/react-start/api'
import { verifyApiKey } from '@/lib/auth'

export const APIRoute = createAPIFileRoute('/api/protected/data')({
  GET: async ({ request }) => {
    // API key authentication
    const apiKey = request.headers.get('x-api-key')
    if (!apiKey) {
      return json({ error: 'Missing API key' }, { status: 401 })
    }

    const client = await verifyApiKey(apiKey)
    if (!client) {
      return json({ error: 'Invalid API key' }, { status: 401 })
    }

    // Rate limit check
    const withinLimits = await checkRateLimit(client.id)
    if (!withinLimits) {
      return json({ error: 'Rate limit exceeded' }, {
        status: 429,
        headers: { 'Retry-After': '60' },
      })
    }

    const data = await fetchDataForClient(client.id)
    return json(data)
  },
})
```

## Server Functions vs API Routes

| Feature | Server Functions | API Routes |
|---------|-----------------|------------|
| Primary use | Internal RPC | External consumers |
| Type safety | Full end-to-end | Manual |
| Response format | JSON (automatic) | Any (manual) |
| Raw request access | Limited | Full |
| URL structure | Auto-generated | Explicit paths |
| Webhooks | Not ideal | Designed for |

## Context

- API routes live in `app/routes/api/` directory
- Support all HTTP methods: GET, POST, PUT, PATCH, DELETE, etc.
- Use `json()` helper for JSON responses
- Return `Response` objects for custom formats
- Ideal for: webhooks, public APIs, file downloads, third-party integrations
- Consider versioning: `/api/v1/users` for public APIs
