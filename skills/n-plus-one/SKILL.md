---
name: n-plus-one
description: Use when investigating a slow endpoint or report of "the page is laggy". Finds N+1 query patterns by reading the code (ORM/data-access layer) and the running query log. Outputs a fix per N+1 site — eager-load, batch loader, or query rewrite.
---

# n-plus-one

The classic database performance bug: one query for a list, then one query per item to load related data. Easy to miss in code review, painful in prod.

## When to use

- A list endpoint takes seconds instead of ms
- Logs show repeated near-identical queries within one request
- Database CPU is high but each individual query looks fast
- Server logs show many `SELECT * FROM x WHERE id = ?` patterns

## Process

1. **Get a sample request.** Either:
   - The user reproduces it once with a timer
   - You attach query logging: `log_statement = 'all'` in dev, or `pg_stat_statements` snapshot before/after
   - The endpoint's tracing data shows N similar spans

2. **Identify the trigger query and the loop.** The smell:

   ```
   SELECT * FROM orders WHERE user_id = $1;          -- 1 query
   SELECT * FROM order_items WHERE order_id = $1;    -- repeated N times
   SELECT * FROM order_items WHERE order_id = $2;
   SELECT * FROM order_items WHERE order_id = $3;
   ...
   ```

   In ORM speak:

   ```ts
   const orders = await db.orders.findMany({ where: { userId } });
   for (const o of orders) {
     o.items = await db.orderItems.findMany({ where: { orderId: o.id } }); // ← N+1
   }
   ```

3. **Pick the fix.** In order of preference:

   ### a) Eager load via the ORM

   Drizzle:
   ```ts
   const orders = await db.query.orders.findMany({
     where: eq(orders.userId, userId),
     with: { items: true },
   });
   ```

   Prisma:
   ```ts
   const orders = await prisma.order.findMany({
     where: { userId },
     include: { items: true },
   });
   ```

   ### b) Manual batch (when ORM eager-load doesn't fit)

   ```ts
   const orders = await db.orders.findMany({ where: { userId } });
   const items = await db.orderItems.findMany({
     where: inArray(orderItems.orderId, orders.map(o => o.id)),
   });
   const grouped = Map.groupBy(items, (i) => i.orderId);
   for (const o of orders) o.items = grouped.get(o.id) ?? [];
   ```

   Two queries instead of N+1, regardless of order count.

   ### c) DataLoader (for GraphQL or heavily-nested cases)

   ```ts
   const itemLoader = new DataLoader(async (orderIds) => {
     const items = await db.orderItems.findMany({ where: inArray(orderItems.orderId, orderIds) });
     return orderIds.map((id) => items.filter((i) => i.orderId === id));
   });
   ```

   Coalesces concurrent loads into one batch query per tick.

   ### d) Single denormalized query

   ```sql
   SELECT o.*,
          json_agg(i.*) FILTER (WHERE i.id IS NOT NULL) AS items
     FROM orders o
LEFT JOIN order_items i ON i.order_id = o.id
    WHERE o.user_id = $1
 GROUP BY o.id;
   ```

   One round trip, server-side aggregation. Good for SSR-rendered lists.

4. **Verify the fix.** Re-run the request, count queries. Should drop from N+1 to 2 (or 1).

   ```sql
   -- before
   SELECT count(*) FROM pg_stat_statements WHERE query LIKE 'SELECT * FROM order_items WHERE order_id = %';
   -- → 47

   -- after applying fix, re-run, then:
   -- → 1
   ```

5. **Add a regression guard** if the project has them. e.g. a test that wraps the endpoint and asserts query count ≤ k:

   ```ts
   const queryCount = await captureQueries(async () => {
     await fetch('/orders');
   });
   expect(queryCount).toBeLessThan(5);
   ```

## What NOT to do

- **Don't blindly add `include: { everything: true }`** — over-eager loading wastes bandwidth and hurts cache. Only the relations the response actually returns.
- **Don't replace one N+1 with another.** A `for` loop calling `await find()` inside a `findMany` `with:` block re-creates the bug.
- **Don't introduce a JOIN that returns a cartesian product** (1 order with 50 items → 50 rows of order data). Use `with:`/`include:` (which the ORM splits into 2 queries) or `json_agg` (server aggregation).
- **Don't call this "premature optimization".** N+1 isn't premature — it's a default behavior bug.
- **Don't fix one N+1 and ship without measuring.** Sometimes the "fix" creates a single 10-second query (cartesian explosion). Verify with EXPLAIN ANALYZE.

## Examples

### Drizzle, before / after

```ts
// before: 1 + N queries
const memberships = await db.select().from(memberships).where(eq(memberships.tenantId, t));
for (const m of memberships) {
  m.user = (await db.select().from(users).where(eq(users.id, m.userId)))[0];
}

// after: 1 query
const rows = await db
  .select({
    membership: memberships,
    user: users,
  })
  .from(memberships)
  .innerJoin(users, eq(users.id, memberships.userId))
  .where(eq(memberships.tenantId, t));
```

### Prisma, before / after

```ts
// before: 1 + N
const posts = await prisma.post.findMany({ take: 20 });
for (const p of posts) p.author = await prisma.user.findUnique({ where: { id: p.authorId } });

// after: 1
const posts = await prisma.post.findMany({ take: 20, include: { author: true } });
```

## Recovery

- **The "list" really is meant to load each item separately** (e.g. running per-item business logic that issues its own queries): consider parallelizing with `Promise.all` and accept the cost, or batch the work into a job.
- **You can't reproduce locally:** turn on `log_min_duration_statement = 100` in staging and capture the request. The pattern shows up immediately.
- **The N+1 is in a third-party library:** open an issue with a minimal repro. Fix it locally with a query interceptor / monkey-patch as a temporary measure.
