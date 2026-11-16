---
name: backbone-run
description: Materialise a built Backbone vault into Snowflake via the backbone-run skill, behind a DML approval gate. DEV requires explicit confirmation; PROD (or any non-DEV target) is refused outright. Requires a prior /backbone-apply (the build must exist).
argument-hint: <tenant_id> [environment] [reload-seed <seed> ...]
---

# /backbone-run

Materialise the **already-built** vault for the supplied tenant into
Snowflake. The build is produced by `/backbone-apply` (it writes the
artefacts under `{estate root}/{tid}/build_control/` — the estate root is
the backend's own `tenant_root`, resolved at the skill's step 0, never the
literal `tenant_settings`). This command
ensures the tenant dbt repository exists (idempotent), then runs
`dbt seed` + `dbt run` **synchronously** and reports the pass/fail
outcome (card #581, epic #571 Phase 1b).

**Usage:** `/backbone-run <tenant_id> [environment] [reload-seed <seed> ...]`

`reload-seed <seed>` (repeatable, card #1167) drops and reloads that seed from
its CSV before the run, needed when the seed's columns changed. Only the named
seeds are reloaded; every other seed loads as usual. The confirmation lists them.

`replay <satellite> [<satellite> …]` (card #1208) replays the named satellites on DEV
after a normal run: each re-reads its latest staging row per key and adds a version
only where the hashdiff changed under a changed hard rule. History is kept, and a
second replay inserts nothing. It cannot be combined with `reload-seed`.

When invoked, run the `backbone-run` skill against tenant `$0`. This is a
**DML operation** — it writes rows into the tenant's Snowflake schemas —
so it sits behind a hard approval gate:

- **PROD (or any non-DEV target): refused outright.** No override, no
  confirmation path — mirroring the `main`/`prod` branch-protection rule.
- **DEV: runs only after an explicit one-line operator confirmation.**
  The skill presents the target schema + model/seed counts + environment
  and waits for you to confirm before running.

This command does **not** build, regenerate, or `dbt compile`. If the
tenant has no build / no active repo, the skill hard-errors with a
redirect to `/backbone-apply {tid}` — it never silently passes. Building
is part of apply; `/backbone-run` materialises what apply already built.

Once dbt has returned, on PASS or FAIL, the skill redraws the tenant's lineage page
(`/backbone-lineage <tid> from-run`) and adds a **Lineage** line to the report; that line
is advisory and never changes the PASS/FAIL headline.

If no tenant ID is provided, fall back to the `BACKBONE_DEFAULT_TENANT_ID`
environment variable. If neither is set, ask the user which tenant to
target.
