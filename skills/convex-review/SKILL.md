---
name: convex-review
description: Comprehensive Convex code review checklist for production readiness. Use when auditing a Convex codebase before deployment, reviewing pull requests, or checking for security and performance issues in Convex functions.
---

# Convex Code Review

Run through this checklist when reviewing Convex code for production readiness.

## Security Checklist

### 1. Argument Validators
- [ ] All public `query`, `mutation`, `action` have `args` with validators
- [ ] No use of TypeScript-only type annotations for public function arguments
- [ ] HTTP actions validate request body shape (e.g., with Zod)

**Search:** `query({`, `mutation({`, `action({` - check each has `args:`

### 2. Access Control
- [ ] All public functions check `ctx.auth.getUserIdentity()` where needed
- [ ] No use of client-provided email/username for authorization
- [ ] Granular functions preferred over generic update functions

**Search:** `ctx.auth.getUserIdentity` should appear in most public functions

### 3. Internal Functions
- [ ] `ctx.runQuery`, `ctx.runMutation`, `ctx.runAction` use `internal.*` not `api.*`
- [ ] `ctx.scheduler.runAfter` uses `internal.*` not `api.*`
- [ ] Crons in `crons.ts` use `internal.*` not `api.*`

**Search:** `api.` in convex directory - should not be used for scheduling/running

### 4. Table Names in DB Calls
- [ ] All `ctx.db.get`, `ctx.db.patch`, `ctx.db.replace`, `ctx.db.delete` include table name

**Search:** `db.get(`, `db.patch(`, `db.replace(`, `db.delete(` - first arg should be string

## Performance Checklist

### 5. Database Queries
- [ ] No `.filter()` on database queries (use `.withIndex()` or filter in code)
- [ ] `.collect()` only used with bounded result sets (<1000 docs)
- [ ] Pagination used for potentially large result sets

**Search:** `\.filter\(\(?q` for filter usage, `\.collect\(` for collect usage

### 6. Indexes
- [ ] No redundant indexes (e.g., `by_foo` and `by_foo_and_bar`)
- [ ] Queries that filter large tables use `.withIndex()`

**Review:** `schema.ts` index definitions

### 7. Date.now() in Queries
- [ ] No `Date.now()` usage in query functions
- [ ] Time-based filtering uses boolean fields or client-passed timestamps

**Search:** `Date.now()` in query functions

### 8. Promise Handling
- [ ] All promises are awaited (`ctx.scheduler`, `ctx.db.*`, etc.)

**ESLint:** Enable `no-floating-promises` rule

## Architecture Checklist

### 9. Action Usage
- [ ] `ctx.runAction` only used when switching between runtimes
- [ ] No unnecessary sequential `ctx.runMutation`/`ctx.runQuery` calls in actions
- [ ] Side effects (external API calls) happen between read and write

### 10. Code Organization
- [ ] Business logic in helper functions, not directly in handlers
- [ ] Shared code in `convex/model/` or similar directory
- [ ] Public API handlers are thin wrappers

### 11. Transaction Consistency
- [ ] Related reads happen in same query/mutation (not separate `ctx.run*` calls)
- [ ] Batch operations use single mutation, not loops

## Quick Regex Searches

| Issue | Regex | Action |
|-------|-------|--------|
| `.filter()` on queries | `\.filter\(\(?q` | Replace with `.withIndex()` |
| `.collect()` usage | `\.collect\(` | Verify bounded results |
| Missing table name | `db\.(get\|patch\|replace\|delete)\([^"']` | Add table name |
| `Date.now()` in query | `Date\.now\(\)` | Remove from queries |
| `api.*` in functions | `api\.[a-z]` | Change to `internal.*` |

## Production Readiness Summary

Before deploying to production:

1. **Security**: Validators + Auth checks + Internal functions
2. **Performance**: Indexes + Bounded queries + No Date.now()
3. **Architecture**: Helper functions + Proper action usage
4. **Code Quality**: Awaited promises + Table names in db calls
