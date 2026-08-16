---
name: Check safety recalls and running costs
description: >-
  Answer "is this car safe and what will it cost to run" from the three live federal-source endpoints
  — NHTSA recalls, NHTSA vPIC specifications and EPA ownership costs — with their coverage caveats
  stated correctly.
api: openapi/vehicles-dev-api-vehicles-api-openapi.yml
operations:
  - getVehicleRecalls
  - getVehicleSpecifications
  - getVehicleOwnershipCosts
generated: '2026-08-16'
method: generated
source: https://vehicles.dev/docs + openapi/_original/vehicles-dev-api-openapi.json
---

# Check safety recalls and running costs

All three operations call a federal source live on every request and are uncached. They are exactly
as current — and as available — as NHTSA and the EPA are, so a `503` here usually means an upstream
agency outage, not a Vehicles.dev failure.

## Steps

1. **Recalls** — `getVehicleRecalls`, `GET /v1/vehicles/recalls/{vin}`.
   Returns every NHTSA campaign for the vehicle's year/make/model. Upstream is the NHTSA recalls API.

2. **Specifications** — `getVehicleSpecifications`, `GET /v1/vehicles/specifications/{vin}`.
   Returns the raw NHTSA vPIC spec sheet: seats, horsepower, GVWR, plant country and the rest of the
   factory build record.

3. **Ownership costs** — `getVehicleOwnershipCosts`, `GET /v1/vehicles/ownership-costs`.
   Query by `year`, `make`, `model`. Returns EPA annual and five-year fuel cost
   (`annualFuelCostUsd`, whole US dollars), combined MPG and CO2. Upstream is EPA fueleconomy.gov.

## Rules you must follow

- **A recall match is not proof.** Campaigns are matched at the year/make/model level, so a listed
  campaign does not by itself establish that this specific VIN is affected. Always say that, and
  point the user at the NHTSA VIN lookup or a dealer for VIN-level confirmation.
- **Ownership costs are fuel only.** They exclude insurance, maintenance, repairs, tyres, taxes,
  registration and depreciation. Do not present the figure as total cost of ownership.
- **`report_date` on a recall is passed through verbatim** in NHTSA's inconsistent formats. It is the
  one field on the API that is not ISO-8601 — parse defensively, and do not assume a format.
- Every other nested field may be absent: null keys are omitted rather than emitted, so treat every
  key inside `specifications` and `recalls` as optional.
- Set your client timeout above the endpoint's upstream budget (5, 10 or 15 seconds, listed per
  endpoint) so you receive the API's `503` problem document rather than timing out blind.
- `getVehicleRecalls` is the second most expensive per-call data endpoint after Ownership Costs
  ($0.01 / $0.045 on Starter). Neither is covered by the Starter plan's included calls.
