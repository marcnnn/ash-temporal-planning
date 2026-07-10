# ash: portion capability + `portion` option on bulk update/destroy

**Repo:** ash-project/ash · **Blocks:** 09
**File as:** GitHub issue (proposal). Per Ash CONTRIBUTING: lead with the **use case**, not the design; AI-policy vetting applies (see README filing guide).

## Minimal change (one small PR; no data-layer implementation here)

```elixir
Ash.bulk_update(Product, :reprice, %{price: 1099},
  portion: [attribute: :valid_at, from: ^f, to: ^t])
```

1. Add `{:portion_of, :update | :destroy}` to the `feature()` union
   (`lib/ash/data_layer/data_layer.ex:74-121`), negotiated via `can?/2`.
2. New optional `portion:` key on `Ash.bulk_update`/`bulk_destroy`, passed
   through the existing options argument of `update_query/4` (:255) /
   `destroy_query/4` (:266).
3. Validate: capability present, attribute is a range type matching the
   bounds; else `Ash.Error.Invalid`.
4. Docs: leftover rows bypass `return_records?`/notifications/hooks;
   `upsert?` can't target temporal keys.

Out of scope: range types in core (issue 07), resource DSL (issue 10).

## Non-regression

- New feature atom: existing data layers' `can?` catch-alls return false —
  compile and behave unchanged; error only if a user passes `portion:`.
- Absent `portion:` ⇒ identical option handling and code paths. No new or
  changed `Ash.DataLayer` callbacks. Suite passes unedited.

## Acceptance

- [ ] Capability documented; unsupporting layer + `portion:` errors clearly.
- [ ] Option reaches `update_query/4`/`destroy_query/4` untouched.
- [ ] Visibility semantics documented.
