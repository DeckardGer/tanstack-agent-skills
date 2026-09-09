# preload-stale-time: Configure Preload Stale Time

## Priority: MEDIUM

## Explanation

Preloaded route data is considered fresh for 30 seconds by default (`preloadStaleTime`). During this window, navigating to a preloaded route is instant. Configure this value based on how frequently your data changes. When using an external cache like TanStack Query, set `defaultPreloadStaleTime: 0` so every preload triggers the loader and lets the external library handle freshness.

## Bad Example

```tsx
// Using TanStack Query but keeping default preloadStaleTime (30s)
// The router's built-in cache prevents the loader from running on preload
const router = createRouter({
  routeTree,
  defaultPreload: 'intent',
  // defaultPreloadStaleTime: 30_000 (default)
  // Problem: TanStack Query's staleTime is ignored during preloads
  // Data may be stale according to Query but fresh according to Router
})
```

## Good Example: With TanStack Query

```tsx
const router = createRouter({
  routeTree,
  defaultPreload: 'intent',
  // Let TanStack Query manage all caching decisions
  defaultPreloadStaleTime: 0,
})
```

## Good Example: Built-In Cache Tuning

```tsx
// Router-level defaults
const router = createRouter({
  routeTree,
  defaultPreload: 'intent',
  defaultPreloadStaleTime: 10_000,  // Preloaded data fresh for 10 seconds
  defaultStaleTime: 5_000,          // Navigation data fresh for 5 seconds
})

// Per-route override for frequently changing data
export const Route = createFileRoute('/live-feed')({
  loader: () => fetchLiveFeed(),
  preloadStaleTime: 5_000,   // Refresh preloaded data after 5 seconds
  staleTime: 0,              // Always reload on navigation
})

// Per-route override for rarely changing data
export const Route = createFileRoute('/about')({
  loader: () => fetchAboutContent(),
  preloadStaleTime: 60_000,  // Preloaded data good for 1 minute
  staleTime: Infinity,       // Never auto-reload
})
```

## Cache Time Comparison

| Option | Default | Controls |
|---|---|---|
| `preloadStaleTime` | `30_000` (30s) | How long preloaded data is fresh before another preload triggers |
| `staleTime` | `0` | How long navigated data is fresh before background reload on re-match |
| `gcTime` | `1_800_000` (30min) | How long unused cached data is kept before garbage collection |

## Context

- `preloadStaleTime` only affects preloaded data — data loaded via actual navigation uses `staleTime`
- Set `defaultPreloadStaleTime: 0` when using TanStack Query, SWR, or other external caches — this ensures every preload triggers the loader so the external library can handle deduplication and freshness
- For built-in cache: higher `preloadStaleTime` = fewer network requests but potentially staler data
- `router.invalidate()` marks all cached data as stale and forces immediate reload of active routes
- `preloadStaleTime` is configurable at the router level (`defaultPreloadStaleTime`) and per-route (`preloadStaleTime`)
