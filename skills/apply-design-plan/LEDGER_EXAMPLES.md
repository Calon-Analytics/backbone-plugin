# Backbone Ledger Examples — canonical YAML per case

The canonical, tenant-agnostic shape of the Backbone **ledger SSOT**
(`tenant_settings/{tid}/configs/ledgers/{source}__{table}__{entity}.yaml`) and its
paired **body** markers, for each case where the ledger shape is load-bearing.

**This file holds examples, not rules.** The *why* (decision) lives in the ADRs, the
*what a term means* lives in `CONTEXT.md`, the *how the skill does it* lives in the
`compile-design-plan` / `apply-design-plan` skill steps. Each case below points **home**
to those; it never restates them. The asset here is the concrete YAML — the thing that
has no other home and that an agent cannot talk itself out of the way prose can.

**Tenant-agnostic by rule.** Every example uses placeholders (`ECOMMERCE` / `CRM` /
`ERP` / `WAREHOUSE` / `GENERIC`, tables `ORDER` / `ORDER_LINE` / `PRODUCT` / `CONTACT`,
`<TID>`) and core-ontology slugs (`party` / `contact` / `salesorder` / `salesorderline`
/ `product` / `currency`). Never a real connector name, tenant ID, or tenant column.

_Examples verified against real ledgers (abstracted); spec-derived cases are flagged._
_New ledger-shaping precedents are appended here (checker-gated) as they are identified._

**Each case also carries a machine-readable ```` ```fingerprint ```` block** — a YAML inline
mapping of the load-bearing markers that distinguish it, in a form a produced ledger can be
reduced to and compared against mechanically (#653b). `scope` says whether the markers
describe one `column`, the whole `file`, or a `file+multiplicity` shape (a count of columns).
Every key maps to a real ledger/body field — see
`app/utils/ledger_example_fingerprint.py` for the exact field-derivation contract.

---

## Top-level envelope

Every ledger file carries this root shape. Load-bearing keys are marked `←`.

```yaml
tenant_id: <TID>
source: ECOMMERCE                 # connector / source-system name
table: ORDER_LINE                 # source table
entity_domain: salesorderline     # ← core-ontology entity slug this table binds to
table_type: unit_of_work          # ← 'hub' | 'unit_of_work'
primary: false                    # ← true only for the master of a multi-table binding
source_filter: null               # ← SQL WHERE clause, or null
columns:                          # ← map of COLUMN_NAME -> column spec (below)
  ORDER_ID: { ... }
metadata_overrides:               # optional; only when a loader default is overridden
  ldts: LOADTIMESTAMP
```

Per-column spec (inside `columns:`). Optional keys appear only when relevant:

```yaml
COLUMN_NAME:
  kind: entity_key                # ← 'entity_key' | 'relationship_key' | 'attribute'
  target: party                   # ← entity slug (entity_key/relationship_key) | 'main' (attribute)
  fk_to_table: null               # ← null = x-direct (default); TABLE = prejoin (exception)
  role: null                      # role slug for a role-qualified binding, else null
  business_key: true              # true for the hub key column(s)
  primary_key: true
  source: auto_configure
  fk_to_column: null              # prejoin only: the referenced table's join PK
  transform: null                 # optional: raw-SQL hard rule (ADR-0013)
  transform_origin: null          # optional: 'plan' | loader-rule name
```

Paired **body** file (`body_<SOURCE>_<TABLE>__<ENTITY>.yaml`) is an index-keyed dict;
the markers that matter downstream are `hk_<entity>_h`, `metadata`, and `lnd_view`:

```yaml
0:
  original: ORDER_ID
  source_column: ORDER_ID
  lnd_view: salesorder            # landing view / target-entity view
  hk_salesorder_h: order_code_bk  # hub hash-key marker: x | x:<role> | <fetched-bk>
  metadata: rk                    # bk | rk | tk | dt | amt
```

---

## 1 · Attribute (baseline / contrast)

Plain descriptive column; lands as a satellite attribute, contributes to no hub key.

**Home:** CONTEXT.md "Derived artifact"

```yaml
# ledger — columns:
FIRST_NAME:
  kind: attribute
  target: main            # ← 'main' = the entity's own satellite, not a foreign key
  fk_to_table: null
  business_key: false
  primary_key: false
