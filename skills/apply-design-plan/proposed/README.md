# Proposed ledger-example candidates (staging area)

`/backbone-apply` **step 10** (`#653b`) stages ledger-shape candidates here on a
*tests-green* successful build, when it fingerprints a produced ledger whose shape
is not already in the canonical [`LEDGER_EXAMPLES.md`](../LEDGER_EXAMPLES.md).

- `candidate_<sig>.md` — a **novel** shape (no catalogue case matched).
- `maturation_<sig>.md` — a shape matching a **parked** `🔧`/`⚠️` case, now seen in
  a real tenant ledger → a `🔧→✅` promotion proposal.

Each file is tenant-agnostic (source / table / column vocab mechanically replaced
with placeholders; core-ontology target + role slugs kept), tagged with its
fingerprint and origin tenant, and named by fingerprint-signature hash so
re-applying the same tenant never re-stages the same shape.

**This is a review queue, not the registry.** The `#653c` checker (dynamic denylist
+ envelope gate + semantic agent) and `#653d` human/`grill-me` review decide what,
if anything, gets appended to the canonical file. Nothing here is authoritative.

The staging logic lives in the backend's `app/utils/ledger_shape_reducer.py`.

## `#653c` checker (gate → agent → auto-append)

The checker (the backend's `app/utils/ledger_example_checker.py`) gates each
candidate above into `LEDGER_EXAMPLES.md`. Two stages:

1. **Deterministic gate (code, hard-fail, never probabilistic).**
   - **Dynamic denylist** = the static base (connector names + tenant-ID shape,
     `app/utils/tenant_leak_denylist.py`) **∪** the originating tenant's own source /
     table / column vocab (read from `tenant_settings/{tid}/`, per the candidate's
     `origin_tenant`). Any surviving token → **hard-fail**.
   - **Envelope conformance** — the `fingerprint` block validates (known keys + known
     scope), the illustration matches its scope, required fields present.
2. **Semantic agent (judgment).** Input = the candidate. Output contract
   (`AgentVerdict`): `{novel: bool, faithful: bool, adr_touching: bool, reason: str}`.
   The agent only judges; it never writes files.

**Promotion decision table** (`decide_promotion`):

| deterministic | novel | faithful | adr_touching | → action |
|---|---|---|---|---|
| fail | – | – | – | withhold → `#653d` |
| pass | false | – | – | withhold → `#653d` |
| pass | true | false | – | withhold → `#653d` |
| pass | true | true | true | withhold → `#653d` |
| pass | true | true | false | **auto-append `🔧`** |

Auto-append (`append_case`) is **always `🔧`**, deduped by fingerprint signature,
numbered contiguously, round-tripped through `load_fingerprints`, and **written to the
working tree without committing** — the commit rides the `#653d` human/`grill-me`
review. The `🔧→✅` maturation flip is `#653d`, human-gated.

## `#653d` review lifecycle (human / `/grill-me`)

Whatever the `#653c` checker did **not** auto-append is left here for a person. The
apply report surfaces the count via `summarize_proposed()` — e.g.
`2 candidate(s) need /grill-me (1 novel, 1 maturation (🔧→✅)) — see proposed/`.

```
staged  ──►  /grill-me candidate review  ──►  promote | merge | reject | mature
```

- **promote** → `append_case()` (`ledger_example_checker.py`); if it warrants a
  governance change, make the ADR / `CONTEXT.md` edit **first**, then append the `🔧`
  case pointing home to it (upholds no-duplication).
- **merge** → the reviewer hand-edits the existing case in `LEDGER_EXAMPLES.md`, then
  `reject_candidate(..., "merged into case N")`.
- **reject** → `reject_candidate(candidate_path, reason)`
  (`ledger_example_maturation.py`) removes the file and appends a dated line to
  `REJECTED.md` (signature-hash + origin_tenant + reason). Idempotent — rejecting an
  already-gone candidate is a no-op.
- **mature (`🔧→✅`)** → `mature_case(case_number)` (`ledger_example_maturation.py`),
  **only after explicit human confirmation** of the `✅` (real + canonical) claim. It
  flips the case heading marker + the case-index status cell, round-trip-proven.
  `mature_case` is deliberately unreachable from any automated path.

All four write the **working tree only — none commit.** The appended / matured changes
ride **one deliberate reviewed commit** at the end of the pass, run by the reviewer
(respecting Branch Protection — never `main` / `prod`). See the grill-me skill's
`LEDGER_CANDIDATE_REVIEW.md` for the full mapping + commit discipline.

- `REJECTED.md` — the durable audit log of rejected candidates (created on first reject).
