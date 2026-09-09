# sec-validate-inputs: Validate All User Inputs with Schemas

## Priority: CRITICAL

## Explanation

All data crossing the client/server boundary must be validated. Server functions accept arbitrary input from the network — never trust it. Use `.inputValidator()` with schema libraries like Zod to validate and type inputs before processing.

## Bad Example

```tsx
// No validation - trusting client input directly
export const updateUser = createServerFn({ method: 'POST' })
  .handler(async ({ data }) => {
    // data is untyped and unvalidated
    await db.users.update({
      where: { id: data.id },
      data: { role: data.role },  // Could set role to 'admin'!
    })
  })

// Type-only validation - no runtime checks
export const createPost = createServerFn({ method: 'POST' })
  .inputValidator((data: { title: string; content: string }) => data)
  .handler(async ({ data }) => {
    // TypeScript types don't exist at runtime
    // data.title could be a number, null, or a 10MB string
    await db.posts.create({ data })
  })
```

## Good Example

```tsx
import { createServerFn } from '@tanstack/react-start'
import { z } from 'zod'

// Zod schema provides runtime validation
const updateUserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email().max(255),
  // Explicitly list allowed fields - no role, no isAdmin
})

export const updateUser = createServerFn({ method: 'POST' })
  .inputValidator(updateUserSchema)
  .handler(async ({ data }) => {
    // data is validated and typed: { id: string, name: string, email: string }
    return db.users.update({
      where: { id: data.id },
      data: { name: data.name, email: data.email },
    })
  })

// Validate with constraints
const createPostSchema = z.object({
  title: z.string().min(1).max(200).trim(),
  content: z.string().min(1).max(50000),
  tags: z.array(z.string().max(50)).max(10).optional(),
})

export const createPost = createServerFn({ method: 'POST' })
  .inputValidator(createPostSchema)
  .handler(async ({ data }) => {
    return db.posts.create({ data })
  })
```

## Good Example: FormData Validation

```tsx
export const submitForm = createServerFn({ method: 'POST' })
  .inputValidator((data) => {
    if (!(data instanceof FormData)) {
      throw new Error('Expected FormData')
    }
    // Parse and validate FormData fields
    const parsed = z.object({
      name: z.string().min(1).max(100),
      email: z.string().email(),
      message: z.string().min(1).max(5000),
    }).parse({
      name: data.get('name'),
      email: data.get('email'),
      message: data.get('message'),
    })
    return parsed
  })
  .handler(async ({ data }) => {
    await sendContactEmail(data)
    return { success: true }
  })
```

## Context

- `.inputValidator()` runs before the handler — invalid data never reaches your logic
- Zod schemas can be passed directly to `.inputValidator()`
- Custom validator functions work too: `(data) => validatedData`
- Validation errors are automatically serialized to the client
- Always constrain string lengths to prevent abuse
- Explicitly list allowed fields — never spread raw input into database queries
- Share schemas between client and server for consistent validation (see `file-shared-validation`)
