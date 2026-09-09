# load-loader-deps: Define loaderDeps for Cache Control

## Priority: HIGH

## Explanation

Search params are not directly available in the loader function. Use `loaderDeps` to explicitly declare which search params (or other reactive values) the loader depends on. This controls the cache key — the loader only re-runs when the declared dependencies change, not on every search param change. Returning the entire search object causes unnecessary cache invalidation.

## Bad Example

```tsx
// Returning the entire search object — cache invalidates on ANY param change
export const Route = createFileRoute('/products')({
  validateSearch: z.object({
    page: z.number().catch(1),
    sort: z.enum(['name', 'price']).catch('name'),
    view: z.enum(['grid', 'list']).catch('grid'), // UI-only param
  }),
  loaderDeps: ({ search }) => search, // BAD: includes 'view' which doesn't affect data
  loader: ({ deps }) => fetchProducts({ page: deps.page, sort: deps.sort }),
})
// Changing 'view' from 'grid' to 'list' unnecessarily refetches data
```

## Good Example

```tsx
export const Route = createFileRoute('/products')({
  validateSearch: z.object({
    page: z.number().catch(1),
    sort: z.enum(['name', 'price']).catch('name'),
    view: z.enum(['grid', 'list']).catch('grid'),
  }),
  // Only declare the params the loader actually uses
  loaderDeps: ({ search: { page, sort } }) => ({ page, sort }),
  loader: ({ deps: { page, sort } }) =>
    fetchProducts({ page, sort }),
  component: ProductsPage,
})

function ProductsPage() {
  const { view } = Route.useSearch()
  const products = Route.useLoaderData()
  // Changing 'view' does NOT refetch — only 'page' and 'sort' trigger reloads
  return <ProductList products={products} view={view} />
}
```

## Good Example: With TanStack Query

```tsx
const productsQueryOptions = (deps: { page: number; sort: string }) =>
  queryOptions({
    queryKey: ['products', deps],
    queryFn: () => fetchProducts(deps),
  })

export const Route = createFileRoute('/products')({
  validateSearch: productSearchSchema,
  loaderDeps: ({ search: { page, sort } }) => ({ page, sort }),
  loader: ({ deps, context: { queryClient } }) =>
    queryClient.ensureQueryData(productsQueryOptions(deps)),
})
```

## How the Cache Key Works

```tsx
// Cache key = route path + loaderDeps return value
// /products with { page: 1, sort: 'name' } → one cache entry
// /products with { page: 2, sort: 'name' } → different cache entry
// /products with { page: 1, sort: 'price' } → different cache entry
// /products with { page: 1, sort: 'name' } → reuses first cache entry
```

## Context

- `loaderDeps` receives the validated search params from `validateSearch`
- Only include dependencies the loader actually needs for fetching data
- UI-only search params (view mode, collapsed state) should be excluded from `loaderDeps`
- The return value of `loaderDeps` is available as `deps` in the loader function
- Without `loaderDeps`, the cache key is based only on the route path and path params
- `loaderDeps` also controls SWR behavior — the loader re-runs when deps change
