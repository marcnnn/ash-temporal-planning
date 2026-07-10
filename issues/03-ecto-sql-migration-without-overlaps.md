# ecto_sql: `WITHOUT OVERLAPS` temporal keys in migrations

**Repo:** elixir-ecto/ecto_sql · **Helps:** 08 (raw DDL works meanwhile)
**File as:** proposal on the [elixir-ecto mailing list](https://groups.google.com/g/elixir-ecto) — GitHub issues there are bugs-only.
**Status (2026-07-10):** Still relevant. The ash_postgres `temporal` branch emits this DDL itself in its migration generator; ecto_sql support would simplify it.

## Problem

PG 18: `PRIMARY KEY (id, valid_at WITHOUT OVERLAPS)` / temporal `UNIQUE`.
Not expressible in the migration DSL. Note `constraint/3` already supports
the PG<18 fallback via `exclude:` (struct field `lib/ecto/migration.ex:549`;
gist `&&` example in docs at :1622) — this issue is the PG 18 form, which
additionally implies NOT NULL/no-empty-range and is referenceable by
`PERIOD` FKs.

## Minimal change — constraint-based only (smallest reviewable diff)

PG allows adding a temporal PK via `ALTER TABLE ... ADD PRIMARY KEY (...)`,
so extend `constraint/3` instead of touching the column DSL:

```elixir
create constraint(:products, :products_pkey,
  primary_key: [:id, without_overlaps: :valid_at])
create constraint(:products, :uniq, unique: [:id, without_overlaps: :valid_at])
```

- `Constraint` struct (defstruct :546-559): add `primary_key`/`unique`
  fields, default `nil`.
- Render in `new_constraint_expr/1` (postgres `connection.ex:1656/:1667` as
  templates); raise on other adapters.
- Docs: requires `btree_gist` (`execute "CREATE EXTENSION ..."`), range/
  multirange column, PG 18+.

Column-level sugar (`add :valid_at, :tstzrange, primary_key: :without_overlaps`)
is a possible follow-up PR, not part of this one.

## Non-regression

- New struct fields default `nil`; existing `check`/`exclude` rendering and
  plain `primary_key: true` output untouched. DDL tests pass unedited.

## Acceptance

- [ ] Temporal PK/UNIQUE create and roll back cleanly on PG 18.
- [ ] Docs position `exclude:` as the PG<18 fallback.
