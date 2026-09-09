# auth-cookie-security: Configure Secure Cookie Settings

## Priority: HIGH

## Explanation

Session cookies are the primary attack surface for authentication. Misconfigured cookies can expose sessions to XSS, CSRF, and man-in-the-middle attacks. Always use `httpOnly`, `secure`, and `sameSite` settings.

## Bad Example

```tsx
// Minimal session config - missing security settings
export function useAppSession() {
  return useSession({
    password: 'short',  // Too short, hardcoded
  })
}

// Storing token in localStorage
function login(token: string) {
  localStorage.setItem('token', token)  // XSS can steal this
}

// Non-httpOnly cookie
export function useAppSession() {
  return useSession({
    password: process.env.SESSION_SECRET!,
    cookie: {
      httpOnly: false,  // JavaScript can read the session cookie
      secure: false,    // Sent over HTTP in production
    },
  })
}
```

## Good Example

```tsx
import { useSession } from '@tanstack/react-start/server'

type SessionData = {
  userId?: string
  email?: string
  role?: string
}

export function useAppSession() {
  return useSession<SessionData>({
    name: 'app-session',
    password: process.env.SESSION_SECRET!,  // At least 32 characters
    cookie: {
      httpOnly: true,      // Not accessible via document.cookie / JavaScript
      secure: process.env.NODE_ENV === 'production',  // HTTPS only in prod
      sameSite: 'lax',     // Blocks cross-site POST but allows navigation
      maxAge: 60 * 60 * 24 * 7,  // 7 days
    },
  })
}
```

## Good Example: Session Operations

```tsx
import { createServerFn } from '@tanstack/react-start'
import { redirect } from '@tanstack/react-router'

export const login = createServerFn({ method: 'POST' })
  .inputValidator(loginSchema)
  .handler(async ({ data }) => {
    const user = await verifyCredentials(data.email, data.password)
    if (!user) throw new Error('Invalid credentials')

    const session = await useAppSession()
    await session.update({
      userId: user.id,
      email: user.email,
      role: user.role,
    })

    throw redirect({ to: '/dashboard' })
  })

export const logout = createServerFn({ method: 'POST' })
  .handler(async () => {
    const session = await useAppSession()
    await session.clear()  // Destroys the session
    throw redirect({ to: '/' })
  })
```

## Cookie Security Settings Reference

| Setting | Recommended | Purpose |
|---------|-------------|---------|
| `httpOnly` | `true` | Prevents XSS from reading cookie via JavaScript |
| `secure` | `true` in prod | Cookie only sent over HTTPS |
| `sameSite` | `'lax'` | Blocks cross-site POST requests (CSRF protection) |
| `maxAge` | App-specific | Session duration in seconds |
| `password` | 32+ random chars | Encryption key for cookie contents |

## Context

- Generate `SESSION_SECRET` with `openssl rand -base64 32`
- `sameSite: 'lax'` allows cookies on top-level navigations (needed for OAuth callbacks)
- `sameSite: 'strict'` provides stronger CSRF protection but may break OAuth flows
- Store minimal data in the session (user ID, role) — fetch full user data on demand
- Implement session rotation on privilege changes (login, role change)
- Consider clearing sessions on password change
- `useSession` is from `@tanstack/react-start/server`
