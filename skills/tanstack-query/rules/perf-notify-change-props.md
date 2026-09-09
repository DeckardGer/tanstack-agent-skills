# perf-notify-change-props: Limit Re-renders with notifyOnChangeProps

## Priority: LOW

## Explanation

TanStack Query uses JavaScript Proxy to track which properties a component accesses from the query result. It only triggers re-renders when one of those accessed properties changes. This happens automatically, but certain patterns can break it.

## Bad Example

```tsx
// Object rest destructuring — disables tracked properties optimization
const { data, ...rest } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
})
// `...rest` accesses ALL properties via the Proxy's get trap
// Component re-renders when ANY property changes (isFetching, isStale, etc.)

// Accessing properties you don't need
function TodoList() {
  const { data, isFetching, isStale, dataUpdatedAt, status } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  })
  // Only uses `data` in render, but re-renders for isFetching, isStale changes too
  return <List items={data} />
}
```

## Good Example

```tsx
// Only destructure what you use — tracking works automatically
function TodoList() {
  const { data } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  })
  // Only re-renders when `data` changes
  return <List items={data} />
}

// When you need isFetching for a spinner, that's fine — it's intentional
function TodoListWithRefresh() {
  const { data, isFetching } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  })
  return (
    <>
      {isFetching && <Spinner />}
      <List items={data} />
    </>
  )
}
```

## Good Example: Manual Override

```tsx
// Explicitly control which properties trigger re-renders
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  notifyOnChangeProps: ['data', 'error'], // Only re-render for data or error changes
})

// Opt out of tracking entirely — re-render on any change
const result = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  notifyOnChangeProps: 'all',
})
```

## Good Example: Prefetch Observer (No Re-renders)

```tsx
// Use notifyOnChangeProps: [] to observe a query without causing re-renders
// Useful for prefetching in a parent component
useQuery({
  queryKey: ['comments', postId],
  queryFn: () => fetchComments(postId),
  notifyOnChangeProps: [], // Never triggers re-render — just populates cache
})
```

## Context

- Tracked properties is the default since v4 — no configuration needed for most cases
- Avoid object rest destructuring (`...rest`) as it breaks the Proxy optimization
- There is an ESLint rule (`no-rest-destructuring`) to catch this pattern
- Tracking only works during render — properties accessed in `useEffect` bodies are not tracked
- Tracking persists across render cycles — once a property is tracked, it stays tracked
- `notifyOnChangeProps: []` is useful for "observer-only" queries that prefetch without rendering
