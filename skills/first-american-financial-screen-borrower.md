---
name: Screen a borrower with First American verification and compliance APIs
description: Run identity, watchlist, SCRA, NMLS, bankruptcy, liens and 4506-C checks through the Digital Gateway, respecting the FCRA / non-FCRA split.
api: openapi/first-american-financial-watchlist-openapi.yml
operations: [post, get, reqssnsearch, reqSsnVerification, reqSsnReport, reqSsnPlus, retrieveIdentity, postOrder, postConsentForm, getTranscript, getTranscriptPDF, getStatus]
---

# Screen a borrower with First American verification and compliance APIs

These operations take Social Security numbers, names and dates of birth and return consumer
information. **Read the permissible-purpose note before writing any code.**

## Permissible purpose is expressed in the contract

First American splits Liens & Judgments into two separate APIs on two separate hosts:

- `openapi/first-american-financial-liens-judgments-fcra-openapi.yml` — `POST /lnj-fcra/order` on
  `api.firstam.io/v1`, the **Fair Credit Reporting Act** variant.
- `openapi/first-american-financial-liens-judgments-non-fcra-openapi.yml` — `POST /lnj/order` on
  `api.dgw.firstam.io/v2`, the **non-FCRA** variant.

Choosing between them is a legal decision about your permissible purpose, not a technical one. An
agent must not pick one because the other returned an error. If you do not know which applies, stop
and escalate to a human.

## Authentication

`x-app-id` and `x-app-key` headers on every request, as with every Digital Gateway data service.

## The checks

| Check | Order | Retrieve | Host |
|---|---|---|---|
| Identity / SSN | `reqssnsearch`, `reqSsnVerification`, `reqSsnReport`, `reqSsnPlus` | `retrieveIdentity` `GET /identity/report` | `api.firstam.io/v1` |
| Watchlist | `post` `POST /watchlist/order/ofac` and `/freddiemac`, `/fhfascp`, `/hudepls`, `/hudldp`, `/nfpd`, `/ineligiblelist` | `get` `GET /watchlist/report` | `api-w2.firstam.io/v1` |
| SCRA | `post` `POST /scra/order` (First/Last name, SSN, DOB, RequestorID) | `get` `GET /scra/report` | `api.firstam.io/v1` |
| Bankruptcy | `post` `POST /bankruptcy/order/basic`, `/extended`, `/premium` | `get` `GET /bankruptcy/report` | `api-w2.firstam.io/v1` |
| NMLS | `get`/`post` `/nmls/lookup`, `/nmls/lookup/company`, `/nmls/lookup/individual`, `/nmls/search*` | synchronous | `api-w2.firstam.io/v1` |
| Liens & Judgments | `post` (see the FCRA split above) | `get` | see above |
| 4506-C tax transcript | `postOrder` `POST /4506t/order`, then `postConsentForm` `POST /4506t/consentForm` | `getStatus`, then `getTranscript` / `getTranscriptPDF` | `api.firstam.io/v1` |

## Rules

- **NMLS is the only synchronous check** — `/nmls/lookup` supports GET with the NMLS ID in the
  query string, and `&includeanalytics=true` adds alert analytics. Everything else is
  order-then-retrieve.
- **4506-C needs a consent form.** `postConsentForm` is a separate operation. Do not order a tax
  transcript for a subject who has not consented.
- **Watchlist searches take an `EntityType`** — submit `Company` and put the company name in
  `LastOrCompanyName`, or `Individual`. Set `IncludeAnalytics: true` for alerts and analytics.
- **Bankruptcy search modes are mutually exclusive.** DefendantName searches need a Defendant plus
  a CourtID or CourtState; CaseNumber searches need a CaseNumber plus a CourtState; Attorney
  searches need an Attorney name plus a CourtState.
- **Every order here is irreversible and billable.** There is no cancel on any data service.
- **Never log the request bodies.** They carry SSNs and dates of birth.
- 400 names the offending field in prose ("Bad Request – SSN is a required field"); do not retry an
  unchanged request. 503 means the usage quota is exhausted.

## See also

- `conformance/first-american-financial-conformance.yml` — the FCRA / OFAC / SCRA / NMLS regime map
- `authentication/first-american-financial-authentication.yml`
