# ecto: `for_portion_of` query clause for update_all/delete_all

**Repo:** elixir-ecto/ecto · **Blocks:** 02, 09
**File as:** proposal on the [elixir-ecto mailing list](https://groups.google.com/g/elixir-ecto) — GitHub issues there are bugs-only.

## Problem

PG 19: `UPDATE products FOR PORTION OF valid_at FROM $1 TO $2 SET ...`.
The clause sits between the table reference and `SET`/`USING` — no fragment
position can express it. Raw SQL is the only workaround and loses query
composition.

## Minimal change (one small PR; SQL generation is issue 02)

```elixir
from(p in Product, where: p.id == ^id)
|> for_portion_of(:valid_at, ^from_ts, ^to_ts)   # nil bound = unbounded
|> Repo.update_all(set: [price: 1099])
```

1. Add `for_portion_of: nil` to the query struct — `lib/ecto/query.ex:398-417`,
   next to `lock:` (:415).
2. New `Ecto.Query.Builder.ForPortionOf`, mirroring `Builder.Lock`
   (`defmacro lock/3`, `query.ex:2536`).
3. Planner: include in normalization/cache key; reject on `:all` operations.
4. Docs: leftover rows are INSERTs (triggers fire, `RETURNING` excludes
   them); temporal keys can't be `on_conflict` targets.

## Non-regression

- `nil` default ⇒ identical code paths; cache keys unchanged for existing
  queries (test explicitly). No new adapter callbacks; adapters that don't
  render the field must raise when non-nil, never drop it. Suite passes
  unedited.

## Acceptance

- [ ] Macro round-trips through planner cache; `:all` rejects it.
- [ ] Non-supporting adapters raise clearly.
- [ ] Docs cover trigger/RETURNING semantics and the upsert limitation.

## Open questions

- Query clause (composable — preferred) vs `update_all` option?
- Multirange support in final PG 19 — confirm before proposing.
