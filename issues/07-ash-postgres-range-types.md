# ash_postgres: range/multirange Ash types + overlap expressions

**Repo:** ash-project/ash_postgres · **Blocks:** 08, 10 · **Start here** (PG 14+, useful standalone)
**File as:** GitHub issue (proposal), use-case-first; AI-policy vetting applies (see README filing guide).

## Minimal change — two stacked PRs, new files only

**PR 1 — types** in `lib/types/` (templates: `ltree.ex`, `timestamptz.ex`):

- `AshPostgres.Range`, parameterized by subtype
  (`subtype: :utc_datetime_usec` → `tstzrange`; `:date` → `daterange`),
  wrapping `Postgrex.Range`; handles unbounded sides, empty ranges,
  inclusivity. Full `Ash.Type` behaviour (`storage_type/1`, `cast_input/2`,
  `cast_stored/2`, `dump_to_native/2`, `apply_constraints/2`).
- `AshPostgres.Multirange` — same over `Postgrex.Multirange`.

**PR 2 — expressions** via the existing expression machinery
(`Ash.DataLayer.functions/1`; `AshPostgres.SqlImplementation`):
`range_overlaps?/2` (`&&`), `range_contains?/2` (`@>`), `range/2..3`
constructor. Names must not collide with existing functions (`contains/2`
exists for strings).

## Non-regression

- New modules only; no existing file semantics change. Type-mapping
  additions must leave existing attribute→column mappings untouched
  (codegen on an existing app ⇒ empty diff). Suite passes unedited.

## Acceptance

- [ ] Cast/dump/load round-trips incl. edge cases; good changeset errors.
- [ ] Migration generator emits correct column types.
- [ ] `expr(range_overlaps?(p.period, range(^f, ^t)))` compiles to `&&`.

## Open questions

- Parameterized type vs concrete aliases (`TstzRange`, `DateRange`)?
- Elixir-side struct: `Postgrex.Range` directly vs normalized struct?
