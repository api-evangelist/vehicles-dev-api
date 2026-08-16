---
name: Identify and value a VIN
description: >-
  Turn a 17-character VIN into a canonical vehicle identity, then attach an asking-price estimate and
  a depreciation curve — choosing between three cheap calls and one expensive composite call.
api: openapi/vehicles-dev-api-vehicles-api-openapi.yml
operations:
  - getVehicleVinDecode
  - getVehicleMarketValue
  - getVehicleDepreciation
  - getVehicleReport
generated: '2026-08-16'
method: generated
source: https://vehicles.dev/docs + openapi/_original/vehicles-dev-api-openapi.json
---

# Identify and value a VIN

## Before you start

- Base URL is `https://api.vehicles.dev`. Send `Authorization: Bearer $VEHICLES_API_KEY` on every
  request. The key is prefixed `vdev_`.
- Call from a backend. The API serves no CORS headers, and any request carrying an `Origin` header is
  refused with `404 route_not_found`.
- Every call is billed only on `2xx`. A 4xx or 5xx costs nothing, so do not build defensive
  pre-flight logic.

## Steps

1. **Decode the VIN** — `getVehicleVinDecode`, `GET /v1/vehicles/vin/{vin}`.
   The only path parameter is `vin`; it is upper-cased server-side. Read `vehicle.year`,
   `vehicle.make`, `vehicle.model` and `vehicle.trim` from the response. Every key inside `vehicle`
   is optional — null fields are omitted, not emitted as `null`.
   Check `origin`: `"store"` means it came from the crawled dataset, `"vpic"` means it was decoded
   live against NHTSA.

2. **Decide: three calls or one.**
   - Composite — `getVehicleReport`, `GET /v1/vehicles/report/{vin}` returns identity, an asking-price
     estimate and a depreciation curve in one call. It is the most expensive operation on the API
     ($1.99 on Starter/Pro, $0.99 on Scale) and it is labelled legacy. Read its `coverage` array to
     see which sections actually resolved.
   - Separate — decode + `getVehicleMarketValue` + `getVehicleDepreciation` costs roughly $0.034 on
     Starter. Prefer this unless you need one atomic call.

3. **Estimate value** — `getVehicleMarketValue`, `GET /v1/vehicles/market-value`.
   Required query parameters: `make`, `model`, `year`. Optional: `trim`, `miles`, `state`,
   `condition`, `color`, `drivetrain`, `fuel`, `transmission`, `body_style`, `base_msrp`.
   `model` is matched case-sensitively against the provider's canonical naming — pass `F-150`, not
   `f150`. `state` is the uppercase two-letter code.
   Read `estimateUsd` (whole US dollars), `medianApePct` for model error, and `inputs` to confirm
   which of your arguments were actually scored.

4. **Attach the curve** — `getVehicleDepreciation`, `GET /v1/vehicles/depreciation`.
   Model-level only: the curve is per make and model, never per trim or VIN.

## Rules you must follow

- **Report the estimate honestly.** `estimateUsd` is an *asking* price derived from live dealer
  listings, not a realized transaction price. Say so whenever you surface the number.
- **Unknown query parameters are rejected**, not ignored. Every query schema is closed, so a typo
  produces `400 request_validation_failed` with an `invalid_params` array naming the JSON pointer.
- **Branch on `code`**, never on `detail` or on the `type` URI. See
  `errors/vehicles-dev-api-problem-types.yml`.
- **Retry on `retryable: true` only** — every `503` and the `429`. Use exponential backoff with
  jitter; honour the `retry-after` header on a `429`. All four operations here are GETs, so a retry
  is always safe.
- **On `402 insufficient_credits`**, stop and report it. There is no balance header to check first,
  and no plan converts this into an overage bill.
- **Log `x-request-id`** from every response. It is the only handle support can trace, because the
  provider redacts `Authorization` from its logs.
