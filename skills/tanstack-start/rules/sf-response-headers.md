# sf-response-headers: Customize Response Headers When Needed

## Priority: MEDIUM

## Explanation

Server functions can read request headers and set response headers using utilities from `@tanstack/react-start/server`. This is useful for caching, authentication, CORS, and other HTTP-level concerns.

## Bad Example

```tsx
// Trying to access request/response directly - not available in server functions
export const getCachedData = createServerFn()
  .handler(async () => {
    // req and res don't exist in server function context
    const authHeader = req.headers.authorization
    res.setHeader('Cache-Control', 'max-age=300')
    return fetchData()
  })
```

## Good Example

```tsx
import { createServerFn } from '@tanstack/react-start'
import {
  getRequest,
  getRequestHeader,
  setResponseHeader,
  setResponseHeaders,
  setResponseStatus,
} from '@tanstack/react-start/server'

// Reading request headers
export const getProtectedData = createServerFn()
  .handler(async () => {
    const authHeader = getRequestHeader('Authorization')
    if (!authHeader?.startsWith('Bearer ')) {
      setResponseStatus(401)
      throw new Error('Unauthorized')
    }
    return fetchProtectedData(authHeader)
  })

// Setting cache headers
export const getCachedData = createServerFn()
  .handler(async () => {
    setResponseHeaders(new Headers({
      'Cache-Control': 'public, max-age=300',
      'CDN-Cache-Control': 'max-age=3600, stale-while-revalidate=600',
    }))
    return fetchData()
  })

// Setting individual headers
export const downloadFile = createServerFn()
  .inputValidator((data: { filename: string }) => data)
  .handler(async ({ data }) => {
    setResponseHeader('Content-Disposition', `attachment; filename="${data.filename}"`)
    setResponseHeader('Content-Type', 'application/octet-stream')
    return getFileContents(data.filename)
  })

// Accessing the full request object
export const handleWebhook = createServerFn({ method: 'POST' })
  .handler(async () => {
    const request = getRequest()
    const signature = request.headers.get('x-webhook-signature')
    const body = await request.text()

    if (!verifySignature(body, signature)) {
      setResponseStatus(403)
      throw new Error('Invalid signature')
    }

    await processWebhook(JSON.parse(body))
    return { received: true }
  })
```

## Context

- All utilities imported from `@tanstack/react-start/server`
- `getRequest()` returns the full `Request` object
- `getRequestHeader(name)` returns a specific request header
- `setResponseHeader(name, value)` sets a single response header
- `setResponseHeaders(headers)` sets multiple headers via a `Headers` object
- `setResponseStatus(code)` sets the HTTP status code
- These utilities use async context — they work anywhere inside a server function handler
