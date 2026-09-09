# cache-defaults: Set Sensible Defaults at QueryClient Level

## Priority: CRITICAL

## Explanation

Configure sensible default options at the `QueryClient` level to avoid repeating the same configuration across every query. The built-in defaults (`staleTime: 0`, `gcTime: 5 minutes`, `retry: 3`) are aggressive — override them based on your application's data patterns.

## Bad Example

```tsx
// No defaults configured — every query must set its own options
const queryClient = new QueryClient()

// Repetitive per-query configuration
const { data: profile } = useQuery({
  queryKey: ['profile'],
  queryFn: fetchProfile,
  staleTime: 60 * 1000,
  retry: 2,
})

const { data: settings } = useQuery({
  queryKey: ['settings'],
  queryFn: fetchSettings,
  staleTime: 60 * 1000,
  retry: 2,
})
```

## Good Example

```tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000,       // 1 minute — prevents unnecessary refetches
      gcTime: 10 * 60 * 1000,     // 10 minutes — retains inactive cache longer
      retry: 2,                    // Fewer retries than default 3
      refetchOnWindowFocus: false, // Disable if not needed
    },
    mutations: {
      retry: 0,  // Mutations should not retry by default
    },
  },
})

// Only override where specific queries differ
const { data: stockPrice } = useQuery({
  queryKey: ['stock', symbol],
  queryFn: () => fetchStockPrice(symbol),
  staleTime: 0,          // Override: real-time data
  refetchInterval: 5000, // Poll every 5 seconds
})

const { data: categories } = useQuery({
  queryKey: ['categories'],
  queryFn: fetchCategories,
  staleTime: 30 * 60 * 1000, // Override: rarely changes
})
```

## Context

- Default `staleTime: 0` means data is immediately stale — every mount triggers a refetch
- Default `gcTime: 300000` (5 min) is reasonable for most apps
- Default `retry: 3` with exponential backoff may be too aggressive for some APIs
- For SSR apps, set `staleTime` above 0 to avoid double-fetching after hydration
- Use `queryOptions` factory to centralize per-data-type overrides
- Mutation defaults are separate from query defaults
