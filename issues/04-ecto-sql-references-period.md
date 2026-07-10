# ecto_sql: `PERIOD` temporal foreign keys in `references/2`

**Repo:** elixir-ecto/ecto_sql · **Helps:** 08 (raw DDL works meanwhile)
**File as:** proposal on the [elixir-ecto mailing list](https://groups.google.com/g/elixir-ecto), can share a thread with 03.
**Status (2026-07-10):** Still relevant. The ash_postgres `temporal` branch validates PERIOD references itself (`lib/transformers/validate_temporal_references.ex`); ecto_sql support would simplify it.

## Problem

PG 18: `FOREIGN KEY (product_no, PERIOD valid_at) REFERENCES products
(product_no, PERIOD valid_at)` — referenced entity must exist for the whole
referencing period. Only `NO ACTION` allowed. `references/2` has composite
FKs (`with:`) but no PERIOD.

## Minimal change (one small PR)

```elixir
add :product_no, references(:products,
  column: :product_no, period: [valid_at: :valid_at])
```

- `Reference` struct (`lib/ecto/migration.ex` defstruct ~:484-509): add
  `period: nil`.
- `references/2` (:1559): raise if `period` is set with `on_delete`/
  `on_update` other than `:nothing` (PG limitation).
- Postgres `reference_expr/3` (`connection.ex:1865`, drop at :1886): emit
  `PERIOD` on both column lists. Other adapters: raise.

## Non-regression

- `nil` period ⇒ identical struct handling, validation, DDL, and constraint
  names (deployed apps' `foreign_key_constraint/3` matching intact).
  FK tests pass unedited.

## Acceptance

- [ ] `period:` renders correct FK DDL in create/alter/drop.
- [ ] Disallowed `on_delete` + `period:` raises at build time.
- [ ] Verify temporal FK violations still raise SQLSTATE 23503 with the
      constraint name (changeset matching unchanged).
