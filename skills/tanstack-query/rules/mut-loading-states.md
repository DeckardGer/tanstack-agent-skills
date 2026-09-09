# mut-loading-states: Use isPending for Mutation Loading States

## Priority: MEDIUM

## Explanation

In TanStack Query v5, `isLoading` was renamed to `isPending` for mutations. Use `isPending` to disable buttons, show spinners, and prevent double submissions. Use `variables` to render optimistic UI while a mutation is in flight.

## Bad Example

```tsx
// Using v4 API — isLoading was renamed in v5
const mutation = useMutation({ mutationFn: createTodo })

// No loading state — allows double submission
<button onClick={() => mutation.mutate(newTodo)}>
  Create
</button>

// Checking status string instead of boolean helpers
if (mutation.status === 'loading') { /* v4 pattern, broken in v5 */ }
```

## Good Example

```tsx
const mutation = useMutation({
  mutationFn: createTodo,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
})

<button
  onClick={() => mutation.mutate(newTodo)}
  disabled={mutation.isPending}
>
  {mutation.isPending ? 'Creating...' : 'Create Todo'}
</button>
```

## Good Example: Inline Optimistic Item with variables

```tsx
function TodoList() {
  const { data: todos } = useQuery({ queryKey: ['todos'], queryFn: fetchTodos })
  const addTodo = useMutation({
    mutationFn: createTodo,
    onSettled: () => queryClient.invalidateQueries({ queryKey: ['todos'] }),
  })

  return (
    <ul>
      {todos?.map((todo) => (
        <li key={todo.id}>{todo.title}</li>
      ))}
      {addTodo.isPending && (
        <li style={{ opacity: 0.5 }}>{addTodo.variables.title}</li>
      )}
    </ul>
  )
}
```

## Mutation Status Reference

| Status | Boolean | Meaning |
|--------|---------|---------|
| `'idle'` | `isIdle` | Mutation has not been called yet |
| `'pending'` | `isPending` | Mutation is currently executing |
| `'error'` | `isError` | Last mutation attempt failed |
| `'success'` | `isSuccess` | Last mutation completed successfully |

## Context

- `isPending` replaced `isLoading` in v5 — check migration guide if upgrading
- `variables` holds the data passed to `mutate()` — available during `isPending` for optimistic UI
- `submittedAt` provides a timestamp for the mutation — useful as a key for optimistic items
- Always disable submit buttons during `isPending` to prevent double submissions
- Use `mutation.reset()` to return to `idle` state after showing success/error
