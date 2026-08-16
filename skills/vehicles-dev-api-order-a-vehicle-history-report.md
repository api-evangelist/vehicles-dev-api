---
name: Order a vehicle history report
description: >-
  Run the asynchronous, account-owned vehicle-history workflow end to end — idempotent create, status
  polling, durable-submitting retry, and result read — without double-charging or double-ordering.
api: openapi/vehicles-dev-api-reports-api-openapi.yml
operations:
  - createVehicleHistoryReport
  - getVehicleHistoryReportStatus
  - retryVehicleHistoryReportSubmission
  - getVehicleHistoryReportResult
generated: '2026-08-16'
method: generated
source: https://vehicles.dev/docs#vehicle-history-reports
---

# Order a vehicle history report

This is the **only write flow** on the Vehicles.dev data plane and the most expensive one ($1.99 on
Starter/Pro, $0.99 on Scale). It is deliberately *not* exposed as an MCP tool. Do not run it
speculatively.

## Before you start

- `Authorization: Bearer $VEHICLES_API_KEY` on every request.
- The machine route requires the `reports:order` permission to create or explicitly retry, and
  `reports:read` to poll and read the result.
- A report ordered by one account is not visible to another.

## Steps

1. **Create** — `createVehicleHistoryReport`, `POST /v1/vehicles/history-reports`.
   Required header: `Idempotency-Key`, a UUID. Body: `{"vin": "<17-character VIN>"}` — the VIN is
   validated at exactly 17 characters.
   Success is `202`, with a stable `id`, a `Location` header and a `Retry-After` header. The body
   carries `id`, `vin`, `status`, `hasResult`, `retryAfterSeconds`, `replayed`, `createdAt`,
   `updatedAt`.
   **Persist the `Idempotency-Key` alongside the VIN before you send the request.** Reusing the key
   with the same VIN returns the existing report; reusing it with a different VIN is
   `409 idempotency_conflict`.

2. **Poll** — `getVehicleHistoryReportStatus`, `GET /v1/vehicles/history-reports/{id}`.
   Poll at the server-supplied `Retry-After` cadence, not a cadence you choose. Valid states are
   `submitting`, `queued`, `processing`, `action_required`, `completed`.
   Polling is read-only and never resubmits.

3. **Retry only a durable `submitting`** — `retryVehicleHistoryReportSubmission`,
   `POST /v1/vehicles/history-reports/{id}/retry`.
   Use this **only** when a create response was uncertain and the local report is still `submitting`.
   The server resubmits with the original stored provider idempotency key. Never call it as a general
   retry for `queued` or `processing`.

4. **Read the result** — `getVehicleHistoryReportResult`,
   `GET /v1/vehicles/history-reports/{id}/result`.
   Fetch only after `status == "completed"` **and** `hasResult == true`. Reading early returns the
   retryable `409 report_not_ready`.
   The response is the canonical JSON the report provider returned, preserved verbatim.

## Rules you must follow

- **Never present missing coverage as a clean record.** When a title, accident, odometer or ownership
  section is absent from the result, say it is unavailable. Reporting "no accidents found" when the
  section did not resolve is a factual error with real consequences for a buyer.
- **Billing settles only when a completed result is stored.** Invalid or unsupported VINs, failed
  generations and `action_required` outcomes do not settle a charge.
- `429 report_rate_limited` means the upstream provider throttled the submission — retryable, back off.
- Do not confuse this with the two legacy GET routes: `getVehicleReport`
  (`GET /v1/vehicles/report/{vin}`) is the composite identity/value/depreciation report, and
  `getVehicleHistory` (`GET /v1/vehicles/history/{vin}`) is marketplace crawl price observations.
  Neither returns the provider-backed history result.
- The dashboard equivalents (`createControlVehicleHistoryReport`,
  `getControlVehicleHistoryReportStatus`, `getControlVehicleHistoryReportResult`,
  `retryControlVehicleHistoryReportSubmission`) require a WorkOS session token and reject a `vdev_`
  key with `401 invalid_credential`. Use the `/v1/vehicles/` routes from a service.