```
```yaml
# body — no hub marker; only the satellite hashdiff marker
original: FIRST_NAME
lnd_view: contact_first_name
hd_ecommerce_contact_party_s: x
```
Load-bearing: `kind: attribute`, `target: main`.

```fingerprint
{scope: column, kind: attribute, target: main, metadata: none}
```

---

## 2 · Hub-forming binding, single declared BK

The declared business key for a hub. `entity_key` + `hk_<entity>_h: x` is what makes
the source load the hub.

**Home:** CONTEXT.md "Source binding" (declared-not-guessed) · ADR-0013 · compile B6/B12 · apply step 4a

```yaml
# ledger — CRM__CONTACT__party, columns:
EMAIL:
  kind: entity_key
  target: party
  fk_to_table: null
  role: null
  business_key: true
  primary_key: true
```
```yaml
# body:
original: EMAIL
lnd_view: party_code
hk_party_h: x            # ← x = this column's value IS the party business key
metadata: bk
```
Load-bearing: `kind: entity_key`, `target`, `hk_<entity>_h: x`, `metadata: bk`.

```fingerprint
{scope: column, kind: entity_key, target: set, fk_to_table: null, role: null, hk: x, metadata: bk, cardinality: single}
```

---

## 3 · Composite BK  🔧 shape verified against code (no committed tenant ledger yet)

Two (or more) columns jointly key the hub — each a **separate** `entity_key` entry with the
same `target`. Declared as a list at the API (`business_keys: [A, B]`), stored as N
`entity_key` columns via the `bk_column_names` stamping path. **Column order is
significant** — the generator hashes `MD5(A || delim || B)`, so `[A,B] ≠ [B,A]`.

**Home:** backbone-engineer memory #582 · CONTEXT.md "Business key" · `auto_configure_service._build_autoconfig_ledger_columns` (~L1531)

```yaml
# ledger — columns: (each member is its own entity_key, same target)
ORDER_CODE:
  kind: entity_key
  target: salesorder
  fk_to_table: null      # entity_key MUST be null (validator-enforced)
  business_key: true
  primary_key: true
STORE_ID:
  kind: entity_key
  target: salesorder
  fk_to_table: null
  business_key: true
  primary_key: true
```
```yaml
# body — ONE shared hub marker set on EVERY member column (not one hk per column):
#   ORDER_CODE -> hk_salesorder_h: x, metadata: bk
#   STORE_ID   -> hk_salesorder_h: x, metadata: bk
```
Load-bearing: ≥2 `entity_key` columns, same `target`, each `fk_to_table: null`; the body
sets the single `hk_<entity>_h: x` on every member; **member order is preserved**.

```fingerprint
{scope: file+multiplicity, kind: entity_key, target: set, fk_to_table: null, role: null, hk: x, metadata: bk, cardinality: composite}
```

---

## 4 · Role-qualified hub

A source whose grain is a *role's* population (keyed by its own BK) binds to the
played-by entity and role-qualifies the key → `hub_<playedby>_<role>`. No `<role>`
entity is ever created.

**Home:** ADR-0014 · CONTEXT.md "Role-qualified hub" · apply step 3/4a

```yaml
# ledger — CRM__CONTACT__party, columns:
EMAIL:
  kind: entity_key
  target: party
  fk_to_table: null
  role: contact          # ← role qualifier; target stays the played-by entity
  business_key: true
  primary_key: true
```
```yaml
# body:
original: EMAIL
lnd_view: party_contact_code
hk_party_h: x:contact    # ← x:<role> → hub_party_contact
metadata: bk
```
Load-bearing: `role:` on the entity_key, `hk_<playedby>_h: x:<role>` in the body.

```fingerprint
{scope: column, kind: entity_key, target: set, fk_to_table: null, role: set, hk: "x:role", metadata: bk, cardinality: single}
```

---

## 5 · x-direct relationship_key  (the DEFAULT)

A line/event column whose *value is itself* the target's business key. The staging
hashes it into `hk_<target>_h` and registers the line in `hub_<target>.source_models`,
so the referenced hub loads from the line. `fk_to_table: null` is what makes it direct.

**Home:** ADR-0008 · CONTEXT.md "Relationship-key wiring" · apply step 5c

```yaml
# ledger — ECOMMERCE__ORDER_LINE__salesorderline, columns:
SKU:
  kind: relationship_key
  target: product
  fk_to_table: null      # ← null = x-direct
  fk_to_column: null
  role: null
