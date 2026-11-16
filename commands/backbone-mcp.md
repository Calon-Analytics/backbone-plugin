---
name: backbone-mcp
description: Manage a tenant's Claude MCP connector via the backbone-mcp skill — refresh re-grants the read-only connector role SELECT on the published views and their base tables, asserts the grant, refreshes the MCP server when its definitions changed and reports the connector users (DEV only, behind an explicit one-line confirmation; /backbone-publish runs it inside its own confirmation); setup writes the full one-off connector setup script, generated from the declaration, for a Snowflake admin to run as ACCOUNTADMIN; add-user declares a person (a dedicated read-only login or an existing login used only through Claude Code) and writes a numbered change script for a Snowflake admin to run; status lists declared users as applied, pending or unknown from live role membership, per environment; cutover writes the connector cutover change script once a green production run is evidenced (DEV moves to the DEV connector role) for a Snowflake admin; check-docs re-reads the Snowflake / Claude security docs against the recorded baseline and reports what changed (report only; publish runs it when the baseline is stale). Never handles a secret and never runs a change script.
argument-hint: <tenant_id> <refresh|setup|add-user|status|cutover|check-docs> [environment]
---

# /backbone-mcp

Run the `backbone-mcp` skill against tenant `$0` with job `$1`.

- **`refresh`** — the connector steps `/backbone-publish` used to run itself (#1047 T2):
  grants the read-only connector role `SELECT` on every published view and base table
  (#904; when the role is absent it writes the connector setup script — the `setup`
  job's file — for you to run as `ACCOUNTADMIN` and stops), asserts the grant live, refreshes the MCP server as the
  tenant runtime role when the definitions changed (#709), then lists the connector users.
  It is DML: **PROD (or any non-DEV target, named by the optional environment argument) is refused outright**, and DEV runs only
  after an explicit one-line `run` confirmation. `/backbone-publish` runs it as
  `/backbone-mcp <tid> refresh from-publish` inside the confirmation it already took — do
  not type `from-publish` yourself; it does not skip the gate outside a publish run.
- **`setup`** — calls `POST /api/config/{tid}/mcp/setup`, which writes the full one-off
  connector setup script, generated from the declaration, to
  `{estate root}/{tid}/mcp/generated/{tid}_mcp_setup.sql` (the first run is also kept as
  `changes/NNN_<date>_initial_setup.sql`). Have an admin run it as `ACCOUNTADMIN`; Backbone
  never runs it. The hand-authored `mcp/setup/` file is history.
- **`add-user`** — asks for the login, the person and the kind (`dedicated` or `existing`),
  calls `POST /api/config/{tid}/mcp/users`, and prints the change script it wrote under
  `{estate root}/{tid}/mcp/generated/changes/`. Run that script as `ACCOUNTADMIN` (or
  `USERADMIN` plus the connector role's owner), set sign-in for a dedicated login, then
  re-run `status`.
- **`status`** — calls `GET /api/config/{tid}/mcp/users` and lists each declared user as
  `applied`, `pending` or `unknown` (grants unreadable — never shown as pending).
- **`cutover`** — asks for the green production run's GitHub Actions URL and the
  engagement-team member who attests it, calls `POST /api/config/{tid}/mcp/cutover`, and
  prints the change script it wrote under `{estate root}/{tid}/mcp/generated/changes/`
  (409 `prod_run_not_evidenced` without that evidence and a team approval). Hand it to the
  client's Snowflake admin to run as `ACCOUNTADMIN`; Backbone never runs it.
- **`check-docs`** — re-reads the Snowflake and Claude pages behind the security baseline
  (`docs/mcp/snowflake_security_baseline.md`) and reports what **changed**, what is **new**
  since its `last_checked`, and what is **unconfirmed**, flagging any change that could
  unblock a parked item. It writes only `last_checked` unless you approve specific fact
  updates, and never edits code, templates or tenant files. `/backbone-publish` runs it when
  the baseline is more than 30 days old.

The skill resolves the estate root first (its step 0) and holds no Snowflake admin role:
every account-level change is a generated script an admin runs.

**Usage:** `/backbone-mcp <tenant_id> refresh` · `/backbone-mcp <tenant_id> setup` ·
`/backbone-mcp <tenant_id> add-user` ·
`/backbone-mcp <tenant_id> status` · `/backbone-mcp <tenant_id> cutover` ·
`/backbone-mcp <tenant_id> check-docs`
