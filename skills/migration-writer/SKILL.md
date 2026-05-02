---
name: migration-writer
description: Use when adding a database migration. Writes the up/down pair as plain SQL, considers backward-compat (deploy order, locking, default values for new NOT NULL columns), and emits a manual rollback test plan. Refuses to ship destructive changes without an explicit confirmation step.
---

# migration-writer

Writes Postgres migrations that survive zero-downtime deploys. Treats every change as: (1) what's the schema delta, (2) what's the data backfill, (3) what could go wrong if reverted halfway, (4) what locks does this take.

## When to use

- The user wants to add/change/drop a column, table, index, or constraint
- They say: "migration for X", "schema change", "alter table"
- Before tagging any release that includes schema work

## Process

1. **Detect the framework.** Look at: `migrations/*.sql`, `prisma/migrations/`, `db/migrate/`, `alembic/versions/`, `Cargo.toml` for sqlx, `flyway.conf`. Use the project's existing tool — don't drag in a new one.

2. **Pick the next migration number.** Count existing files; increment. Common patterns:
   - 4-digit prefix: `0007_add_subscription.sql`
   - timestamp: `20260502_add_subscription.sql`
   - Alembic style: `<hash>_<slug>.py`

3. **Classify the change.** This drives every decision below:

   | category | risk | needs |
   |---|---|---|
   | `additive_safe` (new nullable column, new table, new index CONCURRENTLY) | low | up only |
   | `additive_strict` (new NOT NULL column, new unique constraint) | medium | default value or backfill phase, then NOT NULL |
   | `mutating` (rename, type change) | high | dual-write phase, deploy gate |
   | `destructive` (drop column/table) | high | hard halt, ask user to confirm |

4. **Write up.sql.** Plain SQL when possible — avoids ORM lock-in.

   Patterns:

   - **New table:** straight `CREATE TABLE`, primary key, foreign keys, indexes.
   - **New nullable column:** `ALTER TABLE x ADD COLUMN y TYPE NULL;` — instant in Postgres ≥11.
   - **New NOT NULL with default:** `ALTER TABLE x ADD COLUMN y TYPE NOT NULL DEFAULT 'val';` — also instant in Postgres ≥11. NEVER add NOT NULL without default in one shot on a populated table.
   - **New index:** `CREATE INDEX CONCURRENTLY` (cannot run inside transaction). Drop the BEGIN/COMMIT wrapper if your runner adds one.
   - **Rename column:** ship in 3 migrations: (1) add new col + dual-write trigger, (2) backfill, (3) deploy reads of new col + drop old col.
   - **Drop column:** `ALTER TABLE x DROP COLUMN y;` — but only after a deploy where no code reads `y`. Block in CI if grep finds references.

5. **Write down.sql** when the change is reversible. For destructive migrations, write down with a `-- WARNING: data loss` header and explicit re-creation SQL. Some changes can't be reversed (lossy type narrowing) — say so.

6. **Locking notes.** For each statement that takes locks heavier than `ACCESS EXCLUSIVE` for non-trivial duration, add a comment:

   ```sql
   -- ACCESS EXCLUSIVE on subscriptions: blocks all reads/writes during ALTER.
   -- Verified that table is < 100k rows, so duration is sub-second.
   ALTER TABLE subscriptions ALTER COLUMN status TYPE text;
   ```

7. **Rollback test plan.** Output as comment block at top of the migration:

   ```sql
   -- migrate up:
   --   psql -f migrations/0007_add_subscription_provider.sql
   --   verify: SELECT count(*) FROM subscriptions WHERE provider IS NULL;  -- should be 0
   --
   -- migrate down:
   --   psql -f migrations/0007_add_subscription_provider.down.sql
   --   verify: \d subscriptions  -- 'provider' column gone
   ```

8. **Show the diff, don't run.** Output the migration file path and contents. Suggest the apply command per detected tool. Never auto-apply.

## What NOT to do

- **Don't add `NOT NULL` to a populated column without a default or backfill.** It locks the table and fails when existing rows are NULL.
- **Don't `DROP COLUMN`** in the same migration as code that stops reading it. Ship in two deploys: first remove reads, then in the next migration drop.
- **Don't use the ORM's "auto-migration" features for prod.** Generate SQL, review SQL, ship SQL.
- **Don't run `CREATE INDEX` (without CONCURRENTLY) on a hot table.** It takes ACCESS EXCLUSIVE.
- **Don't truncate or delete data inside a migration** without an explicit `-- I REALLY MEAN IT` confirmation from the user.
- **Don't include data fixes** in schema migrations. Use a separate one-off script.

## Examples

### Simple additive

```sql
-- 0008_add_subscription_provider.sql
-- ACCESS EXCLUSIVE briefly during column add (instant on Postgres ≥11 with default).

ALTER TABLE subscriptions
  ADD COLUMN provider text NOT NULL DEFAULT 'manual';

-- down: 0008_add_subscription_provider.down.sql
ALTER TABLE subscriptions DROP COLUMN provider;
```

### Concurrent index (no transaction)

```sql
-- 0009_idx_orders_user.sql
-- NOTE: must run outside a transaction. If your runner wraps every file in BEGIN/COMMIT, run this manually.

CREATE INDEX CONCURRENTLY orders_user_id_idx ON orders (user_id);
```

### Rename in 3 phases

```sql
-- Phase 1: 0010_user_email_to_contact_email_phase1.sql
ALTER TABLE users ADD COLUMN contact_email text;
UPDATE users SET contact_email = email WHERE contact_email IS NULL;
-- App code now writes to BOTH email and contact_email. Deploy this first.

-- Phase 2: 0011_user_email_to_contact_email_phase2.sql (after deploy 1)
-- App code now reads from contact_email. Stop writing to old email column.

-- Phase 3: 0012_user_email_to_contact_email_phase3.sql (after deploy 2)
ALTER TABLE users DROP COLUMN email;
```

## Recovery

- **No migration tool detected:** Ask the user what they use. Don't pick one for them.
- **Migration would lock for >5s on prod-sized table:** Halt, suggest CONCURRENTLY / phased version.
- **User asks to drop a column with FK references:** List dependent FKs first, ask "drop these constraints first?".