```
```yaml
# body:
original: SKU
lnd_view: product_code
hk_product_h: x          # ← x = value is the product BK
metadata: bk
```
Load-bearing: `kind: relationship_key`, non-null `target`, `fk_to_table: null`,
`hk_<target>_h: x`, `metadata: bk`.

```fingerprint
{scope: column, kind: relationship_key, target: set, fk_to_table: null, fk_to_column: null, role: null, hk: x, metadata: bk}
```

---

## 6 · Prejoin relationship_key, line side  (the EXCEPTION)

The line carries a *reference* whose real BK lives in another same-schema table; fetch
it across a join. `fk_to_table` (the referenced table) + `fk_to_column` (its join PK)
are both non-null — the two fields that turn a direct key into a prejoin.

**Home:** ADR-0008 / ADR-0010 · CONTEXT.md "Relationship-key wiring" · apply step 5c

```yaml
# ledger — CRM__PRODUCT_ORDER__salesorderline, columns:
NAME:
  kind: relationship_key
  target: product
  fk_to_table: PRODUCT   # ← non-null = prejoin to this same-schema table
  fk_to_column: ID       # ← the referenced table's join PK
  role: null
```
```yaml
# body:
original: NAME
lnd_view: product
hk_product_h: product_code_bk   # ← the FETCHED bk, not x
metadata: rk
```
Load-bearing: `fk_to_table`, `fk_to_column`, `hk_<target>_h: <fetched-bk>`, `metadata: rk`.

```fingerprint
{scope: column, kind: relationship_key, target: set, fk_to_table: set, fk_to_column: set, role: null, hk: fetched, metadata: rk}
```

---

## 7 · rk on the referenced/target table side

The counterpart to case 6: the referenced table's own join column is pinned so both
sides of the prejoin alias to the same `{entity}_rk`. Written by the sibling-ledger
post-pass, never duplicated by hand. The join identity carries **no transform**.

**Home:** ADR-0010 · CONTEXT.md "Relationship-key wiring" (join-identity-no-transform) · apply step 5d/6

```yaml
# body — the line referencing salesorder pins the join key:
original: ORDER_ID
order: 1
lnd_view: salesorder     # ← target entity view
hk_salesorder_h: order_code_bk
metadata: rk             # ← rk marks the relationship-key join from the line side
```
Load-bearing: `metadata: rk`, `lnd_view: <target>`, raw (untransformed) join column.

```fingerprint
{scope: column, kind: relationship_key, target: set, fk_to_table: null, fk_to_column: null, role: null, hk: fetched, metadata: rk, transform: absent}
```

---

## 8 · Unit-of-work line, FK-only (no BK)  ✅ verified real

A transactional line identified purely by the entities it connects — `relationship_key`
columns + attributes, **zero `entity_key`**. Legitimately needs no business key; the link
`hk_<entity>_l` is formed over its FKs. Naming a non-key column as a BK here would wrongly
turn the line into a hub.

**Home:** CONTEXT.md "Unit-of-work line" · ADR-0003 · compile B6

```yaml
# ledger — GENERIC__RECONCILIATION__salesorderline (whole-file shape):
tenant_id: <TID>
source: GENERIC
table: RECONCILIATION
entity_domain: salesorderline
table_type: unit_of_work
primary: false
source_filter: IS_SUMMARY = FALSE
columns:
  ORDER_ID: { kind: relationship_key, target: salesorder, fk_to_table: null }
  SKU:      { kind: relationship_key, target: product,    fk_to_table: null }
  # ...attributes... (no entity_key anywhere)
```
Load-bearing: `table_type: unit_of_work`, ≥1 `relationship_key`, **no** `entity_key`.

```fingerprint
{scope: file, table_type: unit_of_work, has_entity_key: false, has_relationship_key: true}
```

---

## 9 · Hard-rule transform on a column

A deterministic raw-SQL transform declared once in the plan and carried on the ledger
column (`transform` + `transform_origin`), stamped into the body `original` at apply.
Scalar, row-level, single-table only.

**Home:** ADR-0013 · CONTEXT.md "Hard rule" · compile B16 / apply step 5e

```yaml
# ledger — columns:
CHANNEL_DESCRIPTION:
  kind: attribute
  target: main
  transform: coalesce(CHANNEL_DESCRIPTION, CHANNEL_NAME)   # ← raw SQL
  transform_origin: plan                                    # ← 'plan' wins over loader-rule
