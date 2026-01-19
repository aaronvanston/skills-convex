---
name: convex-actions
description: Best practices for Convex actions, transactions, and scheduling. Use when writing actions that call external APIs, using ctx.runQuery/ctx.runMutation, scheduling functions, or working with the Convex runtime vs Node.js runtime.
---

# Convex Actions

## Use `runAction` Only When Changing Runtime

`ctx.runAction` has overhead - it creates a separate function call while the parent action waits. Replace with plain TypeScript functions unless you need to call Node.js code from the Convex runtime.

```typescript
// Bad - unnecessary runAction
export const scrapeWebsite = action({
  args: { siteMapUrl: v.string() },
  handler: async (ctx, { siteMapUrl }) => {
    const siteMap = await fetch(siteMapUrl);
    const pages = /* parse the site map */
    await Promise.all(
      pages.map((page) =>
        ctx.runAction(internal.scrape.scrapeSinglePage, { url: page })
      )
    );
  },
});

// Good - plain TypeScript function
// convex/model/scrape.ts
export async function scrapeSinglePage(ctx: ActionCtx, { url }: { url: string }) {
  const page = await fetch(url);
  const text = /* parse the page */
  await ctx.runMutation(internal.scrape.addPage, { url, text });
}

// convex/scrape.ts
export const scrapeWebsite = action({
  args: { siteMapUrl: v.string() },
  handler: async (ctx, { siteMapUrl }) => {
    const siteMap = await fetch(siteMapUrl);
    const pages = /* parse the site map */
    await Promise.all(
      pages.map((page) => Scrape.scrapeSinglePage(ctx, { url: page }))
    );
  },
});
```

**How to find:** Search for `runAction` and check if parent and child use the same runtime.

## Avoid Sequential `ctx.runMutation` / `ctx.runQuery` in Actions

Each `ctx.runMutation` or `ctx.runQuery` runs in its own transaction. Sequential calls may be inconsistent with each other.

**Problem: Inconsistent reads**
```typescript
// Bad - team could change between queries
const team = await ctx.runQuery(internal.teams.getTeam, { teamId });
const teamOwner = await ctx.runQuery(internal.teams.getTeamOwner, { teamId });
assert(team.owner === teamOwner._id); // Could fail!

// Good - single query returns consistent data
const teamAndOwner = await ctx.runQuery(internal.teams.getTeamAndOwner, { teamId });
assert(teamAndOwner.team.owner === teamAndOwner.owner._id); // Always passes
```

**Problem: Loop without atomicity**
```typescript
// Bad - each insert is a separate transaction
for (const member of teamMembers) {
  await ctx.runMutation(internal.teams.insertUser, member);
}

// Good - single mutation, atomic transaction
await ctx.runMutation(internal.teams.insertUsers, { users: teamMembers });
```

**Exceptions:**
- Intentionally processing more data than fits in one transaction (migrations, aggregations)
- Side effects between calls (read → call external API → write result)

## Use `ctx.runQuery` and `ctx.runMutation` Sparingly in Queries/Mutations

Within queries and mutations, prefer plain TypeScript functions over `ctx.runQuery`/`ctx.runMutation`.

```typescript
// Prefer this - plain helper function
import * as Users from './model/users';

export const listMessages = query({
  args: { conversationId: v.id("conversations") },
  handler: async (ctx, { conversationId }) => {
    const user = await Users.getCurrentUser(ctx);  // Plain function
    // ...
  },
});

// Instead of this - unnecessary overhead
export const listMessages = query({
  args: { conversationId: v.id("conversations") },
  handler: async (ctx, { conversationId }) => {
    const user = await ctx.runQuery(api.users.getCurrentUser);  // Extra overhead
    // ...
  },
});
```

**Exceptions:**
- Components require `ctx.runQuery`/`ctx.runMutation`
- Partial rollback on error needs `ctx.runMutation`:

```typescript
export const trySendMessage = mutation({
  handler: async (ctx, { body, author }) => {
    try {
      await ctx.runMutation(internal.messages.sendMessage, { body, author });
    } catch (e) {
      // Record failure, rollback sendMessage writes
      await ctx.db.insert("failures", { kind: "MessageFailed", body, author, error: `${e}` });
    }
  },
});
```

## Await All Promises

Always await promises from `ctx.scheduler.runAfter`, `ctx.db.patch`, etc. Unawaited promises may silently fail or cause unexpected behavior.

**ESLint:** Use `no-floating-promises` rule from typescript-eslint.

```typescript
// Bad - missing await
ctx.scheduler.runAfter(0, internal.tasks.process, { id });
ctx.db.patch(docId, { status: "processing" });

// Good - awaited
await ctx.scheduler.runAfter(0, internal.tasks.process, { id });
await ctx.db.patch(docId, { status: "processing" });
```
