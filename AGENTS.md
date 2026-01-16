# Agent Guidelines

This document provides instructions for AI coding agents working with the TanStack Agent Skills repository.

## Repository Structure

```
tanstack-agent-skills/
├── skills/
│   ├── tanstack-query/
│   │   ├── SKILL.md
│   │   └── rules/
│   ├── tanstack-router/
│   │   ├── SKILL.md
│   │   └── rules/
│   ├── tanstack-start/
│   │   ├── SKILL.md
│   │   └── rules/
│   └── tanstack-integration/
│       ├── SKILL.md
│       └── rules/
├── README.md
├── AGENTS.md
└── CLAUDE.md
```

## Skill Format

### SKILL.md Structure

Each skill's main file uses YAML frontmatter:

```yaml
---
name: skill-name
description: Brief description of what this skill covers
---
```

Followed by markdown sections:
- When to Apply
- Rule Categories by Priority
- Quick Reference
- How to Use

### Rule File Naming

Rules use kebab-case with category prefixes:
- `qk-` — Query keys
- `cache-` — Caching
- `mut-` — Mutations
- `err-` — Error handling
- `pf-` — Prefetching
- `inf-` — Infinite queries
- `ssr-` — SSR patterns
- `perf-` — Performance
- `ts-` — Type safety
- `org-` — Organization
- `load-` — Data loading
- `search-` — Search params
- `nav-` — Navigation
- `split-` — Code splitting
- `preload-` — Preloading
- `ctx-` — Context
- `sf-` — Server functions
- `sec-` — Security
- `mw-` — Middleware
- `auth-` — Authentication
- `file-` — File organization
- `deploy-` — Deployment
- `setup-` — Setup
- `flow-` — Data flow

### Rule File Structure

```markdown
# rule-id: Rule Title

## Priority: CRITICAL | HIGH | MEDIUM | LOW

## Explanation

Why this pattern matters and what problem it solves.

## Bad Example

\`\`\`tsx
// Code showing the anti-pattern
// With comments explaining why it's bad
\`\`\`

## Good Example

\`\`\`tsx
// Code showing the recommended pattern
// With comments explaining the approach
\`\`\`

## Context

- When to apply this rule
- When to skip this rule
- Related considerations
```

## Skill Activation

Skills should activate when agents detect:

### tanstack-query
- Data fetching code (`useQuery`, `useMutation`)
- React Query imports
- Cache management patterns
- Server state handling

### tanstack-router
- Route definitions
- Navigation code
- Search params handling
- Code splitting setup

### tanstack-start
- Server functions (`createServerFn`)
- Middleware setup
- SSR configuration
- Authentication flows

### tanstack-integration
- Combined Query + Router usage
- Full-stack TanStack applications
- SSR with data prefetching

## Priority Levels

| Priority | Meaning |
|----------|---------|
| CRITICAL | Must follow - violations cause bugs or security issues |
| HIGH | Strongly recommended - improves reliability and performance |
| MEDIUM | Good practice - enhances maintainability and UX |
| LOW | Nice to have - optimization and polish |

## Context Efficiency

To minimize token usage:
- Keep individual rule files under 200 lines
- Use concise code examples
- Reference external docs for deep dives
- Only load relevant rules based on context

## Adding New Rules

1. Identify the category and create appropriate prefix
2. Write clear explanation of the problem/solution
3. Provide realistic bad and good examples
4. Include context for when to apply
5. Assign appropriate priority level
6. Update SKILL.md quick reference table
