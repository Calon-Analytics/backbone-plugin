---
name: backbone-onboard
description: Drive a Backbone tenant from nothing (bare Snowflake DB, one source system per schema) to state 1 — connected, in-scope schemas registered as sources, a full source scan written — via the onboard-sources skill, then hand to /backbone-plan. Foreground Socratic HITL; detect-first resume; never touches the private key; works against a local or a hosted backend.
argument-hint: "[tenant_id]"
---

# /backbone-onboard

Drive a tenant from **zero** — only the shared core ontology and a bare
Snowflake database (one source system per schema) — to **state 1**, then hand
to `/backbone-plan`. State 1 is where `/backbone-plan` starts accepting:
connected → in-scope schemas registered as sources → a source scan written.
This is the pre-`plan` on-ramp — the first stage of the chain
**`onboard → plan → apply → run → publish`** (see
ADR-0025).

**Usage:**

- `/backbone-onboard` — no arg → the **clean path**: provision (operator-chosen
  tenant id; on a hosted backend Calon has provisioned it, so ask for the id) → connect hand-off → schema discovery → register sources → scan
  scope → full scan → state 1.
- `/backbone-onboard <tenant_id>` — **detect-first resume**: read the tenant's
  current state, work out which steps are already done, and drive only the gaps. A
  fully-set-up tenant is a no-op ("already at state 1").

When invoked, run the `onboard-sources` skill. It is a **foreground, Socratic,
HITL orchestrator** — ~2 human touchpoints on the clean path (the company
name + one batched registration preview; on a hosted backend, the tenant id Calon
gave you + that preview, plus the connect command you run yourself). It **never** touches the private key
(connect is a browser hand-off on a local backend, and a command you run yourself on
a hosted one) and **never** invents a loader `ldts` (an unknown loader pauses for
the human to author the block locally, and is "ask Calon" on hosted).

The skill takes local or hosted from `backbone_get_estate`; on hosted it uses MCP
tools only. The earlier hosted design is #722.

This command stops **exactly at state 1** — it does **no** entity / ontology /
binding / body / build work. That is `/backbone-plan` onward. When onboarding
finishes, run `/backbone-plan <tenant_id>` with your brief to author the design
plan.

If a tenant id is provided, resume that tenant. If not, on a local backend the skill
drives the clean path and asks for a company name to provision the new tenant; on a
hosted backend it asks for the tenant id Calon gave you.
