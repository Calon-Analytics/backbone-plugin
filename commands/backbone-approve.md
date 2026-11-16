---
name: backbone-approve
description: Give a team approval for a promotion via the backbone-approve skill — shows the DEV evidence for the committed runtime/{tid} subtree hash, asks a named member of the engagement team to reply `approve`, commits approvals/NNN_<date>.yaml on the DEV branch, and prints the push and pull-request commands, running them only on the operator's `push`. Never merges and never touches main.
argument-hint: <tenant_id>
---

# /backbone-approve

Run the `backbone-approve` skill against tenant `$0`.

The skill resolves the estate root first (its step 0, `GET /api/estate`), shows the
evidence from `GET /api/config/{tid}/promotion/evidence` and stops on any refusal code.
A named member of the tenant's declared engagement team (`settings.yaml → engagement.team`)
replies `approve`; the skill then calls `POST /api/config/{tid}/promotion/approve`, which
commits `{estate root}/{tid}/approvals/NNN_<YYYY-MM-DD>.yaml` on the current DEV branch.
It prints the push and `gh pr create --base main` commands and runs them only when you
reply `push`.

Merging is the client admin's; the PROD run checks this approval's runtime hash.

The approval pins `runtime/{tid}`, `prod/` and the PROD workflow (#1196). It is a
proposal: the generated CODEOWNERS makes the client admin the owner of `approvals/`, so the
client holds the approval.

**Usage:** `/backbone-approve <tenant_id>`
