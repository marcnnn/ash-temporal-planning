# ecto_sql: render `FOR PORTION OF` in the Postgres adapter

**Repo:** elixir-ecto/ecto_sql · **Depends on:** 01 · **Blocks:** 09

## Minimal change (small diff in 3 files + tests)

`lib/ecto/adapters/postgres/connection.ex`:

- `update_all/2` (:212): insert the portion expression into the prefix
  built at :217 — `["UPDATE ", from, " AS ", name, portion, " SET "]`.
  **Verify grammar order** first: whether PG 19 places the clause before or
  after the alias.
- `delete_all/1` (:226): same, after `["DELETE FROM ", from, " AS ", name]`.
- myxql/tds connection modules: raise `Ecto.QueryError` when non-nil.

Checks: param ordering (bounds render before SET params); behavior with
`FROM`/`USING` joins (:219/:231) — raise if PG rejects the combination;
CTEs (:214) should compose — test it.

## Non-regression

- Render only when non-nil; nil path builds identical iodata. Existing
  exact-SQL tests pass unedited — that is the proof. CI matrix gains PG 19,
  loses nothing.

## Acceptance

- [ ] Correct SQL for bounded and NULL-bounded portions on PG 19.
- [ ] Clear errors: unsupported adapter; portion + joins if PG rejects it.
- [ ] Integration tests: leftover count, INSERT triggers, RETURNING scope.
