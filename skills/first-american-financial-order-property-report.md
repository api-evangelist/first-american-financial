---
name: Order a First American property or ownership report
description: Use the Digital Gateway order-then-retrieve pattern to request property, ownership, occupancy and FEMA flood data and fetch the resulting report.
api: openapi/first-american-financial-property-openapi.yml
operations: [reqPropertyBasic, reqPropertyExtended, reqPropertyOverview, reqListingHistory, reqLocalMarketTrend, reqNearbyActiveListings, reqPropertyHOA, reqPropertyFEMA, retrieveProperty, get, reqOwnership, reqAddress, reqDetails, reqForeclosure, reqPreforeclosure, reqScheduleREO, reqPrevForeclsoure, retreiveOwnership, Order, Report, ReportPDF]
---

# Order a First American property or ownership report

Every Digital Gateway data service is **order-then-retrieve**, never a single synchronous call.
You POST criteria, get a transaction identifier back, then GET the report with it.

## Authentication

Send `x-app-id` and `x-app-key` as **headers on every request**. They are declared as required
header parameters on each operation rather than in a `securityDefinitions` block, so a generated
client may not add them for you. A wrong or missing pair is HTTP 401 "Invalid App ID or App Key".
Sandbox and production credentials are different, and are issued per App in the Digital Gateway
portal after approval.

## Steps

1. **Order.** Pick the product and POST the criteria:
   - Property (`https://api.firstam.io/v1`) — `reqPropertyBasic` `POST /property/order/basic`
     (characteristics, assessor tax, assessee contact, transaction history, legal description,
     current owner; requires `StreetAddress1`, `City`, `State`, `ZipCode`), `reqPropertyExtended`,
     `reqPropertyOverview`, `reqListingHistory`, `reqLocalMarketTrend`,
     `reqNearbyActiveListings`, `reqPropertyHOA`, and `reqPropertyFEMA` `POST /fema/order` for
     flood zone.
   - Ownership (`https://api-w2.firstam.io/v1`) — `reqOwnership` `POST /ownership/order/full` or
     `/prime` (requires First Name, Last Name, SSN; set `IncludeAnalytics: true` for alerts and
     analytics), plus `reqAddress`, `reqDetails`, `reqPreforeclosure`, `reqForeclosure`,
     `reqPrevForeclsoure` and `reqScheduleREO`.
   - Occupancy (`https://solutions.dgw.firstam.io/v1`) — `Order` `POST /occupancy/order`.

2. **Keep the transaction identifier** from the order response. It is the only handle you have; no
   request-id header is returned and there is no way to list your outstanding orders.

3. **Retrieve.** `retrieveProperty` `GET /property/report`, `get` `GET /fema/report`,
   `retreiveOwnership` `GET /ownership/report/{variant}` (the variant must match the one you
   ordered), `Report` `GET /occupancy/report` or `ReportPDF` `GET /occupancy/report/pdf`.
   Poll until the report is available; no callback or webhook exists on these services.

## Rules

- **Note the base host per product.** They are not the same: Property, 4506-C, Identity, SCRA,
  Income Estimate, Reverse Phone and Liens & Judgments FCRA are on `api.firstam.io/v1`; Ownership,
  Bankruptcy, NMLS, Watchlist and Reverse Address are on `api-w2.firstam.io/v1`; Liens & Judgments
  Non-FCRA is on `api.dgw.firstam.io/v2`; Occupancy is on `solutions.dgw.firstam.io/v1`.
- **Most reads are POST**, deliberately — the docs say search criteria are moved out of the query
  string because they are sensitive. Do not "fix" this by converting to GET.
- **These orders are irreversible and billable.** There is no cancel, void or refund operation on
  any data service. Validate criteria before you POST.
- **No idempotency.** A retried order is a second order. On a timeout, retrieve before you re-order.
- 503 is the quota-exhaustion status on this platform, not 429, and no `Retry-After` is returned.
- Both JSON and XML are accepted and returned on most of these services.

## See also

- `conventions/first-american-financial-conventions.yml` — the order-then-retrieve contract
- `errors/first-american-financial-problem-types.yml`
