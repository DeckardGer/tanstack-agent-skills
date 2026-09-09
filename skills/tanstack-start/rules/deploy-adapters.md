# deploy-adapters: Choose Appropriate Deployment Adapter

## Priority: LOW

## Explanation

TanStack Start is built on Vite and supports deployment to any hosting provider via platform-specific Vite plugins. Each adapter optimizes the build output for its platform's runtime. Configuration is done in `vite.config.ts` using `tanstackStart()` from `@tanstack/react-start/plugin/vite`.

## Bad Example

```tsx
// Using the old app.config.ts pattern - no longer valid
// app.config.ts
import { defineConfig } from '@tanstack/react-start/config'

export default defineConfig({
  server: {
    preset: 'vercel',  // Old preset-based config
  },
})
```

## Good Example: Cloudflare Workers

```bash
pnpm add -D @cloudflare/vite-plugin wrangler
```

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import { cloudflare } from '@cloudflare/vite-plugin'
import viteReact from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [
    cloudflare({ viteEnvironment: { name: 'ssr' } }),
    tanstackStart(),
    viteReact(),
  ],
})
```

```jsonc
// wrangler.jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "tanstack-start-app",
  "compatibility_date": "2025-09-02",
  "compatibility_flags": ["nodejs_compat"],
  "main": "@tanstack/react-start/server-entry"
}
```

## Good Example: Netlify

```bash
npm install -D @netlify/vite-plugin-tanstack-start
```

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import netlify from '@netlify/vite-plugin-tanstack-start'
import viteReact from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [
    tanstackStart(),
    netlify(),
    viteReact(),
  ],
})
```

## Good Example: Nitro (Vercel, Node.js, Docker)

```json
{
  "dependencies": {
    "nitro": "npm:nitro-nightly@latest"
  }
}
```

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import { nitro } from 'nitro/vite'
import viteReact from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [
    tanstackStart(),
    nitro(),
    viteReact(),
  ],
})
```

```dockerfile
# Dockerfile for Node.js deployment
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY .output .output
EXPOSE 3000
CMD ["node", ".output/server/index.mjs"]
```

## Good Example: Bun

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import { nitro } from 'nitro/vite'
import viteReact from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [
    tanstackStart(),
    nitro({ preset: 'bun' }),
    viteReact(),
  ],
})
```

## Adapter Comparison

| Platform | Plugin | Notes |
|----------|--------|-------|
| Cloudflare Workers | `@cloudflare/vite-plugin` | Official partner, edge runtime |
| Netlify | `@netlify/vite-plugin-tanstack-start` | Official partner |
| Railway | Nitro adapter | Auto-detects, push to deploy |
| Vercel | Nitro adapter | Via `nitro/vite` |
| Node.js / Docker | Nitro adapter | `node .output/server/index.mjs` |
| Bun | Nitro with `preset: 'bun'` | Requires React 19 |
| Appwrite Sites | Standard build | Configure output dir in dashboard |

## Context

- All deployment config is in `vite.config.ts` — there is no `app.config.ts`
- `tanstackStart()` must come BEFORE `viteReact()` in the plugins array
- Cloudflare and Netlify have dedicated first-party plugins
- All other platforms use the generic Nitro adapter (`nitro/vite`)
- Edge runtimes have API limitations (no file system, limited Node.js APIs)
- Test locally with `vite build && vite preview`
- ISR uses standard HTTP `Cache-Control` headers, not a framework-specific API