```
Load-bearing: `transform` (SQL string), `transform_origin` (provenance).

```fingerprint
{scope: column, kind: attribute, target: main, metadata: none, transform: present, transform_origin: set}
```

---

## 10 · Loader metadata override

Per-table override of the loader's `ldts` / `smdt` / `etl_runid` when the loader default
column isn't present/correct on this table. A non-column expression additionally requires
a `*_override_reason`.

**Home:** ADR-0009 · CONTEXT.md "Load timestamp (ldts)" · compile B15 / apply step 4e

```yaml
# ledger — top level:
metadata_overrides:
  ldts: LOADTIMESTAMP                    # a real column on this table
  smdt: LOADTIMESTAMP
  etl_runid: to_varchar(LOADTIMESTAMP)   # may be an expression
```
```yaml
# last-resort form (no load column exists) — reason is mandatory:
metadata_overrides:
  ldts: current_timestamp
  ldts_override_reason: no load-date column landed on this table
```
Load-bearing: `metadata_overrides` block; `*_override_reason` for any non-column value.

```fingerprint
{scope: file, metadata_overrides: present}
```

---

## 11 · Source filter on a binding

A row filter pinned to one binding, preserved verbatim; validity is enforced by the
executor (422 on a bad filter), not re-parsed at author time.

**Home:** CONTEXT.md "Source binding" · compile B9 / apply step 5b

```yaml
# ledger — top level:
source_filter: channel_name = 'Despatched' and is_summary = false
```
Load-bearing: top-level `source_filter` (nullable SQL WHERE clause).

```fingerprint
{scope: file, source_filter: present}
```

---

## 12 · Multi-table primary / master  ⚠️ built but parked (#617; not live-proven)

When one source binds >1 table to one entity, the master ledger is marked top-level
`primary: true`. This designates the **canonical source-table alias for FK resolution**
(which of the source's several tables an FK to this entity names in `fk_to_table`) and
triggers **sibling demotion** — every other ledger for the same `(entity_domain, source)`
is auto-rewritten to `primary: false` (`ledger_primary_demote.demote_sibling_primaries`).
It does **NOT** gate hub loading or BK selection — the hub still loads from whichever
ledgers carry `entity_key` columns.

Top-level `primary` (table master) is distinct from column-level `primary_key` (a BK flag).

**Home:** compile B8 · apply step 2/5b · `ledger_primary_demote` · `auto_configure_service._build_domain_to_primary_alias` (~L302)

```yaml
# ledger — top level (master table); siblings for same (entity, source) get primary: false
entity_domain: salesorder
table_type: hub
primary: true            # ← canonical FK-alias master for (salesorder, this source)
```
Load-bearing: top-level `primary: true` on exactly one table per `(entity_domain, source)`;
drives `fk_to_table` alias resolution + sibling demotion, **not** hub keying.

```fingerprint
{scope: file, table_type: hub, primary: true}
```

---

## Case index

| # | Case | Marker that distinguishes it | Home |
|---|---|---|---|
| 1 | Attribute | `kind: attribute`, `target: main` | CONTEXT "Derived artifact" |
| 2 | Hub single BK | `kind: entity_key` + `hk_x_h: x` | ADR-0013, compile B6/B12 |
| 3 | Composite BK 🔧 | ≥2 `entity_key` same target (order significant) | memory #582 |
| 4 | Role-qualified hub | `role:` + `hk_x_h: x:<role>` | ADR-0014 |
| 5 | x-direct rel-key (default) | `relationship_key` + `fk_to_table: null` + `hk_x_h: x` | ADR-0008 |
| 6 | Prejoin rel-key (exception) | `fk_to_table` + `fk_to_column` set | ADR-0008/0010 |
| 7 | rk target side | `metadata: rk` + `lnd_view: <target>` | ADR-0010 |
| 8 | uow line FK-only ✅ | `unit_of_work`, no `entity_key` | ADR-0003, CONTEXT "uow line" |
| 9 | Hard-rule transform | `transform` + `transform_origin` | ADR-0013 |
| 10 | Loader metadata override | `metadata_overrides` block | ADR-0009 |
| 11 | Source filter | top-level `source_filter` | compile B9 |
| 12 | Multi-table primary ⚠️ | top-level `primary: true` (FK-alias master) | compile B8 |

✅ verified against a real ledger · 🔧 shape verified against code (no committed tenant ledger) · ⚠️ built but parked/untested
