# deploy-env-config: Configure Environment Variables Correctly

## Priority: LOW

## Explanation

TanStack Start follows Vite's environment variable conventions. Only `VITE_`-prefixed variables are available on the client via `import.meta.env`. All variables are available in server functions via `process.env`. Use type declarations and runtime validation to catch misconfiguration early.

## Bad Example

```tsx
// Exposing secrets via VITE_ prefix — included in client bundle!
// .env
VITE_DATABASE_URL=postgresql://user:pass@host/db
VITE_API_SECRET=sk-secret-key

// Accessing non-VITE_ vars on the client — undefined at runtime
function AppHeader() {
  return <h1>{process.env.APP_NAME}</h1>  // undefined on client
}

// No validation — silent failures in production
const db = connect(process.env.DATABASE_URL)  // Could be undefined
```

## Good Example: Environment Variable Separation

```bash
# .env — committed to git (no secrets)
VITE_APP_NAME=MyApp
VITE_API_URL=http://localhost:3000/api

# .env.local — gitignored (secrets and local overrides)
DATABASE_URL=postgresql://user:pass@localhost/mydb
SESSION_SECRET=your-32-char-secret-here
API_SECRET=sk-secret-key
```

```tsx
// Server function — access any env var
import { createServerFn } from '@tanstack/react-start'

const getConfig = createServerFn().handler(async () => {
  // Safe: server-only, never in client bundle
  const dbUrl = process.env.DATABASE_URL
  return { connected: !!dbUrl }
})

// Component — only VITE_ prefixed vars
function AppHeader() {
  return <h1>{import.meta.env.VITE_APP_NAME}</h1>
}
```

## Good Example: Type Declarations

```tsx
// src/env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_APP_NAME: string
  readonly VITE_API_URL: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}

declare global {
  namespace NodeJS {
    interface ProcessEnv {
      readonly DATABASE_URL: string
      readonly SESSION_SECRET: string
      readonly API_SECRET: string
      readonly NODE_ENV: 'development' | 'production' | 'test'
    }
  }
}

export {}
```

## Good Example: Runtime Validation with Zod

```tsx
// utils/env.server.ts — validate on startup
import { z } from 'zod'

const serverEnvSchema = z.object({
  DATABASE_URL: z.string().url(),
  SESSION_SECRET: z.string().min(32),
  API_SECRET: z.string().min(1),
  NODE_ENV: z.enum(['development', 'production', 'test']),
})

export const env = serverEnvSchema.parse(process.env)

// utils/env.client.ts — validate client env
const clientEnvSchema = z.object({
  VITE_APP_NAME: z.string().min(1),
  VITE_API_URL: z.string().url(),
})

export const clientEnv = clientEnvSchema.parse(import.meta.env)
```

## Good Example: Runtime Client Variables via Server Functions

```tsx
// For env vars that shouldn't be in the client bundle but are needed at runtime
const getRuntimeConfig = createServerFn().handler(async () => {
  return {
    featureFlags: JSON.parse(process.env.FEATURE_FLAGS ?? '{}'),
    apiVersion: process.env.API_VERSION ?? 'v1',
  }
})

export const Route = createFileRoute('/')({
  loader: () => getRuntimeConfig(),
  component: App,
})
```

## .env File Loading Order

```
.env                # Default values (commit to git)
.env.development    # Dev-specific overrides
.env.production     # Production-specific overrides
.env.local          # Local overrides (always gitignored)
```

## Context

- `VITE_` prefixed variables: client-safe, available via `import.meta.env`
- Non-prefixed variables: server-only, available via `process.env` in server functions
- Never prefix secrets with `VITE_` — they will be in the client bundle
- Use `.env.local` for secrets during development (gitignored by default)
- Type declarations in `src/env.d.ts` give autocompletion and type checking
- Runtime validation catches missing/malformed env vars at startup
- Use server functions to safely pass non-`VITE_` values to the client at runtime
