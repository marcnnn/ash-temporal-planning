# postgrex: verification checklist (expected: no changes)

**Repo:** elixir-ecto/postgrex
**File as:** nothing upstream unless a defect is found — then a GitHub bug report via their bug template, with repro.
**Status (2026-07-10):** Still relevant, new data point: the ash_postgres `temporal` branch states Postgrex cannot decode a literal `'infinity'` range upper bound — add to the checklist as a possible upstream postgrex item.

Driver looks temporal-ready already: `Postgrex.Range`/`Postgrex.Multirange`
(`lib/postgrex/builtins.ex:51/:79`) with binary codecs for the built-in
range types; SQLSTATE `23P01 exclusion_violation` mapped
(`lib/postgrex/errcodes.txt:241`), so constraint-name extraction for Ecto
works.

## Checks (against PG 18 + PG 19 beta)

- [ ] `WITHOUT OVERLAPS` PK violation: `23P01` or `23505`? Answer decides
      `exclusion_constraint/3` vs `unique_constraint/3` in issues 03/08/09.
- [ ] Temporal FK violation raises `23503` with constraint name.
- [ ] Empty/unbounded range round-trips; clean error when a temporal key
      rejects an empty range.
- [ ] `FOR PORTION OF` with `$1`/`$2` scalar bound params executes through
      the extended query protocol.

## Outcome

All green ⇒ close as no-op, record findings here, link from 03/08/09. Any
codec gap ⇒ file upstream with repro; fix must keep existing wire encodings
and struct shapes (test suite passes unedited).
