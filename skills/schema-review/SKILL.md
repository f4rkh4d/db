---
name: schema-review
description: Use when reviewing a database schema for issues before shipping. Walks the catalog and reports actual problems — missing indexes on FKs, columns wider than they need to be, NULLABLE-but-shouldn't-be, orphaned constraints, conflicting unique sets. Outputs a prioritized list, not generic best-practice lectures.
---

# schema-review

Reviews a Postgres schema and reports concrete problems. Targets the issues that bite later: lock contention, query plans degrading, slow migrations. **No abstract "consider partitioning" advice.**

## When to use

- Before deploying a new schema
- When inheriting an unfamiliar codebase ("can you look over the DB?")
- When `pg_stat_statements` shows slow queries and you want to find why

## Process

1. **Connect read-only.** Either use the live DB (preferred) or `pg_dump --schema-only` text. Never modify anything.

2. **Walk the catalog.** Pull these views:

   ```sql
   -- Tables + sizes
   SELECT relname, n_live_tup, pg_total_relation_size(oid)
     FROM pg_class JOIN pg_namespace ON relnamespace = pg_namespace.oid
     WHERE relkind = 'r' AND nspname = 'public';

   -- Foreign keys without supporting index
   SELECT c.conname, c.conrelid::regclass, a.attname
     FROM pg_constraint c
     JOIN pg_attribute a ON a.attrelid = c.conrelid AND a.attnum = ANY(c.conkey)
     WHERE c.contype = 'f'
       AND NOT EXISTS (
         SELECT 1 FROM pg_index i
         WHERE i.indrelid = c.conrelid AND a.attnum = ANY(i.indkey)
       );

   -- Columns that could be NOT NULL (data is fully populated)
   -- Run per table, check pg_stats.null_frac.
   ```

3. **Report by category.** Format as a table for fast scanning:

   ```
   PRIORITY  KIND       LOCATION                  ISSUE                                    FIX
   ----------------------------------------------------------------------------------------------------
   high      missing-idx orders.user_id           FK with no index, table 4M rows           CREATE INDEX CONCURRENTLY orders_user_id_idx ON orders (user_id)
   high      nullable    users.email              0 NULL rows in 50k, but column nullable   ALTER COLUMN email SET NOT NULL (after backfill check)
   med       wide-col    products.description     text used as enum (3 distinct values)     ALTER to small enum or check constraint
   med       no-pk       audit_log_archive        no primary key, INSERT-only               ADD bigserial PK
   low       comment     subscriptions.provider   no comment on a non-obvious column        COMMENT ON COLUMN ...
   ```

4. **Check the indexes you DO have.** Common mistakes:

   - **Duplicate indexes** — `idx1 (a)` and `idx2 (a, b)` — drop the narrower one if it's never used alone (`pg_stat_user_indexes.idx_scan = 0`)
   - **Indexes on tiny tables** — overhead > benefit
   - **Bloated indexes** — `pg_relation_size(indexrelid) / pg_relation_size(indrelid)` > 0.5 → consider REINDEX CONCURRENTLY

5. **Constraints.** Verify:

   - Every table has a primary key (or explicit reason)
   - Foreign keys cascade or restrict — pick consciously, not by default ON DELETE NO ACTION
   - CHECK constraints that should exist (`amount_kzt >= 0`, `email LIKE '%@%'`)

6. **Booleans + status fields.** A `text` column with values like `'active' | 'frozen' | 'cancelled'` should be a Postgres ENUM or a CHECK constraint:

   ```sql
   ALTER TABLE subscriptions
     ADD CONSTRAINT subscriptions_status_check
     CHECK (status IN ('trial', 'active', 'past_due', 'cancelled'));
   ```

7. **Output:** the prioritized list + suggested commands. **Don't run any of them.** The user reviews and applies.

## What NOT to do

- **Don't suggest "consider partitioning" / "consider sharding"** unless the table is genuinely >100GB and queries are slow. Premature scale-out is its own problem.
- **Don't suggest dropping unused indexes from `pg_stat_user_indexes` blindly.** They might be used only by rare admin queries that pgss didn't capture in your sample window.
- **Don't recommend adding indexes for every column.** Each index slows writes and uses disk; only justify-by-query.
- **Don't lecture about 3NF, BCNF, schema purism.** Real schemas have denormalization for performance reasons. Flag obvious redundancy only when it's clearly accidental.

## Examples

### Real audit output

```
PRIORITY  KIND          LOCATION                  ISSUE                                              FIX
-------------------------------------------------------------------------------------------------------------
high      missing-idx   memberships.user_id       FK with no index, joins from users will sequential CREATE INDEX CONCURRENTLY memberships_user_id_idx ON memberships (user_id)
high      nullable      tenants.name              column NOT NULL implicit but TYPE allows NULL      ALTER COLUMN name SET NOT NULL
med       text-enum     subscriptions.status      4 distinct values, no CHECK constraint              ADD CONSTRAINT subscriptions_status_check CHECK (status IN (...))
med       wide-col      orders.notes              text(unbounded), 99% rows < 200 chars              ADD CHECK (length(notes) < 2000) to bound
low       no-comment    products.cubature         what unit? m³? cm³?                                COMMENT ON COLUMN products.cubature IS 'volume in cubic meters'
low       cascade       orders.user_id ON DELETE  RESTRICT — but app expects user delete to work     change to SET NULL or CASCADE based on policy
```

## Recovery

- **No live DB available, only schema dump:** Skip the data-driven checks (NULL fraction, table size). Flag them as "verify against live data".
- **Schema is huge (>100 tables):** Suggest narrowing scope to one feature area. Don't try to review everything in one pass.
- **The user wants ratings ("is my schema good?"):** Refuse to give a grade. Give the issue list. They decide what's worth fixing.
