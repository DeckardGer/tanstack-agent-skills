# inf-loading-guards: Check isFetchingNextPage Before Fetching More

## Priority: HIGH

## Explanation

Always check `isFetching` or `isFetchingNextPage` before calling `fetchNextPage()`. Without this guard, rapid triggers (scroll events, button clicks) can fire multiple concurrent fetches, potentially overwriting background data refreshes and causing data inconsistencies.

## Bad Example

```tsx
function ProjectList() {
  const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
    queryKey: ['projects'],
    queryFn: fetchProjects,
    initialPageParam: 0,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  })

  return (
    <div>
      {data?.pages.map((page) =>
        page.items.map((item) => <ProjectCard key={item.id} item={item} />)
      )}
      {/* No loading guard — can trigger multiple fetches */}
      {hasNextPage && (
        <button onClick={() => fetchNextPage()}>Load More</button>
      )}
    </div>
  )
}

// Infinite scroll without guard — fires on every scroll event
<List onEndReached={() => fetchNextPage()} />
```

## Good Example

```tsx
function ProjectList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetching,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['projects'],
    queryFn: fetchProjects,
    initialPageParam: 0,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  })

  return (
    <div>
      {data?.pages.map((page, i) => (
        <React.Fragment key={i}>
          {page.items.map((item) => (
            <ProjectCard key={item.id} item={item} />
          ))}
        </React.Fragment>
      ))}

      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetching}
      >
        {isFetchingNextPage
          ? 'Loading more...'
          : hasNextPage
            ? 'Load More'
            : 'Nothing more to load'}
      </button>

      {/* Background refetch indicator (not pagination) */}
      {isFetching && !isFetchingNextPage && <Spinner />}
    </div>
  )
}
```

## Good Example: Infinite Scroll

```tsx
<FlatList
  data={data?.pages.flatMap((page) => page.items)}
  onEndReached={() => {
    if (hasNextPage && !isFetching) {
      fetchNextPage()
    }
  }}
  onEndReachedThreshold={0.5}
  ListFooterComponent={isFetchingNextPage ? <Spinner /> : null}
/>
```

## Context

- `isFetchingNextPage` is true only during pagination fetches, not background refetches
- `isFetching` is true during any fetch (pagination or background) — use this for the guard
- `hasNextPage` is `false` when `getNextPageParam` returns `undefined` or `null`
- Disable the trigger element during fetches to prevent duplicate requests
- Distinguish between pagination loading and background refresh in the UI
