# PostgreSQL 18/19 Temporal Features → Elixir/Ash Ecosystem

Issue plans for bringing PostgreSQL's SQL:2011 application-time features
(temporal keys, temporal foreign keys, `FOR PORTION OF`) to ecto, ecto_sql,
ash, and ash_postgres.

## Background

- **PG 18 (released):** `PRIMARY KEY (id, valid_at WITHOUT OVERLAPS)` and
  `FOREIGN KEY (x, PERIOD valid_at) REFERENCES ...`. Range/multirange
  columns, GiST-backed (needs `btree_gist`). Temporal FKs: `NO ACTION` only.
- **PG 19 (beta, GA expected fall 2026):** `UPDATE/DELETE ... FOR PORTION OF
  valid_at FROM x TO y` with automatic row splitting. Leftovers are real
  INSERTs (INSERT triggers fire); `RETURNING` excludes them.
- **Not bitemporal:** no native system time. Full bitemporality = these
  features (valid time) + AshPaperTrail or triggers (transaction time).

## Principles

1. **Minimal diffs.** Each issue is scoped to the smallest change a human
   can review in one sitting; anything bigger is split into stacked PRs.
2. **Zero regressions.** Everything is additive and opt-in: `nil` struct
   defaults, absent-by-default DSL options, capabilities defaulting to
   false. Unused feature ⇒ byte-identical SQL/migrations/snapshots, no new
   required callbacks, existing test suites pass **unedited**. Unsupported
   adapter/PG version ⇒ descriptive error, never wrong SQL.
3. **Errors over magic.** Semver-minor releases only.

## Issues

| # | File | Repo | Notes |
|---|------|------|-------|
| 01 | [ecto: `for_portion_of` query clause](issues/01-ecto-for-portion-of-query.md) | ecto | The one hard upstream blocker |
| 02 | [ecto_sql: FOR PORTION OF SQL](issues/02-ecto-sql-for-portion-of-sql-generation.md) | ecto_sql | Pairs with 01 |
| 03 | [ecto_sql: WITHOUT OVERLAPS keys](issues/03-ecto-sql-migration-without-overlaps.md) | ecto_sql | Sugar; raw DDL works today |
| 04 | [ecto_sql: `PERIOD` foreign keys](issues/04-ecto-sql-references-period.md) | ecto_sql | Sugar; raw DDL works today |
| 05 | [postgrex: verification](issues/05-postgrex-verification.md) | postgrex | Likely no-op |
| 06 | [ash: portion capability + bulk option](issues/06-ash-data-layer-portion-capability.md) | ash | Needed for 09 |
| 07 | [ash_postgres: range types](issues/07-ash-postgres-range-types.md) | ash_postgres | No deps — start here |
| 08 | [ash_postgres: temporal DDL codegen](issues/08-ash-postgres-migration-generator-temporal.md) | ash_postgres | Depends on 07 |
| 09 | [ash_postgres: portion bulk writes](issues/09-ash-postgres-for-portion-of.md) | ash_postgres | Depends on 01, 02, 06 |
| 10 | [ash_temporal: extension design](issues/10-ash-temporal-extension.md) | new package | Depends on 07, 08 |

## Filing guide (checked against each project's contribution policy)

**ecto / ecto_sql / postgrex (issues 01–05):** these repos disable blank
GitHub issues and accept only bug reports there. Feature proposals go to
the [elixir-ecto mailing list](https://groups.google.com/g/elixir-ecto) —
so 01–04 must be posted there as proposals (02–04 can reference the 01
thread). 05 is a verification task; open a GitHub bug (using their bug
template, with repro) only if a defect is found.

**ash / ash_postgres (issues 06–09):** GitHub issues are the right venue
for proposals, but per their CONTRIBUTING.md the discussion must center on
the **use case, not the proposed feature** — when filing, lead with the
scenario (temporal pricing/booking/history requirements) and attach the
design as supporting material. Talk to the core team (issue + Discord/
Forum) before starting implementation. PRs require `mix check` green and
automated tests.

**Ash AI policy (org-wide `AI_POLICY.md`, acceptance is part of their
issue/PR templates):** AI-assisted contributions are welcome, but all
content must be fully vetted and vouched for by the human submitter; no AI
co-author trailers; AI text must never be pasted as your own words unless
explicitly labeled AI-generated; keep issues/PRs the smallest reasonable
scope; zero tolerance, up to bans. The ecto projects publish no formal AI
policy — the same discipline applies as courtesy.

**Consequently, for this repo:** these documents were AI-assisted research
drafts. Before filing anything upstream: (1) personally verify the claims
and file:line anchors, (2) rewrite the submission in your own words or
explicitly label AI-generated parts, (3) trim to the minimal text the
venue expects.

## Sequencing

```
07 range types ──► 08 migration DDL ──► 10 ash_temporal extension
                                          ▲
01 ecto query ──► 02 ecto_sql SQL ──► 09 ash_postgres portion writes
                                          ▲
                    06 ash capability ────┘
03 / 04 migration sugar (independent) · 05 verification (anytime)
```

Phase 1 (PG 18, now): 07 → 08, plus 03/04. Phase 2 (PG 19): 01 → 02, then
06 → 09. Phase 3: 10.

## Cross-cutting gotchas

1. `ON CONFLICT DO UPDATE` can't target GiST/exclusion constraints → no
   upserts on temporal keys; temporal upsert = portion-update + insert.
2. `FOR PORTION OF` leftovers fire INSERT triggers and are invisible to
   `RETURNING` → partial visibility for returned records/notifications.
3. Verify which SQLSTATE a `WITHOUT OVERLAPS` PK violation raises
   (`23P01` vs `23505`) before wiring error mapping (issue 05).

## Reproducing the research

File:line anchors were verified against upstream `master` on 2026-07-09:

```sh
git clone --depth 1 https://github.com/elixir-ecto/ecto
git clone --depth 1 https://github.com/elixir-ecto/ecto_sql
git clone --depth 1 https://github.com/elixir-ecto/postgrex
git clone --depth 1 https://github.com/ash-project/ash
git clone --depth 1 https://github.com/ash-project/ash_postgres
```

PG 19 details are beta-cycle information — re-verify against final release
notes before opening issues 01/02 upstream.
