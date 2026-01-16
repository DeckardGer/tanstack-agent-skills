---
name: tanstack-integration-best-practices
description: Best practices for integrating TanStack Query with TanStack Router and TanStack Start. Patterns for full-stack data flow, SSR, and caching coordination.
---

# TanStack Integration Best Practices

Guidelines for integrating TanStack Query, Router, and Start together effectively. These patterns ensure optimal data flow, caching coordination, and type safety across the stack.

## When to Apply

- Setting up a new TanStack Start project
- Integrating TanStack Query with TanStack Router
- Configuring SSR with query hydration
- Coordinating caching between router and query
- Setting up type-safe data fetching patterns

## Rule Categories by Priority

| Priority | Category | Rules | Impact |
|----------|----------|-------|--------|
| CRITICAL | Setup | 3 rules | Foundational configuration |
| HIGH | Data Flow | 4 rules | Correct data fetching patterns |
| MEDIUM | Caching | 3 rules | Performance optimization |
| LOW | SSR | 3 rules | Server rendering patterns |

## Quick Reference

### Setup (Prefix: `setup-`)

- `setup-query-client-context` — Pass QueryClient through router context
- `setup-provider-wrapping` — Correctly wrap with QueryClientProvider
- `setup-stale-time-coordination` — Coordinate staleTime between router and query

### Data Flow (Prefix: `flow-`)

- `flow-loader-query-pattern` — Use loaders with ensureQueryData
- `flow-suspense-query-component` — Use useSuspenseQuery in components
- `flow-mutations-invalidation` — Coordinate mutations with query invalidation
- `flow-server-functions-queries` — Use server functions for query functions

### Caching (Prefix: `cache-`)

- `cache-single-source` — Let TanStack Query manage caching
- `cache-preload-coordination` — Coordinate preloading between router and query
- `cache-invalidation-patterns` — Unified invalidation patterns

### SSR (Prefix: `ssr-`)

- `ssr-dehydrate-hydrate` — Configure dehydration/hydration
- `ssr-per-request-client` — Create QueryClient per request
- `ssr-streaming-queries` — Handle streaming with queries

## How to Use

Each rule file in the `rules/` directory contains:
1. **Explanation** — Why this pattern matters
2. **Bad Example** — Anti-pattern to avoid
3. **Good Example** — Recommended implementation
4. **Context** — When to apply or skip this rule

## Full Reference

See individual rule files in `rules/` directory for detailed guidance and code examples.
