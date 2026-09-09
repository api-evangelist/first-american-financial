---
name: Open a First American title and escrow order
description: Find the right First American office, open a title and escrow order, and confirm the assigned officers and file number.
api: openapi/first-american-financial-title-settlement-openapi.yml
operations: [AAuthz_GetToken, FAoffices, EscrowOfficers_Get, TitleOfficers_Get, OrdersPost]
---

# Open a First American title and escrow order

Use this to open a title and escrow order on the First American Title & Settlement (Mortgage
Services) API. This is a real-money, real-property transaction: an order opens a file with a title
company. Do not submit one speculatively.

## Before you start

- You need an approved Digital Gateway App. Access is not self-service — sign up at
  `https://developer.firstam.io/signup`, an administrator reviews the account (1–2 business days),
  and production access additionally requires the First American relationship team and may require
  a contract or SOW.
- You need `client_id`, `client_secret`, `scope` and `grant_type`, plus the authentication URL.
  All five are issued privately — `LVIS.support@firstam.com` issues them. The token host in the
  specification is labelled "THIS ENDPOINT IS FOR DEMONSTRATION PURPOSES ONLY"; do not call it.

## Steps

1. **Get a token.** `AAuthz_GetToken` — `POST /api/token`, form-encoded with `client_id`,
   `client_secret`, `scope` and `grant_type` (all four required). The response carries
   `access_token`, `token_type` and `expires_in`. Send it as `Authorization: Bearer <token>` on
   every subsequent call; every other operation on this API declares `bearerAuth` and will return
   401 without it.

2. **Find the office.** `FAoffices` — `GET /offices`, filtered by city, state or ZIP. Take
   `officeId` from the office you want. This is the only genuinely read-only step in the flow.

3. **Pick the officers (optional).** `EscrowOfficers_Get` — `GET /employees/escrowOfficers` and
   `TitleOfficers_Get` — `GET /employees/titleOfficers`, both keyed on the office id. Take the
   officer `Code` values if you want to request specific officers.

4. **Choose an `externalTrackingId` and keep it.** It is client-supplied and **must be unique per
   transaction**, alongside the property address. It is the only correlation key you get: every
   document upload, message, cancellation and webhook event for this order is addressed by it.
   Generate it before the write, persist it before the write, and never reuse one.

5. **Open the order.** `OrdersPost` — `POST /orders` with an `OrderRequest`: `service`,
   `parties` (`IndividualParty` and/or `LegalEntityParty` with roles, addresses and contact
   points), `propertyAddress`, `officeId`, `escrowOfficerCode`, `titleOfficerCode`,
   `transactionType`, `loan`, `transactionDetails` and any `documents`. The response is a
   `ServiceResponse` carrying `ID`, `FileNumber` and the `Services` array with the assigned
   `Officer` and `Assistant`. Record the `FileNumber` — it is First American's handle on the file.

## Rules that apply to every step

- **There is no idempotency contract.** No `Idempotency-Key` header exists on this API. The
  uniqueness constraint on `externalTrackingId` means a duplicate submission is *rejected*, not
  deduplicated — so on a timeout do **not** blindly retry. Re-issue the same request with the same
  `externalTrackingId`; a rejection tells you the first attempt landed.
- **Reversal is `Orders_Cancel` and there is no published window.** `POST /orders/{externalTrackingId}/cancel`
  exists, but the specification says only "Cancel an Order previously created". Nothing states
  whether it works after funding approval or recording. Confirm with the escrow officer rather than
  assuming.
- **Status codes:** 200 success, 400 invalid request (the body names the offending field), 401
  invalid credentials or expired token, 500 system error, **503 usage limit exceeded** — 503 is
  this platform's quota signal, not 429, and no `Retry-After` or `RateLimit-*` header is returned.
- `AppSource` is required if you are using source-level credentials.

## See also

- `conventions/first-american-financial-conventions.yml`
- `errors/first-american-financial-problem-types.yml`
- `authentication/first-american-financial-authentication.yml`
