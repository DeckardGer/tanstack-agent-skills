# mut-rollback-context: Provide Rollback Context from onMutate

## Priority: HIGH

## Explanation

When implementing optimistic updates via cache manipulation, return a snapshot of the previous data from `onMutate`. This value is passed as the third argument to `onError`, `onSuccess`, and `onSettled`, enabling cache rollback on failure.

## Bad Example

```tsx
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] })

    // Optimistically update without saving previous state
    queryClient.setQueryData(['todos'], (old: Todo[]) =>
      old.map((t) => (t.id === newTodo.id ? { ...t, ...newTodo } : t))
    )

    // No return — nothing to rollback with
  },
  onError: (err) => {
    // Can't rollback — previous data is lost
    // Forced to refetch everything
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
})
```

## Good Example

```tsx
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    // 1. Cancel outgoing refetches to prevent overwriting optimistic update
    await queryClient.cancelQueries({ queryKey: ['todos'] })

    // 2. Snapshot the previous value
    const previousTodos = queryClient.getQueryData<Todo[]>(['todos'])

    // 3. Optimistically update the cache
    queryClient.setQueryData(['todos'], (old: Todo[]) =>
      old.map((t) => (t.id === newTodo.id ? { ...t, ...newTodo } : t))
    )

    // 4. Return snapshot as rollback context
    return { previousTodos }
  },
  onError: (err, newTodo, context) => {
    // Rollback to the snapshot on failure
    if (context?.previousTodos) {
      queryClient.setQueryData(['todos'], context.previousTodos)
    }
  },
  onSettled: () => {
    // Always refetch to sync with server truth
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
})
```

## Good Example: Multiple Cache Entries

```tsx
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] })
    await queryClient.cancelQueries({ queryKey: ['todo', newTodo.id] })

    // Snapshot both the list and detail caches
    const previousTodos = queryClient.getQueryData<Todo[]>(['todos'])
    const previousTodo = queryClient.getQueryData<Todo>(['todo', newTodo.id])

    queryClient.setQueryData(['todos'], (old: Todo[]) =>
      old.map((t) => (t.id === newTodo.id ? { ...t, ...newTodo } : t))
    )
    queryClient.setQueryData(['todo', newTodo.id], (old: Todo) => ({
      ...old,
      ...newTodo,
    }))

    return { previousTodos, previousTodo }
  },
  onError: (err, newTodo, context) => {
    if (context?.previousTodos) {
      queryClient.setQueryData(['todos'], context.previousTodos)
    }
    if (context?.previousTodo) {
      queryClient.setQueryData(['todo', newTodo.id], context.previousTodo)
    }
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
})
```

## Context

- The return value from `onMutate` is the third argument (`context`) in `onError`, `onSuccess`, and `onSettled`
- Always cancel related queries before optimistic updates to prevent race conditions
- Always call `invalidateQueries` in `onSettled` to reconcile with the server
- For simple UI-only optimistic updates, use `mutation.variables` and `mutation.isPending` instead — no rollback needed
- Test error scenarios to verify rollback restores correct state
