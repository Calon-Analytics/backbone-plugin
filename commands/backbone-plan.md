---
name: backbone-plan
description: Loop-aware planning dispatcher (ADR-0023). Classifies a brief into vault intent and consumption intent, confirms the split with the operator, then compiles each half — vault via compile-design-plan ({estate root}/{tid}/configs/design_plan.md), consumption via compile-consumption-plan (configs/consumption/{use_case}.md). Authoring only — execution is /backbone-apply.
argument-hint: <tenant_id> [brief | @path/to/brief.md]
---

# /backbone-plan

Compile a tenant's brief into its plan(s). One verb, two loops
(ADR-0023 §1):

| Intent in the brief | Sub-skill | Plan written |
|---|---|---|
| **Vault / source** — entities, source systems, tables, business keys, filters, FKs | `compile-design-plan` (Loop 1) | `{estate root}/{tid}/configs/design_plan.md` |
| **Consumption** — questions to answer, measures, metrics, data products, business rules, time policy | `compile-consumption-plan` (Loop 2) | `{estate root}/{tid}/configs/consumption/{use_case}.md` |

Both plans live **inside `configs/`** so the demo-state snapshot tools pick
them up. Both are the durable artefacts of declared intent — the contracts
`/backbone-apply` reads (ADR 0001,
ADR-0031 §1).

**Usage:** `/backbone-plan <tenant_id>`

## Agree the estate first (#971)

If no tenant ID was given, ask which tenant to target **before** this check —
there is no silent default for authoring. Then:

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

**On hosted, SQL is shown before it is written (BB-1301).** Each sub-skill shows
every source filter, transform, flatten filter, formula and derived expression
as a card (the field, where it is written, the clause it renders as, its origin
and the exact value), flags one that reads another relation, and writes the plan
only after the operator answers for it: its step 0 **Hosted SQL review**. On a
local estate nothing changes.

Below, `{estate root}` is the resolved `tenant_root`, which the sub-skills
resolve the same way.

The command was invoked with: `$ARGUMENTS`. The first whitespace-separated
token is the tenant id (`$0`), and only that token. Anything after it is the
brief: inline text ("we need a DW for products and salesorders, and we want
order value by channel by month"), or an `@path/to/brief.md` reference, which
is the brief file and never a tenant id. With nothing after the tenant id, the
brief comes from the conversation context.

## Step 0 — classify and split (ADR-0023 §3)

Before invoking any sub-skill, read the whole brief and partition it:

- **Vault intent** — anything that names *what exists and where it comes
  from*: business entities, source systems, tables, keys, source filters,
  relationships.
- **Consumption intent** — anything that names *what the business wants to
  know*: a question, a measure ("order value"), a metric, a data product, a
  business rule, a date that should drive an answer ("by closed date").

Print the split as two short lists, one line per item quoting the brief's
own words, and ask the operator to confirm — **automate, then confirm**:

> **Vault:** "products and salesorders from the ecommerce and warehouse
> systems" · "orders keyed by order number"
> **Consumption:** "order value by channel by month" · "count of orders"
> Move anything I have put in the wrong loop, or say "go".

A brief that carries only one kind of intent skips the question but still
says which loop it is taking and why. An item the operator moves is moved;
the split is theirs, not the classifier's. A mixed brief is good practice
(ADR-0023 context) — never ask the operator to pre-split it.

## Step 1 — the vault half → `compile-design-plan`

Run `compile-design-plan` against tenant `$0` with the vault half of the
brief. The skill walks the brief one entity at a time, resolves each
concept against the tenant ontology, confirms source bindings table by
table, and surfaces aligned-but-unmapped tables. If `design_plan.md` already
exists it runs in incremental mode.

## Step 2 — the consumption half → `compile-consumption-plan`, or capture it

**On hosted:** the built-vault test is `backbone_get_vault_contract`
(`not_found` means none is frozen). A deferred capture is stored with
`backbone_put_consumption_plan(tenant_id, use_case, text)`, never written as a
file. End the stage with its receipt: print `commit_sha` and `push` from that
write's response, and call `backbone_repo_status` only when they are missing.

Consumption is grounded in the **frozen vault contract**, never the source
scan. So:

- **The tenant has a built vault** (`backbone_get_tenant_context` shows a
  build; `configs/vault_manifest.yaml` or a green build) → run
  `compile-consumption-plan` against `$0` with the consumption half. One
  `use_case` per consumer / question cluster; the skill names it.
- **No built vault yet** (a cold-start brief, or step 1 just planned new
  entities the vault does not hold) → **capture, do not compile.** Write
  `configs/consumption/{use_case}.md` with the `## Brief` section only and a
  plan block of `use_case: {use_case}` + `status: deferred`, and say:

  > *"Captured `{use_case}` — it compiles after the vault builds. Run
  > `/backbone-apply {tid}` then `/backbone-run {tid}`, and re-run
  > `/backbone-plan {tid}` to compile it."*

  Never auto-compile on a later freeze; the operator re-runs, the dispatcher
  detects readiness (below).

## Step 3 — readiness re-check (deferred intent)

**On hosted:** list with `backbone_list_consumption_plans(tenant_id)` and read
each with `backbone_get_consumption_plan`.

On every run, list `configs/consumption/*.md` and read each plan's status:

- a plan with `status: deferred` on a tenant that **now has** a built vault
  → tell the operator and compile it now via `compile-consumption-plan`
  (incremental mode: the skill reads the captured brief and asks "what's
  changing?" with the vault contract in hand);
- a plan with `status: deferred` on a tenant that still has no vault → say so
  and leave it.

## What this command does not do

- It **does not** call any MCP write tool, ledger endpoint, render endpoint or build.
  Authoring and execution are separate surfaces (ADR 0001). The only writes it makes are
  `compile-design-plan` step 3.5's operator-confirmed source and loader registrations (#971).
- It **does not** add a mode flag. Classify-and-split replaces
  `/backbone-plan consumption` deliberately (ADR-0023 alternatives rejected).
- It **does not** chain into `/backbone-apply`.

Run `/backbone-apply <tenant_id>` to execute the plan(s).

If no tenant ID is provided, ask the user which tenant to target.
