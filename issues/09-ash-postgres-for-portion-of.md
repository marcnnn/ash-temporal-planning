# ash_postgres: `FOR PORTION OF` bulk writes

**Repo:** ash-project/ash_postgres · **Depends on:** 01, 02, 06 · **PG 19+**
**File as:** GitHub issue (proposal), use-case-first, after the upstream ecto thread exists; AI-policy vetting applies.

## Minimal change (one small PR)

`lib/data_layer.ex`:

- `update_query/4` (:1754): when `options[:portion]` present, apply ecto's
  `for_portion_of` to the built query before `repo.update_all` (:1841).
- `destroy_query/4` (:2074): same before `repo.delete_all` (:2136; audit
  the other `delete_all` at :4131).
- `can?(_, {:portion_of, _})`: true iff server version ≥ 19, reusing the
  existing `min_pg_version` gating pattern.
- Raise clearly if an `upsert?` action targets a temporal key.

Document: portion writes + `return_records?` exclude leftover rows
(re-query for full fidelity); leftovers bypass Ash changes → AshPaperTrail
misses them (handled in issue 10). Test portion + atomic updates together.

## Non-regression

- Branch only on `options[:portion]`; nil path issues the identical query.
  New `can?` clause sits above the existing catch-all; all current answers
  unchanged. No extra DB roundtrips for non-users. Suite passes unedited on
  PG 14–18.

## Acceptance

- [ ] `Ash.bulk_update/bulk_destroy` with `portion:` emit correct SQL on
      PG 19; capability false + clear error on ≤ 18.
- [ ] End-to-end row-splitting verified from an Ash action.
