# nav-active-states: Configure Active Link States

## Priority: MEDIUM

## Explanation

The `<Link>` component automatically tracks whether it matches the current route and provides `activeProps`, `inactiveProps`, and a `data-status="active"` attribute for styling. Configure `activeOptions` to control matching behavior — exact path, search param inclusion, and hash matching.

## Bad Example

```tsx
// Manually tracking active state — duplicates router logic, error-prone
function NavLink({ to, children }) {
  const location = useLocation()
  const isActive = location.pathname === to // Doesn't handle nested routes

  return (
    <Link to={to} className={isActive ? 'active' : ''}>
      {children}
    </Link>
  )
}
```

## Good Example

```tsx
// Using built-in activeProps/inactiveProps
function Navigation() {
  return (
    <nav>
      <Link
        to="/dashboard"
        activeProps={{ className: 'font-bold text-blue-600' }}
        inactiveProps={{ className: 'text-gray-600' }}
      >
        Dashboard
      </Link>

      <Link
        to="/posts"
        activeProps={{ style: { fontWeight: 'bold' } }}
      >
        Posts
      </Link>
    </nav>
  )
}
```

## Good Example: CSS with data-status

```tsx
// Style active links purely with CSS using the data-status attribute
<Link to="/dashboard" className="nav-link">Dashboard</Link>
```

```css
.nav-link {
  color: gray;
}
.nav-link[data-status='active'] {
  color: blue;
  font-weight: bold;
}
```

## Good Example: activeOptions

```tsx
// Exact matching — only active on /dashboard, not /dashboard/settings
<Link
  to="/dashboard"
  activeOptions={{ exact: true }}
  activeProps={{ className: 'active' }}
>
  Dashboard
</Link>

// Include search params in matching
<Link
  to="/products"
  search={{ category: 'electronics' }}
  activeOptions={{ includeSearch: true }} // Default is true
  activeProps={{ className: 'active' }}
>
  Electronics
</Link>

// Include hash in matching
<Link
  to="/docs"
  hash="installation"
  activeOptions={{ includeHash: true }}
  activeProps={{ className: 'active' }}
>
  Installation
</Link>
```

## Good Example: Render Prop for isActive

```tsx
<Link to="/posts">
  {({ isActive }) => (
    <span className={isActive ? 'text-blue-600' : 'text-gray-400'}>
      Posts
      {isActive && <ChevronIcon />}
    </span>
  )}
</Link>
```

## activeOptions Reference

| Option | Default | Description |
|---|---|---|
| `exact` | `false` | Only active on exact path match (not prefix) |
| `includeSearch` | `true` | Check if link's search params are a subset of current |
| `includeHash` | `false` | Check if link's hash matches current hash |
| `explicitUndefined` | `false` | `undefined` values in search must NOT be present |

## Context

- By default, a link is active when the current path starts with the link's path (prefix matching)
- Set `exact: true` for index/home links to avoid them always being active
- `activeProps` and `inactiveProps` merge styles and concatenate classNames with the base props
- `data-status="active"` is set on the rendered `<a>` element for CSS-only styling
- Use the render prop `{({ isActive }) => ...}` for conditional child rendering
