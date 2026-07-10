# ash_temporal: resource extension (design doc)

**Repo:** new package (pattern: AshArchival/AshPaperTrail) · **Depends on:** 07, 08; later 06/09

## Vision

```elixir
use Ash.Resource, data_layer: AshPostgres.DataLayer, extensions: [AshTemporal]

temporal do
  valid_time :valid_at, subtype: :utc_datetime_usec
  mode :domain            # or :history — see below
end

Ash.read!(Product)                                   # current rows only
Ash.read!(Product, as_of: ~U[2025-06-01 00:00:00Z])  # time travel
```

Bitemporal = this (valid time, DB-enforced) + AshPaperTrail (transaction
time) — declarative constraints instead of trigger emulation.

## Two modes

- **`:domain`** — time is business data (prices, bookings). The table
  becomes temporal: composite `(id, valid_at)` PK, PERIOD FKs, portion
  writes. API-invisible reads/updates via a `valid_at @> now()` base
  filter, but *not* schema-invisible (plain FKs, upserts, one-row-per-id
  assumptions affected).
- **`:history`** — invisible audit trail. Main table untouched; triggers
  maintain a `<table>_history` shadow table with `tstzrange` +
  `WITHOUT OVERLAPS`; a read-only `*.History` resource provides `as_of`
  queries. Zero impact until queried; works on PG 18. Compare with
  AshPaperTrail before choosing (paper trail: no DDL, actor metadata;
  history mode: SQL-queryable ranges, overlap integrity, catches writes
  that bypass Ash).

## Mechanics (`:domain`)

Transformers inject the range attribute (07), mark it `primary_key?`,
configure `temporal_key` (08), and add the current-rows base preparation.
Writes: close-and-insert change on PG 18; native portion writes (09) when
the capability is present. Verifiers reject duplicate identities and
temporal-key upserts. `as_of` read option rewrites the filter.

## Non-regression / safety

- Extension attaches only to resources that opt in — other apps/resources
  untouched by construction.
- `:history` mode guarantees zero changes to the host table; generated
  trigger DDL rolls back cleanly.
- Enabling `:domain` on an existing resource is a breaking migration *for
  that app* (PK rewrite): codegen must detect plain FKs pointing at the
  table and fail with guidance instead of generating a breaking migration.
- Public ash/ash_postgres extension APIs only.

## Open questions

- Package home: `ash-project/ash_temporal` vs inside ash_postgres?
- Leftover rows bypass Ash changes → paper-trail gap: document + re-query,
  DB trigger, or prefer close-and-insert when paper trail is present?
- `now()` base filter vs query caching (stable per transaction — verify).

## MVP acceptance

- [ ] `temporal do ... end` yields attr, temporal PK, migrations,
      current-by-default + `as_of` reads.
- [ ] Close-and-insert works on PG 18 (no PG 19 needed).
- [ ] Docs: bitemporal recipe with AshPaperTrail, PG version matrix.
