---
name: Build a campaign delivery and performance report from Basis
description: >-
  Walk a Basis campaign down to its line items and pull delivery and performance
  statistics for a date range, joining impressions, clicks, spend and eCPM/eCPC/eCPA
  back to the media plan.
api: openapi/basis-analytics-api-openapi.yml
operations:
  - GET /v1/campaigns
  - GET /v1/campaigns/{id}
  - GET /v1/campaigns/{campaign_id}/line_items
  - GET /v1/campaigns/{campaign_id}/addons
  - GET /v1/stats/{scope}
  - GET /v1/kpis
generated: '2026-08-13'
method: generated
source: >-
  openapi/basis-analytics-api-openapi.yml (paths, parameters and response schemas)
  and the "Statistics" / "Resource Relationships" sections of the Basis Platform
  API description at https://api.basis.net/swagger.json. The specification
  declares no operationIds, so operations are identified by method + path.
---

# Build a campaign delivery report

Authenticate first — see `basis-authenticate-and-read.md`. Every call below needs
`Authorization: Bearer <access_token>`.

## Step 1 — find the campaign

```
GET /v1/campaigns?status=live&query=<name fragment>
```

`status` accepts `live`, `approved` or `completed`. You can also narrow with
`client_id` or `brand_id`. The response gives you `id`, `name`, `client_id`,
`brand_id`, `approved_budget`, `goal`, `objectives`, `kpi_name`, `start_date` and
`end_date`.

Page with the cursor, never by guessing an offset:

```
GET /v1/campaigns?cursor=<metadata.cursor from the previous page>
```

Stop when `metadata.cursor` comes back null. `metadata.total` tells you how many
campaigns match so you can size the walk up front.

## Step 2 — pull the media plan

```
GET /v1/campaigns/{campaign_id}/line_items
GET /v1/campaigns/{campaign_id}/addons
```

Line items are the unit delivery is measured against. Each carries the contracted
side of the plan — `media_rate`, `rate_type`, `media_spend_contracted`,
`media_contracted_units`, `total_spend_contracted`, `margin_pct_contracted`,
`ad_serving_*` — plus `property_id`, `vendor_id` and `kpi_id`.

Add-ons are fees on the campaign that are **not** media. Include them when you
report total campaign cost or your spend will be understated.

## Step 3 — pull delivery

```
GET /v1/stats/{scope}?campaign_id=<uuid>&start_date=YYYY-MM-DD&end_date=YYYY-MM-DD
```

Pick `{scope}` for the shape of the report you are building:

| scope | Row grain | Use it for |
|---|---|---|
| `line_item` | one row per line item, whole period | a flight-to-date summary |
| `daily_by_line_item` | one row per line item per day | pacing and trend charts |
| `daily` | one row per day | a campaign-level burn-down |
| `daily_by_conversion` | one row per day per conversion | conversion attribution |

You can also filter by `line_item_id` or `line_item_lineage_id`.

Rows carry delivery metrics (`delivered_impressions`, `delivered_clicks`,
`delivered_units`, `delivered_video_starts`, `delivered_video_completions`,
`delivered_interactions`, `delivered_viewable_impressions`,
`delivered_measurable_impressions`, `auctions_won`), spend
(`delivered_inventory_spend`, `delivered_data_spend`, `ad_serving_spend`,
`delivery_spend_pct`) and computed performance (`ecpm`, `ecpc`, `ecpv`, `ecpcv`,
`ecpa`, `ecpvi`, `click_through_rate`).

**Check `data_through_date` before you report.** It tells you how far the
delivery data is actually complete; reporting up to `end_date` when
`data_through_date` is earlier will show a false drop at the tail.

## Step 4 — join

Join stats rows back to the plan on `line_item_id`. Resolve the KPI with
`GET /v1/kpis/{id}` using the line item's `kpi_id` to get `name` and `goal_type`,
then compare it against the line item's `kpi_goal`.

Delivered-versus-contracted is the report: `delivered_impressions` against
`media_contracted_units`, and `delivered_inventory_spend + delivered_data_spend +
ad_serving_spend` against `total_spend_contracted`.

## Rules that apply throughout

- **Pagination is cursor-only.** There is no page-size parameter; you cannot ask
  for bigger pages.
- **Identifier formats differ by resource.** Campaigns and line items are UUID v4;
  KPIs are numeric strings. A mismatch returns `400` echoing the expected regex.
- **Budget your calls.** 75,000 requests per hour per API user, across all
  endpoints. A daily-by-line-item walk over a large agency is the request pattern
  most likely to hit it. Prefer one `daily_by_line_item` call filtered by
  `campaign_id` over per-line-item calls in a loop.
- **Third-party integrators report historical cutoffs** (~90 days for
  geographic dimensions, ~180 days for campaign/audience dimensions). Basis does
  not confirm these; treat an empty result for an old range as expected rather
  than as an error. See `rate-limits/basis-rate-limits.yml`.

## Related

- `data-model/basis-data-model.yml`
- `conventions/basis-conventions.yml`
- `errors/basis-problem-types.yml`
