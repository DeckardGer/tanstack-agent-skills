# file-shared-validation: Share Validation Schemas Between Client and Server

## Priority: LOW

## Explanation

Validation schemas (Zod, Valibot, etc.) are safe to share between client and server. Define them in dedicated files and reuse them in server function `.inputValidator()`, form validation, and search param validation. This ensures consistent validation rules and types on both sides.

## Bad Example

```tsx
// Duplicating validation logic
// server
export const createUser = createServerFn({ method: 'POST' })
  .inputValidator((data: { name: string; email: string }) => {
    if (!data.name || data.name.length > 100) throw new Error('Invalid name')
    if (!data.email?.includes('@')) throw new Error('Invalid email')
    return data
  })
  .handler(async ({ data }) => { /* ... */ })

// client — different rules, easy to drift
function CreateUserForm() {
  const validate = (values) => {
    const errors = {}
    if (!values.name) errors.name = 'Required'
    if (values.name?.length > 50) errors.name = 'Too long'  // Different max!
    if (!values.email) errors.email = 'Required'
    return errors
  }
}
```

## Good Example

```tsx
// utils/schemas/user.schema.ts — single source of truth
import { z } from 'zod'

export const createUserSchema = z.object({
  name: z.string().min(1, 'Name is required').max(100, 'Name is too long').trim(),
  email: z.string().email('Invalid email address').max(255),
  role: z.enum(['user', 'editor']).default('user'),
})

export type CreateUserInput = z.infer<typeof createUserSchema>

export const updateUserSchema = createUserSchema.partial().required({ name: true })

export type UpdateUserInput = z.infer<typeof updateUserSchema>
```

```tsx
// utils/users.functions.ts — server uses the same schema
import { createServerFn } from '@tanstack/react-start'
import { createUserSchema } from './schemas/user.schema'

export const createUser = createServerFn({ method: 'POST' })
  .inputValidator(createUserSchema)
  .handler(async ({ data }) => {
    // data is validated and typed as CreateUserInput
    return db.users.create({ data })
  })
```

```tsx
// components/CreateUserForm.tsx — client uses the same schema
import { createUserSchema, type CreateUserInput } from '../utils/schemas/user.schema'

function CreateUserForm() {
  const [errors, setErrors] = useState<Record<string, string>>({})

  const handleSubmit = async (formData: FormData) => {
    const raw = Object.fromEntries(formData)
    const result = createUserSchema.safeParse(raw)

    if (!result.success) {
      // Show validation errors before hitting the server
      setErrors(result.error.flatten().fieldErrors)
      return
    }

    await createUser({ data: result.data })
  }
}
```

## Good Example: Search Param Validation

```tsx
// utils/schemas/search.schema.ts
import { z } from 'zod'

export const postsSearchSchema = z.object({
  q: z.string().optional(),
  page: z.number().default(1),
  sort: z.enum(['newest', 'oldest', 'popular']).default('newest'),
})

// routes/posts.tsx — reuse in route search validation
export const Route = createFileRoute('/posts')({
  validateSearch: postsSearchSchema,
  loaderDeps: ({ search }) => ({ search }),
  loader: ({ deps }) => getPosts({ data: deps.search }),
})
```

## Context

- Validation schemas contain no server-side code — safe to import on the client
- Zod schemas can be passed directly to `.inputValidator()` on server functions
- Schemas provide both runtime validation and TypeScript types via `z.infer<typeof schema>`
- Client-side validation improves UX with instant feedback before server round-trip
- Server-side validation is still required — client validation can be bypassed
- Put schemas in a `schemas/` directory or use `.schema.ts` suffix
- Keep schemas close to their domain (user.schema.ts, post.schema.ts)
