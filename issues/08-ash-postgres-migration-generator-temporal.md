# ash_postgres: temporal DDL in the migration generator

**Repo:** ash-project/ash_postgres · **Depends on:** 07 · **Blocks:** 10

Highest regression risk of the set (codegen + snapshots serve every
deployed app) ⇒ split into **three stacked PRs**, each independently
reviewable:

## PR A — exclusion constraints (PG 14+, smallest)

DSL entity `exclusion_constraints` (`exclude :no_overlap, using: :gist,
elements: [id: :=, valid_at: :&&]`), emitted via ecto_sql's existing
`constraint(..., exclude:)`; wire generated names into
`exclusion_constraint_names` (`lib/data_layer.ex:351`; error branch exists
at `lib/repo.ex:105`). Auto-add `btree_gist` to installed extensions when
used.

## PR B — `temporal_key` (PG 18)

`temporal_key :valid_at` in the `postgres` section: PK becomes
`(pk_attrs..., valid_at WITHOUT OVERLAPS)`. Emission point:
`lib/migration_generator/operation.ex:21-22` (`maybe_add_primary_key/1`,
used at :302/:345/:369) — via ecto_sql issue 03 when available, else raw
`execute` in the generated migration. Verifiers: reject identities
duplicating the temporal key and `upsert?` actions targeting it
(ON CONFLICT limitation).

## PR C — `period:` on references (PG 18)

`reference :product, period: {:valid_at, :valid_at}` mapped where
`AshPostgres.Reference` options become `references/2` args in
`lib/migration_generator/migration_generator.ex`; require
`on_delete/on_update :nothing`.

## Non-regression (hard requirements, all PRs)

- Snapshot keys written **only when used**; old snapshots load silently.
  CI test: upgrade + `mix ash.codegen` on an untouched app ⇒ **zero diff**.
- `maybe_add_primary_key/1` keeps its boolean contract.
- Generated names for existing constraints/indexes/references unchanged
  (renames would break deployed apps' error mapping).
- New verifiers only fire on resources declaring temporal options.
- Suite passes unedited.

## Acceptance

- [ ] codegen → migrate → rollback works on PG 18 (temporal PK, PERIOD FK,
      btree_gist), including tenant/`global?` splits.
- [ ] Constraint violations map to actionable Ash errors (SQLSTATE answer
      from issue 05).
