---
name: backbone-lineage
description: Draw a tenant's dbt lineage page from its runtime manifest with the repo's reviewed renderer, publish it to the tenant's one recorded artifact URL, and record what was published. Skips the publish when the page is unchanged. Runs no DML and no dbt; /backbone-run and /backbone-publish call it at their end. On the hosted Backbone the service renders the page and records the publication.
argument-hint: <tenant_id>
---

# /backbone-lineage

Run the `backbone-lineage` skill against tenant `$0`.

The skill resolves the estate root first (its step 0, `GET /api/estate`), renders the page
locally with the reviewed renderer, `render_lineage_page.py`, into this session's scratchpad,
publishes that file through the Artifact tool to the URL recorded in
`{estate root}/{tid}/lineage/publication.json` (a new artifact when there is none), and
records it with `PUT /api/config/{tid}/lineage/publication`. When the recorded page is
already this page it publishes nothing and says so. On the hosted Backbone it gets a one-time
download link from `backbone_get_lineage_page`, downloads the page with `curl` and checks
its sha256, and records it with `backbone_put_lineage_publication` instead of the script
and the REST call.

`/backbone-run` and `/backbone-publish` run it as `/backbone-lineage <tid> from-run` /
`from-publish` at their end; do not type the marker yourself.

**Usage:** `/backbone-lineage <tenant_id>`
