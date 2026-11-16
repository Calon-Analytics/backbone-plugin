---
name: backbone-test
description: Run the already-generated dbt tests for a Backbone tenant via the backbone-test skill — executes existing tests only, never regenerates. Requires a prior /backbone-apply (tests emitted + synced into the runtime dbt repo).
argument-hint: <tenant_id>
---

# /backbone-test

Run the generated dbt tests for the supplied tenant. The tests are
emitted from the ledger/ontology and synced into the runtime dbt repo by
`/backbone-apply` (card #613, ADR-0012).
This command **only executes** them — it does **not** regenerate or
re-sync.

**Usage:** `/backbone-test <tenant_id>`

When invoked, run the `backbone-test` skill against tenant `$0`. The
skill confirms the tenant has an active runtime dbt repo with generated
tests present, executes `dbt test` via the existing quality REST endpoint
(`POST /api/vault/quality/{tid}/test`), and reports a pass/fail summary
that splits **structural-error** failures (block-worthy, exit non-zero)
from **warn-severity** coverage failures (advisory, apply proceeds), per
ADR-0012.

This command does **not** generate tests and does **not** build. If the
tenant has no generated tests in its runtime dbt repo, the skill
hard-errors with a redirect to `/backbone-apply {tid}` — it never
silently passes. Generating tests is part of apply/build; `/backbone-test`
re-runs what apply already produced.

On a **structural-green** result (no error-severity failures), the skill
then freezes the vault contract by POSTing to
`POST /api/config/{tid}/vault-manifest/freeze` — the certificate of a green
revolution (#689 S6, ADR-0022). A non-green build is never frozen (the
endpoint 409s and the last green manifest persists). See the skill's
step 5 for details.

If no tenant ID is provided, fall back to the `BACKBONE_DEFAULT_TENANT_ID`
environment variable. If neither is set, ask the user which tenant to
target.
