---
name: rls-policy
description: Use when adding Row-Level Security policies in Postgres for multi-tenant or per-user isolation. Writes ENABLE ROW LEVEL SECURITY + CREATE POLICY pairs with proper USING/WITH CHECK clauses, role grants, and a fail-closed test. Always uses set_config()/current_setting() over SET LOCAL when binding params.
---

# rls-policy

Postgres RLS done right. Default-deny, app-role separated from admin-role, parameterized via `current_setting('app.X', missing_ok=true)` so missing context fails closed.

## When to use

- Multi-tenant table that must never leak across tenants
- Per-user data (each user only sees their own rows)
- Audit-log style tables where readers should be scoped

Skip when: read-only public reference tables (categories, plans), platform-level tables only the admin role touches.

## Process

1. **Identify the role split.** Two roles minimum:
   - `<app>_app` — what the application uses, RLS applies, no superuser
   - `<app>_admin` — what super-admin UI uses, `BYPASSRLS`, sees everything

   If they don't exist, write a `0000_roles.sql`:

   ```sql
   DO $$ BEGIN
     IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'app_role') THEN
       CREATE ROLE app_role NOLOGIN;
     END IF;
     IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'admin_role') THEN
       CREATE ROLE admin_role NOLOGIN BYPASSRLS;
     END IF;
   END $$;
   GRANT USAGE ON SCHEMA public TO app_role, admin_role;
   ```

2. **Identify the scope key.** Almost always `tenant_id uuid` or `user_id uuid`. The column must be NOT NULL on every protected table — RLS can't reason about NULL safely.

3. **Write a helper function** that reads the current scope:

   ```sql
   CREATE OR REPLACE FUNCTION current_tenant_id() RETURNS uuid
   LANGUAGE sql STABLE AS $$
     SELECT NULLIF(current_setting('app.tenant_id', true), '')::uuid;
   $$;
   ```

   `, true` makes `current_setting` return NULL on missing var instead of erroring. `NULLIF(..., '')` handles empty-string set. Return NULL → policy fails → 0 rows. **Fail-closed by default.**

4. **For each table to protect:**

   ```sql
   ALTER TABLE memberships ENABLE ROW LEVEL SECURITY;

   GRANT SELECT, INSERT, UPDATE, DELETE ON memberships TO app_role;
   GRANT ALL ON memberships TO admin_role;

   CREATE POLICY tenant_isolation ON memberships
     USING (tenant_id = current_tenant_id())
     WITH CHECK (tenant_id = current_tenant_id());
   ```

   - `USING` — visible rows for SELECT/UPDATE/DELETE
   - `WITH CHECK` — required match on INSERT/UPDATE
   - Both clauses → can neither read nor write across tenants

5. **App-side context binding.** Document that the application sets context per request:

   ```ts
   await tx.execute(sql`SELECT set_config('app.tenant_id', ${tenantId}, true)`);
   ```

   `set_config(name, value, is_local=true)` — local to the current transaction. Only safe path. NEVER use `SET LOCAL app.tenant_id = ${id}` because Postgres `SET` doesn't accept bind parameters and the postgres-js client throws "syntax error at $1".

6. **Write the test.** A canonical RLS isolation test catches future regressions when someone adds a tenant table without a policy:

   ```ts
   test('tenant A cannot see tenant B rows', async () => {
     const a = await db.insert(tenants).values({ slug: 'a' }).returning();
     const b = await db.insert(tenants).values({ slug: 'b' }).returning();

     // Insert one row per tenant under privileged role
     await db.insert(memberships).values({ tenantId: a.id, userId: u.id, role: 'owner' });
     await db.insert(memberships).values({ tenantId: b.id, userId: u.id, role: 'owner' });

     // Read scoped to A under app_role
     const visible = await db.transaction(async (tx) => {
       await tx.execute(sql`SET LOCAL ROLE app_role`);
       await tx.execute(sql`SELECT set_config('app.tenant_id', ${a.id}, true)`);
       return tx.select().from(memberships);
     });
     expect(visible).toHaveLength(1);
     expect(visible[0].tenantId).toBe(a.id);
   });
   ```

   **Critical:** the `SET LOCAL ROLE` and the `set_config` and the SELECT must run in the **same transaction**. A nested `db.transaction()` opens a new pool connection and loses the outer setting.

7. **Add to existing policies index.** If the project keeps a list of RLS-protected tables in a comment block somewhere (`docs/rls.md` or similar), update it. Future-you needs the list.

## What NOT to do

- **Don't use `SET LOCAL app.X = $1`** — Postgres SET doesn't bind. Always `set_config(...)`.
- **Don't use a single role for app + admin.** If app_role has BYPASSRLS, RLS is theatre.
- **Don't grant ANY rights on protected tables to PUBLIC.** Always to a specific role.
- **Don't make `tenant_id` nullable.** RLS evaluates `tenant_id = current_tenant_id()` as UNKNOWN for NULL on either side, which gets filtered → confusion.
- **Don't `FORCE ROW LEVEL SECURITY` on dev.** It applies even to the table owner, which makes seeding harder. Use it in prod only if owner-bypass concerns you.

## Recovery

- **Existing table has rows with NULL tenant_id:** Halt. Backfill first via a one-off script, add NOT NULL, then policy.
- **Need cross-tenant admin reports:** Use `admin_role` (BYPASSRLS) and audit every query that role makes.
- **pgbouncer in statement-pooling mode:** RLS will be inconsistent because connections shuffle mid-transaction. Switch to transaction-pooling.
