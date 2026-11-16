---
name: backbone-apply
description: Loop-aware apply dispatcher (ADR-0023) — strict batch, no UI clicks. Applies the vault plan via apply-design-plan ({estate root}/{tid}/configs/design_plan.md), then every compiled consumption plan via apply-consumption-plan (configs/consumption/*.md → ledgers → the three generated configs). Deferred consumption plans are named, never auto-compiled. Requires a design plan (run /backbone-plan first).
argument-hint: <tenant_id>
---

# /backbone-apply

Apply the plan(s) for the supplied tenant. One verb, two loops
(ADR-0023 §1);
both plans are authored by `/backbone-plan`
(ADR 0001).

**Usage:** `/backbone-apply <tenant_id>`

## Agree the estate first (#971)

**No tenant ID given?** Call `backbone_get_estate()` with an empty `tenant_id`
first, and take the tenant from its report's `mcp.default_tenant_id` — the
`BACKBONE_DEFAULT_TENANT_ID` the running MCP server holds, **not** the shell
environment. Then run the check below with that tid, and name the chosen tid in
the estate line (*"… tenant `<tid>` (from the MCP server's default)"*). If
`mcp.default_tenant_id` is `null`, ask the user which tenant to target.

<!-- estate-agreement:begin (#971) -->
**0a. Estate agreement — before any other read or write (#971).**

Three things must name **the same estate**: the REST origin this session reads
through, the backend the `backbone-snowflake` MCP server writes through (its
`BACKBONE_API_URL`), and the estate that MCP process resolves concepts in (its
own `TENANT_SETTINGS_PATH` — `backbone_resolve_core_concept` reads the tenant
library in-process on a local server; a remote one asks the backend and needs
`tenant_id`). Every apply write goes through
the MCP server, whose environment is fixed when Claude Code launches it, so a
disagreement cannot be repaired in-session. Check it first.

1. **Define `$BACKBONE`.** It is the origin the operator names in the
   invocation (*"the backend on port N"* ⇒ `http://localhost:N`). If they name
   none, it is the MCP server's own `mcp.api_url` as the tool reports it. Every
   REST call in this run — the estate-root step below included — goes to
   `$BACKBONE` and nowhere else. Never take it from a skill default or a shell
   variable.
2. **Call `backbone_get_estate(tenant_id=<tid>, rest_url=$BACKBONE)`** — with
   `rest_url` empty when the operator named no origin, so the tool compares the
   MCP backend with itself and with its local estate. It is read-only and
   issues GET only.
   - **The tool is missing** (the running MCP server predates #971): STOP —
     *"The `backbone-snowflake` MCP server has no `backbone_get_estate` tool; it
     is older than this skill. Please fully quit and relaunch Claude Code, then
     re-run."* Never skip the check because the tool is absent.
3. **`agree: false` → STOP. Nothing has been read or written.** Print every
   `blocking` finding, and the `.mcp.json` edit built from `mcp_json_env_fix`:

   > **Estate disagreement: nothing has been read or written.**
   > REST `$BACKBONE` serves estate `<rest id>` at `<rest tenant_root>`. The MCP
   > server writes to `<mcp api_url>`, which serves `<backend id>` at
   > `<backend tenant_root>`, and it resolves concepts under
   > `<local tenant_root>`.
   > Blocking: `<code>`: `<detail>` (one line each).
   > Fix: in `.mcp.json` → `mcpServers.backbone-snowflake.env`, set
   > `"BACKBONE_API_URL": "<…>"`, `"TENANT_SETTINGS_PATH": "<…>"`,
   > `"BACKBONE_DEFAULT_TENANT_ID": "<tid>"` (only the keys `mcp_json_env_fix`
   > carries).
   > Then **fully quit and relaunch Claude Code**. A reconnect does not re-read
   > the server's environment. Re-run the same command for `<tid>`.

   When `mcp_json_env_fix` is `null`, no env edit fixes it: name the backend or
   tenant to fix instead (start the backend, correct the origin, provision the
   tenant). The agent never edits `.mcp.json` itself. If the operator meant the
   *other* estate, they start that estate's backend, or name its origin, and
   re-run.
4. **`default_tenant_differs`** (severity `info`) → print it as a one-line note,
   offer `BACKBONE_DEFAULT_TENANT_ID`, and continue. Write tools given an
   explicit tenant id are unaffected.
5. **`agree: true`** → state one line (*"Estate agreed — REST, MCP backend and
   MCP concept resolution all serve `<id>` at `<tenant_root>`."*) and continue
   with the estate-root step, addressing `$BACKBONE`.

| Finding code (all `blocking`) | Meaning |
|---|---|
| `mcp_local_mode` | The MCP server has no `BACKBONE_API_URL`, so its writes have no backend |
| `estate_mismatch` | `$BACKBONE` serves a different estate (`id` or `tenant_root`) from the MCP backend |
| `local_estate_mismatch` | The MCP process resolves concepts under a different root from its backend's |
| `local_estate_unresolvable` | The MCP process cannot build its own estate (e.g. a bad `ESTATE_ID`) |
| `estate_root_unresolvable` | The MCP backend is hosted (`tenant_root` null) — no estate root to plan under |
| `mcp_backend_unreachable` / `mcp_backend_http_error` / `mcp_backend_invalid_response` | The MCP backend did not answer `GET /api/estate` as a Backbone backend |
| `rest_backend_unreachable` / `rest_backend_http_error` / `rest_backend_invalid_response` | `$BACKBONE` did not answer `GET /api/estate` as a Backbone backend |
| `rest_url_invalid` | `$BACKBONE` does not start `http://` or `https://` |
| `tenant_not_in_estate` | `<tid>` is not in the MCP backend's `GET /api/config/tenants` |
| `tenant_list_failed` | That tenant list could not be read |

The check runs in the dispatcher and again in the sub-skill. When the
dispatcher already ran it for the same `<tid>` this turn and it agreed, the
sub-skill still calls the tool but says *"estate already agreed this turn"* and
skips the repeat announce.
<!-- estate-agreement-hosted:begin (BB-1217) -->
**0h. Hosted estate (BB-1217).** When the report shows `mcp.mode` `remote` with
`mcp.api_url` `null` (the tool's own hosted marker: it nulls the URL only on a
hosted server), the backend is the shared hosted Backbone service, not an estate on
this machine. Steps 1 and 3 to 5 above are read with these changes, and nothing
else in the block changes:

- **MCP tools only.** There is no `$BACKBONE` (the tool reports `mcp.api_url`
  as `null`): make no REST or curl call, and read or write no `{estate root}`
  path. Everything goes through the Backbone MCP tools.
- **Skipped on hosted:** the `$BACKBONE` step, the local-estate and
  estate-root checks, and the *"Resolve the estate root"* step, including its
  STOP on a null `tenant_root` (the hosted tool always returns it as `null`).
- **Agreement** rests on one fact, `tenant_in_estate`, the tenant being listed
  by the backend's `GET /api/config/tenants`:
  - `true` → say *"Hosted estate agreed — tenant `<tid>` is served by the
    hosted Backbone."* and continue. Print no root.
  - `false` → STOP: *"Tenant `<tid>` is not on the hosted Backbone. Check the
    tenant id, or ask Calon to add it."*
  - `null` (the list could not be read) → not the same as "not listed". STOP
    and report the finding below.
- **Hosted refusals** are told in plain words, never as a stack trace or a raw
  payload:
  - `mcp_backend_http_error` → *"The hosted Backbone refused the request
    (`<status>`). Your access token may be wrong or revoked: ask Calon for a
    new token."* Show the finding's detail.
  - `tenant_list_failed` → *"The hosted Backbone could not list its tenants, so
    `<tid>` cannot be confirmed. Try again; if it persists, tell Calon."*
- Never ask for, print, store or pass on the Backbone token: the plugin sends
  it to the configured Backbone URL itself, and no command in this run puts it
  in a curl call, a file, an environment variable or a message.
<!-- estate-agreement-hosted:end -->
<!-- estate-agreement:end -->

## Resolve the estate root first (#886, ADR-0038 §2)

The estate root is whatever `TENANT_SETTINGS_PATH` the backend you are
talking to was started with (ADR-0038 §2, card #869) — `<repo>/tenant_settings`
for a Calon-account tenant, `/calon/clients/{client}` for a client engagement.
Never assume the literal `tenant_settings`; ask the backend with
`GET /api/estate` (`tenant_root`) when you need the absolute path.

State it in one line before the first read or write, so the operator can see
which client this run is pointed at:

> *"Estate `<id>` (`<label>`) — tenant root `<tenant_root>`. This run reads
> and writes under `<tenant_root>/<tid>/`."*

**On a hosted estate (`mcp.mode` `remote` with `mcp.api_url` `null`) this whole section is skipped**
(block step 0h).

**If the estate cannot be resolved, STOP.** `GET /api/estate` unreachable,
a null `tenant_root` (hosted mode), or no `<tid>` directory beneath it:
*"Cannot resolve the estate root for `<tid>` — `GET /api/estate` `<reason>`.
Start the backend on the right estate (`TENANT_SETTINGS_PATH`) and re-run."*
**Never fall back to `<repo>/tenant_settings`** — that silently reads, and
on an apply *writes*, another client's tenant.

**On hosted:** a null `tenant_root` is not a STOP. When `backbone_get_estate`
reported a hosted estate (block step 0h), the run is in **remote mode**: skip
this section and continue with the Backbone MCP tools only. The sub-skills
carry the same rule, and each step's hosted tool, in their step 0.

**On hosted, SQL is shown before it is written (BB-1301).** Before the first
write of each plan it applies, the sub-skill shows one consolidated SQL review
(every source filter, transform, flatten filter, formula and derived expression
the plan carries, each with where it is written, its origin and the exact value,
and a warning on one that reads another relation) and waits for one `go` from
the operator. One review per plan, never a prompt per field: its step 0
**Hosted SQL review**. On a local estate nothing changes.

Below, `{estate root}` is the resolved `tenant_root`, which the sub-skills
resolve the same way.

## Step 1 — the vault plan → `apply-design-plan`

**On hosted:** the skill reads the stored plan with `backbone_get_design_plan`
and writes only through the hosted tools its steps name; each writing stage
prints the repository `head`.

Run the `apply-design-plan` skill against tenant `$0`. The skill reads
`{estate root}/{tid}/configs/design_plan.md`, validates it at its read site
(the plan-contract gate), calls the MCP write tools
(`backbone_create_entity`, `backbone_auto_configure_sources`,
`backbone_stamp_body_config`, `backbone_start_build`), applies any
per-binding overrides, and polls build status until terminal.

If `design_plan.md` is missing or invalid, the skill hard-errors with a
redirect to `/backbone-plan {tid}`. Stop there: consumption is grounded in
the vault contract and cannot be applied ahead of it.

## Step 2 — the consumption plans → `apply-consumption-plan`

**On hosted:** list the plans with `backbone_list_consumption_plans(tenant_id)`;
a completed build is read from `backbone_get_build_status`.

When step 1 reports a **completed** build (or the tenant already had one and
`design_plan.md` is unchanged), list `{estate root}/{tid}/configs/consumption/*.md`:

- for every plan **without** `status: deferred`, run `apply-consumption-plan`
  against `$0` and that `use_case` — read-site validation, then the single
  strict-batch endpoint that compiles the plan to `grain__*` / `fact__*` /
  `metric__*` / `product__*` ledgers and renders `bv_integration_config`,
  `business_rules_config` and `metrics_mart_config` from them
  (ADR-0031 §1/§4);
- for every plan **with** `status: deferred`, do **not** apply it; say:

  > *"`{use_case}` was captured before the vault existed and has not been
  > compiled — re-run `/backbone-plan {tid}` to compile it, then
  > `/backbone-apply {tid}` again."*

  This is the ADR-0023 §3 hand-back: the contract is now frozen while
  deferred consumption intent exists, and the operator — not the dispatcher —
  decides to compile it.

A consumption plan that fails its read-site check hard-errors with the
findings verbatim and a redirect to `/backbone-plan {tid}`; the other plans
are still applied. Nothing in step 2 runs dbt.

**On hosted, the dispatched skill skips its own step 3b** (its build): this
dispatcher builds once for the whole consumption half, in step 2b, so several
plans never mean several builds.

## Step 2b — one build for the consumption half

**On hosted:** the vault build of step 1 predates every consumption apply, so it holds no consumption model; without a further build the run would materialise the vault only. Before the first consumption plan, call `backbone_repo_status(tenant_id)` and note its `local_sha`. After the last plan, the consumption half changed something when any apply's receipt `commit_sha` differs from that `local_sha`, or any apply returned `bv_integration_config_changed` or `consumption_configs_changed` true, a `definition_versions_bumped` entry or a `conditions_removed` entry. Then start one build, once, after the last plan: `backbone_start_build(tenant_id)`, polled with `backbone_get_build_status(tenant_id)` every 15 seconds for up to 60 minutes, as apply-design-plan step 8 does (terminal when `is_building` is false and `last_build_id` equals your `build_id`). Report the `build_id`, `last_build_status`, its `repo_sync`, and the seal: the `build_seal` evidence `backbone_get_lifecycle_position(tenant_id)` returns. End the stage with its receipt: print `commit_sha` and `push`.

- **When the consumption half changed nothing** (no plan applied, every plan
  deferred or refused, or every apply left the repository as it was), start
  no build: say *"No consumption change: no build started; build `<id>` from
  step 1 stands."*
- **A failed build** is handled as apply-design-plan step 8's *A failed build*;
  a message naming a consumption rule is a plan defect, redirected to
  `/backbone-plan {tid}`.

The vault half keeps its own build (apply-design-plan steps 7 and 8): a
consumption plan is checked and applied against that built vault, and the
vault stage's post-build gate and findings read it, so it cannot be deferred
to this step. One apply therefore builds twice on hosted when consumption
changed: the vault build, then this one, which carries both.

On a local estate this step is skipped, and the local flow is unchanged: the
hand-off below names the build.

## Step 3 — report and hand off

**On hosted:** report each stage's receipt (`commit_sha` and `push`), the vault build and the step 2b build (`build_id`, status, `repo_sync`, seal) or that none was needed. Then hand off to `/backbone-run <tid>`, which runs the latest build (vault and consumption models) as a run request on the hosted Backbone; `/backbone-test <tid>` and `/backbone-publish <tid>` follow it, and `/backbone <tid>` shows where the tenant stands. Do not start the run.

Report the vault build id and status, then per consumption plan what was
written / unchanged / preserved and whether each generated config changed.
Then:

> Build complete for tenant `{tid}`. The vault and the consumption models
> are generated but not materialised. Run `/backbone-run {tid}` to execute
> against Snowflake, `/backbone-test {tid}` to reconcile, and
> `/backbone-publish {tid}` to expose the governed semantic view.

This command **does not** chain into authoring, into `/backbone-run`, or
into `/backbone-publish`. Re-planning and re-execution are separate cadences
per ADR 0001.

If no tenant ID is provided, take `mcp.default_tenant_id` from
`backbone_get_estate()` (see *Agree the estate first*) — never the shell
environment. If it is `null`, ask the user which tenant to target.
