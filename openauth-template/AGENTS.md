# AGENTS.md

This guide helps AI agents review, maintain, and contribute to the OpenAuth template. It documents patterns, common issues, and testing strategies learned from real PR reviews.

## Quick Reference

### Essential Commands

```bash
# Typecheck the code
pnpm run check

# Generate binding types from wrangler.json
pnpm run cf-typegen

# Deploy (with DB migrations)
pnpm run deploy

# Run locally
pnpm run dev
```

### PR Review Checklist

1. **package.json** - Has description and cloudflare metadata
2. **README.md** - Has dashboard markers and deploy button at top
3. **TypeScript** - `pnpm run check` passes (strict mode enabled)
4. **Database** - D1 migrations exist if schema changed
5. **Bindings** - KV and D1 bindings defined in wrangler.json

---

## Code Style Guidelines

### Imports

- Use `import { x } from "y"` (named imports)
- Group imports: external libs first, then internal
- No default exports except for the Worker handler

### Formatting

- 2-space indentation, no tabs
- Semicolons required
- Single quotes for strings

### Types

- Strict mode enabled in tsconfig
- Use `interface` for objects, `type` for unions/utility types
- Always type env bindings (generated via cf-typegen)
- Use `satisfies ExportedHandler<Env>` for the handler

### Naming

- `camelCase` for variables/functions
- `PascalCase` for types/interfaces
- `UPPER_CASE` for constants
- Database binding: `AUTH_DB` (uppercase)
- KV binding: `AUTH_STORAGE` (uppercase)

### Error Handling

- Throw explicit errors with meaningful messages
- Validate DB results before use
- No unhandled promise rejections

---

## Database Patterns

### Schema Changes

1. Create migration: `wrangler d1 migrations create AUTH_DB <name>`
2. Write SQL in migration file
3. Test locally: `wrangler d1 migrations apply AUTH_DB --local`
4. Predeploy hook runs migrations automatically

### User Lookup Pattern

```typescript
async function getOrCreateUser(env: Env, email: string): Promise<string> {
	const result = await env.AUTH_DB.prepare(
		`
    INSERT INTO user (email) VALUES (?)
    ON CONFLICT (email) DO UPDATE SET email = email
    RETURNING id;
  `,
	)
		.bind(email)
		.first<{ id: string }>();

	if (!result) throw new Error(`Unable to process user: ${email}`);
	return result.id;
}
```

---

## Common Issues

### Missing wrangler types

Run `pnpm run cf-typegen` after changing bindings

### Database binding not working

Check wrangler.json has correct database_name and database_id

### Console.log in production

Remove before deploying (or use proper logging)

### Missing nodejs_compat

Required for valibot - already in wrangler.json compatibility_flags

---

## Existing Cursor Rules

None present in codebase.
