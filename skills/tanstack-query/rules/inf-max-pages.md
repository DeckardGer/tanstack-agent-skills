# inf-max-pages: Consider maxPages for Large Datasets

## Priority: MEDIUM

## Explanation

By default, `useInfiniteQuery` keeps all fetched pages in memory and refetches all of them on stale data refresh. For large datasets, this causes excessive memory usage and slow refetches. Use the `maxPages` option to cap how many pages are retained in the cache.

## Bad Example

```tsx
// No maxPages — all pages accumulate in memory
const { data } = useInfiniteQuery({
  queryKey: ['feed'],
  queryFn: fetchFeedPage,
  initialPageParam: 0,
  getNextPageParam: (lastPage) => lastPage.nextCursor,
  // User scrolls through 50 pages → 50 pages in memory
  // Background refetch fetches all 50 pages sequentially
})
```

## Good Example

```tsx
const { data, fetchNextPage, fetchPreviousPage } = useInfiniteQuery({
  queryKey: ['feed'],
  queryFn: fetchFeedPage,
  initialPageParam: 0,
  getNextPageParam: (lastPage) => lastPage.nextCursor,
  getPreviousPageParam: (firstPage) => firstPage.prevCursor,
  maxPages: 5, // Only keep 5 pages in cache at a time
})

// When maxPages is reached:
// - Fetching next page drops the first page
// - Fetching previous page drops the last page
// - Background refetch only refetches the retained pages
```

## Good Example: Offset-Based Pagination with maxPages

```tsx
const { data } = useInfiniteQuery({
  queryKey: ['products'],
  queryFn: ({ pageParam }) => fetchProducts({ page: pageParam, limit: 20 }),
  initialPageParam: 1,
  getNextPageParam: (lastPage, allPages, lastPageParam) => {
    if (lastPage.length === 0) return undefined
    return lastPageParam + 1
  },
  getPreviousPageParam: (firstPage, allPages, firstPageParam) => {
    if (firstPageParam <= 1) return undefined
    return firstPageParam - 1
  },
  maxPages: 3, // Keep 3 pages (60 items) in memory
})
```

## Context

- When `maxPages` is set, `getPreviousPageParam` **must** also be defined so evicted pages can be re-fetched
- Pages are evicted in FIFO order — fetching forward drops the earliest page, fetching backward drops the latest
- Background refetches only refetch the retained pages, not all historically fetched pages
- Without `maxPages`, refetching fetches all pages sequentially from the first one
- Choose `maxPages` based on memory constraints and typical scroll depth
- For virtualized lists, `maxPages` pairs well with scroll-based page eviction
