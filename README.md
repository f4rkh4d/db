# db

4 skills for Postgres work. Installable via [skl](https://github.com/f4rkh4d/skl) or by hand.

| skill | what it does |
|---|---|
| `/migration-writer` | up/down migrations with zero-downtime patterns, refuses destructive without confirm |
| `/rls-policy` | Postgres RLS — fail-closed by default, role-separated, with isolation test |
| `/schema-review` | walks pg_catalog, reports prioritized real issues (no "consider partitioning" filler) |
| `/n-plus-one` | find ORM N+1 patterns, fix with eager-load / batch / DataLoader / json_agg |

## install

```sh
skl install f4rkh4d/db
```

## why these four

these are the database skills i wish existed when i was reviewing other people's code or shipping new tables. each one knows the actual Postgres-specific rules — `CREATE INDEX CONCURRENTLY` outside a transaction, `set_config` instead of `SET LOCAL` for parameter binding, `BYPASSRLS` semantics — so the skill doesn't generate plausible-but-wrong SQL.

## license

MIT
