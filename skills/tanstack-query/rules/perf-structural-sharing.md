# perf-structural-sharing: Leverage Structural Sharing

## Priority: LOW

## Explanation

TanStack Query uses structural sharing to preserve referential identity for unchanged data between refetches. When a background refetch returns data, unchanged portions keep their original object references, preventing unnecessary re-renders in components that depend on stable references.

## Bad Example

```tsx
// Disabling structural sharing unnecessarily
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  structuralSharing: false, // Every refetch creates new references for everything
  // Downstream useMemo/useEffect with data deps re-trigger on every refetch
})

// Using non-JSON data types breaks structural sharing silently
const { data } = useQuery({
  queryKey: ['dates'],
  queryFn: async () => {
    const res = await fetch('/api/events')
    const events = await res.json()
    // Returning Map/Set/Date instances — structural sharing only works with JSON types
    return new Map(events.map((e) => [e.id, new Date(e.date)]))
  },
})
```

## Good Example

```tsx
// Structural sharing is enabled by default — just works
const { data: todos } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  // structuralSharing: true (default)
  // If refetch returns same data, `todos` keeps identical references
})

// Safe to use in dependency arrays
useEffect(() => {
  // Only runs when data actually changes, not on every refetch
  analytics.track('todos-loaded', { count: todos?.length })
}, [todos])
```

## Good Example: Custom Structural Sharing

```tsx
// For non-JSON types, provide a custom comparator
import { replaceEqualDeep } from '@tanstack/react-query'

const { data } = useQuery({
  queryKey: ['events'],
  queryFn: fetchEvents,
  structuralSharing: (oldData, newData) => {
    // Custom comparison that handles your data types
    return replaceEqualDeep(oldData, newData)
  },
})
```

## Context

- Structural sharing only works with JSON-compatible values (objects, arrays, strings, numbers, booleans, null)
- Map, Set, Date, and class instances are not structurally compared — consider keeping data as plain JSON
- The comparison happens twice when using `select`: once on the raw result, once on the selected output
- Disable with `structuralSharing: false` only when using non-JSON data types or when the comparison cost exceeds the render savings
- This is why TanStack Query can refetch aggressively (window focus, mount) without causing excessive re-renders
