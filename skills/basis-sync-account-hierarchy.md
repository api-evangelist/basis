---
name: Sync the Basis account hierarchy and reference taxonomy
description: >-
  Mirror an agency's Basis account structure — agency, clients, brands, campaigns
  — together with the vendor, property, vertical, KPI, creative, conversion and
  delivery-source reference data every other Basis payload refers to by id.
api: openapi/basis-analytics-api-openapi.yml
operations:
  - GET /v1/agency
  - GET /v1/clients
  - GET /v1/brands
  - GET /v1/campaigns
  - GET /v1/vendors
  - GET /v1/properties
  - GET /v1/verticals
  - GET /v1/kpis
  - GET /v1/creatives
  - GET /v1/conversions
  - GET /v1/delivery_sources
generated: '2026-08-13'
method: generated
source: >-
  openapi/basis-analytics-api-openapi.yml (paths, parameters, response schemas)
  and data-model/basis-data-model.yml. The specification declares no
  operationIds, so operations are identified by method + path.
---

# Sync the Basis account hierarchy

Basis payloads are full of bare id fields — `client_id`, `brand_id`, `vendor_id`,
`property_id`, `kpi_id`, `delivery_source_id`, `conversion_id`, `creative_id` —
and the API has no field-expansion parameter. There is no way to inline a related
object. You have to hold the reference tables locally, which is what this skill
builds.

Authenticate first — see `basis-authenticate-and-read.md`.

## Step 1 — anchor on the agency

```
GET /v1/agency
```

Returns `id`, `name`, `dsp_advertiser_id`, `created_at`. Everything a credential
can see hangs off this one organization; store `id` as the tenant key for the
whole sync.

## Step 2 — walk the hierarchy

```
GET /v1/clients
GET /v1/brands                      # optionally ?client_id=<uuid>
GET /v1/campaigns                   # optionally ?client_id= or ?brand_id=
```

The shape is agency → clients → brands, with campaigns carrying both `client_id`
and `brand_id`. Clients also carry billing and contact detail
(`billing_name`, `contact_first_name`, `contact_last_name`, `contact_email`,
`contact_phone`, `notes`) — treat that as personal data and store it accordingly.

Every list endpoint is cursor-paginated. The loop is always the same:

1. call without `cursor`
2. append `data`
3. if `metadata.cursor` is non-null, call again with it
4. stop when it is null

`metadata.total` on the first page tells you how many pages to expect.

## Step 3 — pull the reference tables

These are small, slow-changing and referenced by id from everywhere else. Pull
them once per sync, before the fact tables:

```
GET /v1/verticals          # numeric ids
GET /v1/kpis               # numeric ids, plus goal_type
GET /v1/vendors            # who sells the inventory
GET /v1/properties         # the inventory itself, with url and verticals
GET /v1/creatives          # 40-char alphanumeric ids
GET /v1/conversions        # note: primary key is conversion_id, NOT id
GET /v1/delivery_sources   # what the stats are computed from
```

**Watch the identifier formats.** They are not uniform: UUID v4 for clients,
brands, campaigns, line items, groups, tactics, conversions and delivery sources;
numeric strings for verticals and KPIs; a 40-character alphanumeric for creatives;
unconstrained strings for agency and properties. Storing them all in a UUID column
will fail on three of these tables.

**Watch `conversions`.** It is the one resource whose primary key is
`conversion_id` rather than `id`. A generic mapper keyed on `id` will silently
produce null keys.

## Step 4 — the activation side

```
GET /v1/groups
GET /v1/tactics            # each carries group_id
```

Groups are budget-and-pacing containers; tactics are the targeting/bidding
strategies inside them. Be aware of a real gap in the public contract: **groups
and tactics carry no campaign or line-item reference**, so the activation side
cannot be joined to the planning side through this API alone. Sync them as their
own subtree and do not invent a join.

## Step 5 — refresh policy

- Reference tables (verticals, KPIs, vendors, properties, creatives, conversions,
  delivery sources): daily is plenty.
- Hierarchy (clients, brands, campaigns): daily, or on demand before a reporting
  run.
- There is no `updated_at` field and no delta or webhook mechanism anywhere in the
  API, so every sync is a full re-read. Diff locally.

## Budget

75,000 requests per hour per API user across all endpoints. A full hierarchy sync
for a large agency is dominated by campaign paging — filter by `client_id` and run
clients in sequence rather than fanning out, so a partial failure resumes cleanly.

## Related

- `data-model/basis-data-model.yml`
- `conventions/basis-conventions.yml`
- `json-schema/`
