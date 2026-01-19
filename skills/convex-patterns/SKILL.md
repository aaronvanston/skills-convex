---
name: convex-patterns
description: Code organization patterns and TypeScript best practices for Convex. Use when structuring a Convex project, writing helper functions, working with types like QueryCtx/MutationCtx/ActionCtx, or organizing code in a convex/model directory.
---

# Convex Patterns

## Use Helper Functions for Shared Code

Most logic should be plain TypeScript functions. The `query`, `mutation`, and `action` wrappers should be thin wrappers calling helper functions.

**Recommended structure:**
```
convex/
├── _generated/
├── model/           # Business logic lives here
│   ├── users.ts
│   ├── conversations.ts
│   └── teams.ts
├── conversations.ts # Thin wrappers defining public API
├── users.ts
└── schema.ts
```

**Example helper functions:**
```typescript
// convex/model/users.ts
import { QueryCtx } from '../_generated/server';

export async function getCurrentUser(ctx: QueryCtx) {
  const userIdentity = await ctx.auth.getUserIdentity();
  if (userIdentity === null) throw new Error("Unauthorized");

  const user = /* query ctx.db to load the user */
  const userSettings = /* load related documents */
  return { user, settings: userSettings };
}
```

```typescript
// convex/model/conversations.ts
import { QueryCtx, MutationCtx } from '../_generated/server';
import { Id, Doc } from '../_generated/dataModel';
import * as Users from './users';

export async function ensureHasAccess(
  ctx: QueryCtx,
  { conversationId }: { conversationId: Id<"conversations"> }
) {
  const user = await Users.getCurrentUser(ctx);
  const conversation = await ctx.db.get("conversations", conversationId);
  if (conversation === null || !conversation.members.includes(user._id)) {
    throw new Error("Unauthorized");
  }
  return conversation;
}

export async function listMessages(
  ctx: QueryCtx,
  { conversationId }: { conversationId: Id<"conversations"> }
) {
  await ensureHasAccess(ctx, { conversationId });
  const messages = /* query ctx.db */
  return messages;
}
```

**Thin public API:**
```typescript
// convex/conversations.ts
import * as Conversations from './model/conversations';

export const listMessages = internalQuery({
  args: { conversationId: v.id("conversations") },
  handler: async (ctx, { conversationId }) => {
    return Conversations.listMessages(ctx, { conversationId });
  },
});
```

## TypeScript Types

### Context Types

```typescript
import { QueryCtx, MutationCtx, ActionCtx, DatabaseReader, DatabaseWriter } from "./_generated/server";
import { Auth, StorageReader, StorageWriter, StorageActionWriter } from "convex/server";

// MutationCtx also satisfies QueryCtx interface
export function myReadHelper(ctx: QueryCtx, id: Id<"channels">) { /* ... */ }
export function myActionHelper(ctx: ActionCtx, doc: Doc<"messages">) { /* ... */ }
```

### Document and ID Types

```typescript
import { Doc, Id } from "./_generated/dataModel";

function Channel(props: { channelId: Id<"channels"> }) { /* ... */ }
function MessagesView(props: { message: Doc<"messages"> }) { /* ... */ }
```

### Infer Types from Validators

```typescript
import { Infer, v } from "convex/values";

export const courseValidator = v.union(
  v.literal("appetizer"),
  v.literal("main"),
  v.literal("dessert")
);

export type Course = Infer<typeof courseValidator>;
// Inferred as 'appetizer' | 'main' | 'dessert'
```

### Document Types Without System Fields

```typescript
import { WithoutSystemFields } from "convex/server";
import { Doc } from "./_generated/dataModel";

export async function insertMessageHelper(
  ctx: MutationCtx,
  values: WithoutSystemFields<Doc<"messages">>
) {
  await ctx.db.insert("messages", values);
}
```

### Function Return Types (Client-side)

```typescript
import { FunctionReturnType } from "convex/server";
import { UsePaginatedQueryReturnType } from "convex/react";
import { api } from "../convex/_generated/api";

function MyHelper(props: {
  data: FunctionReturnType<typeof api.myFunctions.getSomething>;
}) { /* ... */ }

function MyPaginatedHelper(props: {
  paginatedData: UsePaginatedQueryReturnType<typeof api.myFunctions.getSomethingPaginated>;
}) { /* ... */ }
```

## Argument Validation for Type Inference

With argument validators, Convex infers argument types automatically:

```typescript
export default mutation({
  args: {
    body: v.string(),
    author: v.string(),
  },
  // Convex knows args type is { body: string, author: string }
  handler: async (ctx, args) => {
    const { body, author } = args;
    await ctx.db.insert("messages", { body, author });
  },
});
```

For internal functions with complex types, annotate manually:

```typescript
export default internalMutation({
  handler: async (ctx, args: { body: string; author: string }) => {
    const { body, author } = args;
    await ctx.db.insert("messages", { body, author });
  },
});
```
