# mut-error-handling: Handle Mutation Errors Gracefully

## Priority: HIGH

## Explanation

Mutations can fail for many reasons — network errors, validation failures, server issues. Always handle mutation errors explicitly using `onError` callbacks, `mutateAsync` with try/catch, or `throwOnError` with error boundaries. Use `reset()` to clear error state for retry flows.

## Bad Example

```tsx
const mutation = useMutation({
  mutationFn: createPost,
  // No error handling — errors silently stored in mutation.error
  // User has no way to know what happened or retry
})

// Fire and forget — no feedback on failure
<button onClick={() => mutation.mutate(newPost)}>Create</button>
```

## Good Example: onError Callback

```tsx
const mutation = useMutation({
  mutationFn: createPost,
  onError: (error) => {
    toast.error(`Failed to create post: ${error.message}`)
  },
  onSuccess: () => {
    toast.success('Post created!')
    queryClient.invalidateQueries({ queryKey: ['posts'] })
  },
})
```

## Good Example: Inline Error Display with reset()

```tsx
function CreatePostForm() {
  const mutation = useMutation({ mutationFn: createPost })

  return (
    <form onSubmit={(e) => {
      e.preventDefault()
      mutation.mutate(formData)
    }}>
      {/* ... form fields ... */}

      {mutation.isError && (
        <div className="error">
          <p>Error: {mutation.error.message}</p>
          <button type="button" onClick={() => mutation.reset()}>
            Dismiss
          </button>
        </div>
      )}

      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Creating...' : 'Create Post'}
      </button>
    </form>
  )
}
```

## Good Example: mutateAsync with try/catch

```tsx
async function handleSubmit(data: FormData) {
  try {
    const result = await mutation.mutateAsync(data)
    router.navigate({ to: `/posts/${result.id}` })
  } catch (error) {
    // Error is also stored in mutation.error
    // Handle navigation-blocking errors here
  }
}
```

## Context

- `onError` on the hook fires for every `mutate` call; `onError` on `mutate()` fires only for the last concurrent call
- Use `mutation.reset()` to clear error/data state back to idle
- `mutateAsync` throws on error — use when you need to block on the result
- `throwOnError: true` propagates errors to the nearest error boundary
- For optimistic updates, combine `onError` with rollback context from `onMutate`
- Mutation `retry` defaults to `0` — mutations don't retry unless configured
