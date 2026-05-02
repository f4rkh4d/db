---
name: pg-explain
description: Use when reading EXPLAIN / EXPLAIN ANALYZE output for a slow Postgres query. Walks the plan top-down, points at the actual cost driver (Seq Scan on a big table, Hash Join with a bad row estimate, Nested Loop blowing up), and proposes the smallest change that fixes it. Refuses to read a plan without ANALYZE — estimates without measurements lie.
---

# pg-explain

Reading a Postgres query plan well is a learnable skill. Reading it badly is the most common reason "I added an index and it didn't help."

## When to use

- A query is slow and you want to know why
- pg_stat_statements pointed at a query but the WHY is unclear
- You added an index and it isn't being used

## Process

1. **Always EXPLAIN ANALYZE, never just EXPLAIN.** EXPLAIN gives planner estimates; ANALYZE adds real measurements (rows actually returned, time spent). Estimates lie. Always:

   ```sql
   EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT) <query>;
   ```

   - **ANALYZE** runs the query (don't do this on UPDATE/DELETE in prod without a transaction wrapped in ROLLBACK)
   - **BUFFERS** shows shared/local hit/read counts — vital for "is this in cache or hitting disk"
   - **VERBOSE** shows the column list each node returns — sometimes you find a giant projection you didn't realise

2. **Read top-down, but execute is bottom-up.** The plan tree displays the top-level operation first; the leaf scans run first. Identify the leaves: those are your data access points. Anything above them is a transformation.

3. **Hunt the cost driver.** For each node check three numbers:

   - `actual time=X..Y rows=N loops=M` — actual cost
   - `cost=A..B rows=R width=W` — planner estimate
   - row count discrepancy `rows=N` (actual) vs `rows=R` (estimated)

   The bug is usually one of:
   - **Seq Scan on a large table where you expected Index Scan** → missing index, or planner thinks Seq Scan is cheaper because it'll match >5% of rows. Check `pg_stat_user_indexes.idx_scan` to see if your index is even being used.
   - **Hash Join with `Rows Removed by Filter:` huge** → join condition isn't selective enough, you're materialising too much. Sometimes fixed by composite index on the join keys + filter columns.
   - **Nested Loop with loops=N where N is in the thousands** → classic N+1 inside the database. Either the planner misestimated row count and chose a bad join, or you're using a function that's volatile and calling it per-row. Force the join order with CTE materialisation if the planner won't behave.
   - **Bitmap Heap Scan with `Rows Removed by Index Recheck:`** → lossy index match (e.g., a partial GIN index), need to either tighten the index condition or add a follow-up filter that the planner pushes down.
   - **Sort node spilling to disk** (`Sort Method: external merge Disk: NkB`) → either bump `work_mem` for that session, or reduce the sort input via a pre-filter.

4. **Row estimate vs reality is your most diagnostic signal.** A single node where the planner thinks 10 rows and reality is 100,000 cascades into bad join algorithm choices upstream. Fix the estimate first:

   - Run `ANALYZE <table>` if stats are stale
   - Increase `default_statistics_target` for the column (default 100, can go to 1000 or 10000 for high-cardinality columns)
   - For correlated columns, create extended statistics: `CREATE STATISTICS my_stats (dependencies, ndistinct) ON col1, col2 FROM tbl;`
   - For function results, the planner has no clue without help — wrap in a STABLE/IMMUTABLE function with `ROWS` declaration

5. **`Buffers: shared hit=A read=B`** — the cache story.
   - All hits, no reads → you're in shared_buffers, query is CPU-bound
   - Mostly reads → cold cache, repeat the query and see if it speeds up; if not, your working set exceeds shared_buffers
   - `dirtied` and `written` columns appear on writes — useful for spotting unintended page modifications

6. **Use `auto_explain` for production.** Configure it once, get plans for every slow query that hit prod, no replay needed:

   ```ini
   shared_preload_libraries = 'auto_explain'
   auto_explain.log_min_duration = '500ms'
   auto_explain.log_analyze = on
   auto_explain.log_buffers = on
   ```

   Plans land in postgres logs alongside the query text. Worth the slight overhead.

## What NOT to do

- **Don't add indexes from EXPLAIN guessing.** If the planner chose Seq Scan for a query that's matching 30% of rows, an index on that column won't help — index access is more expensive than seq scan past ~5% selectivity. Check the actual selectivity first.
- **Don't trust cost numbers between queries.** Cost is unitless and only meaningful for comparing alternative plans of the same query. "Cost 100,000" doesn't mean "slow" by itself.
- **Don't re-run EXPLAIN ANALYZE on cold cache and conclude the query is slow.** First run pays for cold reads; subsequent runs are warm. Run twice, measure the second.
- **Don't ignore `Rows Removed by Filter` on intermediate nodes.** A filter discarding 99% of input post-join is a sign the join order is wrong — push the filter into the inner side.

## Examples

### Reading a real plan

```
Limit  (cost=12345.67..12345.68 rows=10 width=64) (actual time=2843.124..2843.131 rows=10 loops=1)
  ->  Sort  (cost=12345.67..13245.67 rows=360000 width=64) (actual time=2843.121..2843.123 rows=10 loops=1)
        Sort Key: orders.created_at DESC
        Sort Method: top-N heapsort  Memory: 27kB
        Buffers: shared hit=4521 read=12389
        ->  Seq Scan on orders  (cost=0.00..8345.67 rows=360000 width=64) (actual time=0.024..2740.118 rows=362145 loops=1)
              Filter: (status = 'paid'::text)
              Rows Removed by Filter: 137855
              Buffers: shared hit=4521 read=12389
Planning Time: 0.123 ms
Execution Time: 2843.245 ms
```

Diagnosis: a Seq Scan over 500k rows to filter 360k of them, then a top-N heapsort to find 10. The cost driver is the Seq Scan (2740ms of the 2843ms total).

Fix candidates (in order of likely impact):
1. Index on `(status, created_at DESC)` — supports both the filter and the ORDER BY. Should turn the whole thing into an Index Only Scan with Limit.
2. If `status` has only 3-4 values, a partial index `(created_at DESC) WHERE status = 'paid'` is even tighter.
3. Don't over-index on `(created_at)` alone — planner might still pick Seq Scan because `status` filter selectivity is high enough.

After indexing, expected plan:

```
Limit  (cost=0.42..1.23 rows=10 width=64) (actual time=0.045..0.067 rows=10 loops=1)
  ->  Index Scan using orders_status_created_at_idx on orders  (cost=0.42..29130.18 rows=360000 width=64) (actual time=0.044..0.064 rows=10 loops=1)
        Index Cond: (status = 'paid'::text)
        Buffers: shared hit=12
```

3 orders of magnitude faster. The Index Scan stops after Limit because the index is already in the right order.

## Recovery

- **Plan is huge (deep tree, hundreds of nodes):** focus on nodes where `actual time` is a non-trivial fraction of `Execution Time`. The shape doesn't matter; the cost driver does. Filter the plan top-down to the slowest sub-tree, ignore the rest.
- **EXPLAIN ANALYZE itself is too slow to run:** the query is genuinely slow, that's data. Use auto_explain on prod with a short timeout and look at the plan from the original execution.
- **No write access to a prod replica:** EXPLAIN ANALYZE on the primary inside a transaction with ROLLBACK is safe for SELECT/UPDATE/DELETE. For DDL, never; use a copy of the schema on a dev machine.
