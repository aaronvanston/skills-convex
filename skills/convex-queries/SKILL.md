---
name: convex-queries
description: Best practices for Convex database queries, indexes, and filtering. Use when writing or reviewing database queries in Convex, working with `.filter()`, `.collect()`, `.withIndex()`, indexes in schema.ts, or optimizing query performance.
---

# Convex Queries

## Avoid `.filter()` on Database Queries

The `.filter()` method on database queries has the same performance as filtering in code. Use `.withIndex()` or `.withSearchIndex()` instead for efficient filtering, or filter in plain TypeScript for better readability.

```typescript
// Bad - using .filter()
const tomsMessages = await ctx.db
  .query("messages")
  .filter((q) => q.eq(q.field("author"), "Tom"))
  .collect();

// Good - use an index
const tomsMessages = await ctx.db
  .query("messages")
  .withIndex("by_author", (q) => q.eq("author", "Tom"))
  .collect();

// Good - filter in code (if index not needed)
const allMessages = await ctx.db.query("messages").collect();
const tomsMessages = allMessages.filter((m) => m.author === "Tom");
```

**Finding `.filter()` usage:** Search with regex `\.filter\(\(?q`

**Exception:** Paginated queries benefit from `.filter()` since it returns the requested document count including filtered results.

## Only Use `.collect()` with Small Result Sets

All results from `.collect()` count towards database bandwidth. If any document changes, the query re-runs or mutation hits a conflict.

For potentially large result sets (1000+ documents), use:

**1. Index filtering:**
```typescript
// Bad - potentially unbounded
const allMovies = await ctx.db.query("movies").collect();
const moviesByDirector = allMovies.filter(m => m.director === "Steven Spielberg");

// Good - bounded by index
const moviesByDirector = await ctx.db
  .query("movies")
  .withIndex("by_director", (q) => q.eq("director", "Steven Spielberg"))
  .collect();
```

**2. Pagination:**
```typescript
// Bad - potentially unbounded
const watchedMovies = await ctx.db
  .query("watchedMovies")
  .withIndex("by_user", (q) => q.eq("user", "Tom"))
  .collect();

// Good - paginated
const watchedMovies = await ctx.db
  .query("watchedMovies")
  .withIndex("by_user", (q) => q.eq("user", "Tom"))
  .order("desc")
  .paginate(paginationOptions);
```

**3. Limit or denormalize:**
```typescript
// Good - use .take() with "99+" display
const watchedMovies = await ctx.db
  .query("watchedMovies")
  .withIndex("by_user", (q) => q.eq("user", "Tom"))
  .take(100);
const count = watchedMovies.length === 100 ? "99+" : watchedMovies.length.toString();

// Good - denormalize count in separate table
const watchedMoviesCount = await ctx.db
  .query("watchedMoviesCount")
  .withIndex("by_user", (q) => q.eq("user", "Tom"))
  .unique();
```

**Finding `.collect()` usage:** Search with regex `\.collect\(`

## Check for Redundant Indexes

Indexes like `by_foo` and `by_foo_and_bar` are usually redundant - keep only `by_foo_and_bar`.

```typescript
// Bad - redundant indexes
.index("by_team", ["team"])
.index("by_team_and_user", ["team", "user"])

// Good - single combined index
// Query for all team members: omit the user condition
const allTeamMembers = await ctx.db
  .query("teamMembers")
  .withIndex("by_team_and_user", (q) => q.eq("team", teamId))
  .collect();

// Query for specific user: include both conditions
const currentTeamMember = await ctx.db
  .query("teamMembers")
  .withIndex("by_team_and_user", (q) =>
    q.eq("team", teamId).eq("user", currentUserId))
  .unique();
```

**Exception:** `.index("by_foo", ["foo"])` is really an index on `foo` + `_creationTime`. If you need sorting by `foo` then `_creationTime`, you need the separate index.

## Don't Use `Date.now()` in Queries

Queries don't re-run when `Date.now()` changes, leading to stale results. It also invalidates query cache frequently.

```typescript
// Bad - stale results, cache thrashing
const releasedPosts = await ctx.db
  .query("posts")
  .withIndex("by_released_at", (q) => q.lte("releasedAt", Date.now()))
  .take(100);

// Good - use a boolean field updated by scheduled function
const releasedPosts = await ctx.db
  .query("posts")
  .withIndex("by_is_released", (q) => q.eq("isReleased", true))
  .take(100);
```

**Alternative:** Pass target time as argument from client, rounding to minimize cache invalidation.
