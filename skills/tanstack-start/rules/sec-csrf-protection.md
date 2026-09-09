# sec-csrf-protection: Implement CSRF Protection for Mutations

## Priority: CRITICAL

## Explanation

Cross-Site Request Forgery (CSRF) attacks trick authenticated users into making unintended requests. TanStack Start server functions use RPC calls (not plain form submissions), which provides some inherent protection. Cookie-based sessions should use `sameSite` settings for additional CSRF defense.

## Bad Example

```tsx
// No sameSite on session cookie - vulnerable to CSRF
export function useAppSession() {
  return useSession({
    password: process.env.SESSION_SECRET!,
    cookie: {
      httpOnly: true,
      secure: true,
      // Missing sameSite! Browser sends cookie on cross-site requests
    },
  })
}

// GET server function that mutates data - can be triggered by img tags
export const deleteAccount = createServerFn()  // GET is default
  .handler(async () => {
    const session = await useAppSession()
    await db.users.delete({ where: { id: session.data.userId } })
  })
```

## Good Example

```tsx
import { useSession } from '@tanstack/react-start/server'

// Configure session cookies with sameSite protection
export function useAppSession() {
  return useSession<SessionData>({
    password: process.env.SESSION_SECRET!,  // At least 32 characters
    cookie: {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',    // Blocks cross-site POST requests
      maxAge: 60 * 60 * 24 * 7,
    },
  })
}

// Use POST for all mutations - never GET
export const deleteAccount = createServerFn({ method: 'POST' })
  .handler(async () => {
    const session = await useAppSession()
    if (!session.data.userId) {
      throw redirect({ to: '/login' })
    }
    await db.users.delete({ where: { id: session.data.userId } })
    await session.clear()
    throw redirect({ to: '/' })
  })
```

## Good Example: OAuth State Parameter for CSRF in Auth Flows

```tsx
export const startOAuth = createServerFn({ method: 'POST' })
  .handler(async () => {
    const session = await useAppSession()
    const state = crypto.randomUUID()

    await session.update({ oauthState: state })

    const authUrl = new URL('https://provider.com/oauth/authorize')
    authUrl.searchParams.set('state', state)
    authUrl.searchParams.set('client_id', process.env.OAUTH_CLIENT_ID!)
    authUrl.searchParams.set('redirect_uri', process.env.OAUTH_CALLBACK_URL!)

    throw redirect({ href: authUrl.toString() })
  })

export const handleOAuthCallback = createServerFn()
  .inputValidator((data: { code: string; state: string }) => data)
  .handler(async ({ data }) => {
    const session = await useAppSession()

    // Verify state to prevent CSRF
    if (data.state !== session.data.oauthState) {
      throw new Error('Invalid OAuth state - possible CSRF attack')
    }

    // Exchange code for token...
    const token = await exchangeCodeForToken(data.code)
    await session.update({ userId: token.userId, oauthState: undefined })
    throw redirect({ to: '/dashboard' })
  })
```

## Context

- `sameSite: 'lax'` — cookies sent on top-level navigations but not cross-site POST/AJAX
- `sameSite: 'strict'` — cookies never sent on cross-site requests (may break OAuth flows)
- Server functions use RPC (fetch + JSON), not plain form submissions, adding inherent CSRF resistance
- Always use `POST` for state-changing operations
- For OAuth flows, always use a random `state` parameter stored in the session
- The combination of `sameSite: 'lax'` + POST-only mutations provides strong CSRF protection
