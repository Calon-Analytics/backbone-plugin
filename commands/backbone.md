---
name: backbone
description: The Backbone conductor (ADR-0042) — work out where a tenant stands in its lifecycle, print the position map, propose the next stage with its evidence, and on the operator's go run that existing stage (apply, run, test, publish) in place, then re-derive. Coordination only; writes nothing itself. A tenant id is required (no default tenant).
argument-hint: <tenant_id> [text]
---

# /backbone

Run the `backbone-conductor` skill against tenant `$0`. The command was
invoked with: `$ARGUMENTS`. The first whitespace-separated token is the tenant
id; pass any text after it through to the skill.

The conductor reads the tenant's derived lifecycle position, prints it, and
walks the existing stages one at a time, each only on your **go**. The stages
keep their own dialogues and DML confirmations. The six `/backbone-*` commands
remain the direct entry points to each stage
(ADR-0042
decision 4).

**Usage:** `/backbone <tenant_id> [text]`

A tenant id is **required**. Unlike the other `/backbone-*` commands, there is
no default tenant: with no tid the skill stops with this usage line. The
`[text]` argument is not supported yet (#1044 C1); the skill says so and stops.
