# TanStack Agent Skills

Comprehensive best practices for building applications with the TanStack ecosystem. These skills provide AI coding agents with structured guidelines for TanStack Query, TanStack Router, and TanStack Start.

Skills follow the [Agent Skills](https://agentskills.io/) format.

## Available Skills

### [tanstack-query](./skills/tanstack-query/)

Best practices for data fetching, caching, mutations, and server state management with TanStack Query (React Query).

**Use when:**
- Fetching or caching server data with `useQuery` or `useMutation`
- Setting up React Query for a new project
- Reviewing data fetching patterns for performance issues

**32+ rules across 8 categories:**
- Query Keys (CRITICAL) — Proper key structure and organization
- Caching (CRITICAL) — staleTime, gcTime, and invalidation patterns
- Mutations (HIGH) — Optimistic updates, error handling
- Error Handling (HIGH) — Error boundaries, retry logic
- Prefetching (MEDIUM) — Intent-based and route prefetching
- Infinite Queries (MEDIUM) — Pagination patterns
- SSR Integration (MEDIUM) — Hydration and dehydration
- Performance (LOW) — Select transforms, memoization

### [tanstack-router](./skills/tanstack-router/)

Best practices for type-safe routing, data loading, search params, and navigation with TanStack Router.

**Use when:**
- Setting up routing for a React app
- Working with search params, loaders, or code splitting
- Configuring type-safe navigation

**30+ rules across 8 categories:**
- Type Safety (CRITICAL) — Router registration, type narrowing
- Route Organization (CRITICAL) — File-based routing, route trees
- Data Loading (HIGH) — Loaders, Query integration
- Search Params (HIGH) — Validation, type inheritance
- Navigation (MEDIUM) — Link component, active states
- Code Splitting (MEDIUM) — Lazy routes, critical path
- Preloading (MEDIUM) — Intent-based preloading
- Route Context (LOW) — Dependency injection

### [tanstack-start](./skills/tanstack-start/)

Best practices for full-stack React applications with TanStack Start, including server functions, middleware, SSR, and authentication.

**Use when:**
- Building full-stack apps with server functions
- Implementing authentication or middleware
- Configuring SSR, streaming, or deployment

**29+ rules across 8 categories:**
- Server Functions (CRITICAL) — createServerFn patterns
- Security (CRITICAL) — Input validation, CSRF protection
- Middleware (HIGH) — Request/function middleware
- Authentication (HIGH) — Sessions, route protection
- SSR (MEDIUM) — Hydration safety, streaming
- Error Handling (MEDIUM) — Server errors, redirects
- File Organization (LOW) — Code separation patterns
- Deployment (LOW) — Environment config

### [tanstack-integration](./skills/tanstack-integration/)

Best practices for integrating TanStack Query with TanStack Router and TanStack Start together.

**Use when:**
- Combining Query + Router in the same app
- Setting up SSR with data prefetching
- Wiring loaders to query cache

**13 rules across 4 categories:**
- Setup (CRITICAL) — QueryClient context, provider wrapping
- Data Flow (HIGH) — Loader + Query patterns
- Caching (MEDIUM) — Single source of truth
- SSR (LOW) — Dehydration/hydration

## Installation

```bash
npx skills add DeckardGer/tanstack-agent-skills
```

Or manually add to your project's `.cursor/` or Claude Code configuration.

## Usage

Skills are automatically available once installed. The agent will use them when relevant tasks are detected.

## Skill Structure

Each skill follows the [Agent Skills](https://agentskills.io/) standard:

```
skills/
├── tanstack-query/
│   ├── SKILL.md          # Main skill instructions
│   └── rules/            # Individual rule files
│       ├── qk-array-structure.md
│       ├── cache-stale-time.md
│       └── ...
├── tanstack-router/
│   ├── SKILL.md
│   └── rules/
├── tanstack-start/
│   ├── SKILL.md
│   └── rules/
└── tanstack-integration/
    ├── SKILL.md
    └── rules/
```

## Rule File Structure

Each rule file contains:

1. **Priority** — CRITICAL, HIGH, MEDIUM, or LOW
2. **Explanation** — Why this pattern matters
3. **Bad Example** — Anti-pattern to avoid with explanation
4. **Good Example** — Recommended implementation
5. **Context** — When to apply or skip the rule

## Contributing

Contributions welcome! Please ensure:

- Rules are practical and battle-tested
- Examples are clear and runnable
- Priority levels are appropriate
- Context helps agents decide applicability

## Agent Compatibility

Works with any AI coding agent that supports `AGENTS.md`, including Claude Code, Codex, Opencode, and others.

## Resources

- [TanStack Query Docs](https://tanstack.com/query)
- [TanStack Router Docs](https://tanstack.com/router)
- [TanStack Start Docs](https://tanstack.com/start)
- [Agent Skills Format](https://agentskills.io/)

## License

MIT
