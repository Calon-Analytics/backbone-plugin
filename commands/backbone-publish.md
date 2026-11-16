---
name: backbone-publish
description: Publish/refresh a tenant's governed Snowflake semantic view (scoped dbt run --select tag:semantic_view) via the backbone-publish skill, behind a DEV-only DML approval gate, then run /backbone-mcp <tid> refresh inside the same confirmation — connector-role grants, the live grant assertion, the delegated MCP-server refresh and the connector-users report. DEV requires explicit confirmation; PROD (or any non-DEV target) is refused outright. Requires a prior /backbone-apply (the semantic model must exist).
argument-hint: <tenant_id> [environment]
---

# /backbone-publish

Publish (or refresh) the **already-built** governed Snowflake semantic view for the
supplied tenant, then keep its MCP grounding in sync (epic #649, Slice 0 Commit 4 /
card #649d). The build is produced by `/backbone-apply` (it emits the `sem_*.sql`
model under `{estate root}/{tid}/build_control/vault/dataproducts/generated/` —
the estate root is the backend's own `tenant_root`, resolved at the skill's
step 0, never the literal `tenant_settings`).

When invoked, run the `backbone-publish` skill against tenant `$0`. The skill:

1. Runs the offline **pre-publish gate** + build-readiness check.
2. Publishes via a **scoped** `dbt run --select tag:semantic_view` (publish-only — no
   `dbt seed`, no full suite) behind the DML gate.
3. Runs **`/backbone-mcp <tid> refresh`** (#1047 T2) inside the same confirmation — no
   second prompt. Refresh:
   - grants the read-only connector role (default `{TID}_MCP_RO`; per-tenant names in
     `settings.yaml` `mcp_connector`) `SELECT` on every view and base table when
     `SHOW ROLES LIKE` finds the role (#904). When the role is absent it writes the
     generated connector setup script (`mcp/generated/{tid}_mcp_setup.sql`, the
     `/backbone-mcp setup` job's file) for you to run as `ACCOUNTADMIN`, and stops — re-run `/backbone-publish` afterwards. It never runs `CREATE ROLE`;
   - asserts the connector role holds `SELECT` on the views and base tables (live `SHOW GRANTS`);
   - regenerates `mcp/generated/{tid}_mcp_server.sql` **only when the composed definitions
     changed** (the hand-authored `mcp/setup/{tid}_mcp_setup.sql`, now history, is left untouched) and
     applies the `CREATE OR REPLACE MCP SERVER` + `GRANT USAGE` as the tenant runtime role
     (`{TID}_LOADER`, #709); on a pre-#709 tenant it prints the DDL for you to run as
     `ACCOUNTADMIN`. Reconnect the client to surface a new tool;
   - lists the declared connector users (applied / pending / unknown).
4. Reports: headline (PASS / OPERATOR ACTION REQUIRED / FAIL), the publish result, then
   grant, reader role, MCP and connector users from the refresh, then security docs: when the
   security baseline (`docs/mcp/snowflake_security_baseline.md`) was last checked more than
   30 days ago it runs `/backbone-mcp <tid> check-docs` and adds its summary (advisory; it
   never changes the headline), then a **Lineage** line: after the refresh it redraws the
   tenant's lineage page (`/backbone-lineage <tid> from-publish`), which is advisory and never
   changes the headline.

**Usage:** `/backbone-publish <tenant_id>`

This is a **DML operation** (`CREATE OR REPLACE SEMANTIC VIEW`), so it sits behind a
hard approval gate identical to `/backbone-run`'s:

- **PROD (or any non-DEV target): refused outright.** No override, no confirmation
  path — mirroring the `main`/`prod` branch-protection rule.
- **DEV: runs only after an explicit one-line `run` confirmation.** The skill presents
  the target schema + the semantic model(s) + environment and waits.

The skill holds **no elevated credential** (no `ACCOUNTADMIN`): the recurring MCP-server
refresh runs as the tenant runtime role, account-level SQL is always handed to you, and it
does **not** claim it refreshed the client connector. If the tenant has no
built semantic model, the skill hard-errors with a redirect to `/backbone-apply {tid}`
— it never silently passes.

If no tenant ID is provided, fall back to the `BACKBONE_DEFAULT_TENANT_ID` environment
variable. If neither is set, ask the user which tenant to target.
