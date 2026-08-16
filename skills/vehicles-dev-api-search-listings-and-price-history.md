---
name: Search dealer listings and read price history
description: >-
  Page through the crawled US dealer-listings store with the only paginated endpoint on the API, then
  read per-VIN price and mileage observations — handling the Scale-only plan gate correctly.
api: openapi/vehicles-dev-api-vehicles-api-openapi.yml
operations:
  - getVehicleListings
  - getVehicleHistory
  - getVehiclePhotos
generated: '2026-08-16'
method: generated
source: https://vehicles.dev/docs + openapi/_original/vehicles-dev-api-openapi.json
---

# Search dealer listings and read price history

## Steps

1. **Search** — `getVehicleListings`, `GET /v1/vehicles/listings`.
   All parameters are optional query parameters:
   `make`, `model`, `year_min`, `year_max`, `price_min`, `price_max`, `mileage_max`, `state`,
   `condition`, `seller_type`, `source`, `active`, `sold`, `valid_vin`, `min_quality`, `sort`,
   `order`, `limit`, `offset`.
   Response envelope: `count`, `limit`, `offset`, `total`, `results`, `source`.

2. **Paginate** — this is the **only** paginated endpoint on the API. Use `limit` (1–500, default 50)
   and `offset` against `total`. There are no cursors and no `Link` headers. Every other endpoint is a
   single-object lookup.

3. **Read price history** — `getVehicleHistory`, `GET /v1/vehicles/history/{vin}`.
   Returns `observations` with `observed_at`, plus `firstSeen` and `lastSeen`.
   **Scale plan only.** On Starter and Pro this returns `403 plan_upgrade_required`, and credits
   cannot buy it. Detect the plan gate once and disable the feature rather than retrying.

4. **Attach an image** — `getVehiclePhotos`, `GET /v1/vehicles/photos/{vin}`.
   Returns one hero image and a link to the source listing — not a gallery. The photo URL points at a
   CDN owned by the source marketplace; Vehicles.dev links to it and does not license or sublicense
   the imagery.

## Rules you must follow

- **Do not treat `is_active` as a live signal.** It means the row was still live at the last crawl
  pass that covered it, and no crawl timestamp is exposed. Likewise `days_on_market` counts from the
  provider's first observation, not the dealer's listing date.
- **No maximum staleness is published** for the store-backed endpoints. If your use case needs a hard
  freshness bound or a per-row crawl timestamp, neither is available through the API today.
- **Unknown query parameters are an error**, not a silently dropped extra — the query schema is
  closed. Query strings are coerced to their declared integer and boolean types.
- Nested `results` elements keep upstream snake_case keys while the envelope is camelCase. That
  asymmetry is deliberate and stable.
- Search and Photos are two of the three endpoints covered by the Starter plan's 1,000 included
  monthly calls (the third is VIN Decode). Listing Price History is never included.
