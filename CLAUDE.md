# Claude Code Guidelines

Instructions for Claude Code when working with TanStack applications.

## Skill Activation

Load the appropriate skill when you detect:

### TanStack Query Context
- `@tanstack/react-query` imports
- `useQuery`, `useMutation`, `useInfiniteQuery` usage
- `QueryClient`, `QueryClientProvider`
- Data fetching patterns

**Load:** `skills/tanstack-query/SKILL.md`

### TanStack Router Context
- `@tanstack/react-router` imports
- `createFileRoute`, `createRootRoute` usage
- Route definitions, loaders, search params
- `Link`, `useNavigate`, `useParams` usage

**Load:** `skills/tanstack-router/SKILL.md`

### TanStack Start Context
- `@tanstack/react-start` imports
- `createServerFn` usage
- Server functions, middleware
- Full-stack patterns

**Load:** `skills/tanstack-start/SKILL.md`

### Integration Context
- Both Query and Router being used together
- SSR with data prefetching
- `ensureQueryData` in loaders

**Load:** `skills/tanstack-integration/SKILL.md`

## Applying Rules

When generating or reviewing code:

1. **Check CRITICAL rules first** — These prevent bugs
2. **Apply HIGH priority patterns** — Improve reliability
3. **Consider MEDIUM patterns** — Better UX
4. **Suggest LOW patterns** — When optimizing

## Common Scenarios

### New TanStack Query Setup
1. Load tanstack-query skill
2. Apply `qk-factory-pattern` for query organization
3. Apply `cache-stale-time` and `cache-gc-time`
4. Set up error boundaries per `err-error-boundaries`

### New TanStack Router Setup
1. Load tanstack-router skill
2. Apply `ts-register-router` for type safety
3. Apply `org-file-based-routing` conventions
4. Configure preloading per `preload-intent`

### Adding Server Functions
1. Load tanstack-start skill
2. Apply `sf-create-server-fn` patterns
3. Always apply `sf-input-validation`
4. Consider `mw-request-middleware` for auth

### Full-Stack Data Flow
1. Load tanstack-integration skill
2. Apply `setup-query-client-context`
3. Follow `flow-loader-query-pattern`
4. Configure `cache-single-source`

## Code Review Checklist

When reviewing TanStack code:

- [ ] Query keys are arrays (`qk-array-structure`)
- [ ] All dependencies in query keys (`qk-include-dependencies`)
- [ ] Router types registered (`ts-register-router`)
- [ ] Loaders use ensureQueryData (`load-ensure-query-data`)
- [ ] Search params validated (`search-validation`)
- [ ] Server function inputs validated (`sf-input-validation`)
- [ ] Mutations invalidate queries (`mut-invalidate-queries`)
- [ ] Error boundaries in place (`err-error-boundaries`)

## Priority Shortcuts

Use these when discussing trade-offs:

- **"This is CRITICAL"** — Must fix, causes bugs
- **"This is HIGH priority"** — Should fix, improves reliability
- **"Consider this pattern"** — MEDIUM, nice improvement
- **"Optimization opportunity"** — LOW, polish

## Related Documentation

Reference these for deeper information:
- https://tanstack.com/query/latest/docs
- https://tanstack.com/router/latest/docs
- https://tanstack.com/start/latest/docs
