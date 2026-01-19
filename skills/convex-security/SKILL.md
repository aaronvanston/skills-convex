---
name: convex-security
description: Security best practices for Convex functions including argument validation, access control, and internal functions. Use when writing public queries/mutations/actions, implementing authentication, adding authorization checks, or reviewing Convex functions for security vulnerabilities.
---

# Convex Security

## Use Argument Validators for All Public Functions

Public functions can be called by anyone. Argument validators ensure you receive expected data types and prevent malicious input.

```typescript
// Bad - no validation, client could pass anything
export const updateMovie = mutation({
  handler: async (
    ctx,
    { id, update }: { id: Id<"movies">; update: Pick<Doc<"movies">, "title" | "director"> }
  ) => {
    await ctx.db.patch("movies", id, update);
  },
});

// Good - validated arguments
export const updateMovie = mutation({
  args: {
    id: v.id("movies"),
    update: v.object({
      title: v.string(),
      director: v.string(),
    }),
  },
  handler: async (ctx, { id, update }) => {
    await ctx.db.patch("movies", id, update);
  },
});
```

**How to find:** Search for `query`, `mutation`, `action` without `args:` property.

**ESLint:** Use `@convex-dev/require-argument-validators` rule.

**HTTP actions:** Use Zod or similar to validate request shape.

## Use Access Control for All Public Functions

Every public function needs authentication and/or authorization checks.

**Key principles:**
- Use `ctx.auth.getUserIdentity()` for auth checks (cannot be spoofed)
- Never use client-provided email/username for access control
- Prefer granular functions (`setTeamOwner`) over generic ones (`updateTeam`)

```typescript
// Bad - no checks
export const updateTeam = mutation({
  args: { id: v.id("teams"), update: v.object({ name: v.optional(v.string()) }) },
  handler: async (ctx, { id, update }) => {
    await ctx.db.patch("teams", id, update);
  },
});

// Bad - uses spoofable email
export const updateTeam = mutation({
  args: { id: v.id("teams"), email: v.string(), /* ... */ },
  handler: async (ctx, { id, email, update }) => {
    const teamMembers = /* load team members */
    if (!teamMembers.some((m) => m.email === email)) {
      throw new Error("Unauthorized");
    }
    await ctx.db.patch("teams", id, update);
  },
});

// Good - uses ctx.auth
export const updateTeam = mutation({
  args: { id: v.id("teams"), update: v.object({ name: v.optional(v.string()) }) },
  handler: async (ctx, { id, update }) => {
    const user = await ctx.auth.getUserIdentity();
    if (user === null) throw new Error("Unauthorized");

    const isTeamMember = /* check if user is a member */
    if (!isTeamMember) throw new Error("Unauthorized");

    await ctx.db.patch("teams", id, update);
  },
});
```

**Patterns:**
- Custom functions like `authenticatedQuery` for consistent auth checks
- Row Level Security (RLS) for document-level access control
- Helper functions: `isTeamMember`, `isTeamAdmin`, `loadTeam`

## Only Schedule and `ctx.run*` Internal Functions

Functions called only within Convex should be `internal`. Public functions require careful security auditing.

```typescript
// Bad - using api for scheduled/internal calls
crons.daily(
  "send daily reminder",
  { hourUTC: 17, minuteUTC: 30 },
  api.messages.sendMessage,  // Public function!
  { author: "System", body: "Share your daily update!" }
);

// Good - use internal function
export const sendInternalMessage = internalMutation({
  args: { body: v.string(), author: v.string() },
  handler: async (ctx, { body, author }) => {
    await sendMessageHelper(ctx, { body, author });
  },
});

crons.daily(
  "send daily reminder",
  { hourUTC: 17, minuteUTC: 30 },
  internal.messages.sendInternalMessage,
  { author: "System", body: "Share your daily update!" }
);
```

**How to find:** Search for `ctx.runQuery`, `ctx.runMutation`, `ctx.runAction`, `ctx.scheduler` and ensure they use `internal.foo.bar` not `api.foo.bar`.

**Tip:** Never import `api` from `_generated/api.ts` in your Convex functions directory.

## Include Table Name in `ctx.db` Functions

Since convex NPM v1.31.0, `ctx.db` functions accept table name as first argument. Always include it.

```typescript
// Bad - no table name
await ctx.db.get(movieId);
await ctx.db.patch(movieId, { title: "Whiplash" });
await ctx.db.replace(movieId, { title: "Whiplash", director: "Damien Chazelle", votes: 0 });
await ctx.db.delete(movieId);

// Good - table name included
await ctx.db.get("movies", movieId);
await ctx.db.patch("movies", movieId, { title: "Whiplash" });
await ctx.db.replace("movies", movieId, { title: "Whiplash", director: "Damien Chazelle", votes: 0 });
await ctx.db.delete("movies", movieId);
```

**ESLint:** Use `@convex-dev/explicit-table-ids` rule with autofix.

**Codemod:** Use `@convex-dev/codemod` for automatic migration.
