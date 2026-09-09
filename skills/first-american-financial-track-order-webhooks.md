---
name: Subscribe to First American order milestone webhooks
description: Register, filter, pause and delete webhook subscriptions so an agent is notified of the 21 published order milestone events instead of polling.
api: openapi/first-american-financial-title-settlement-openapi.yml
operations: [AAuthz_GetToken, Webhooks_Post, Webhooks_Post_Multiple, Webhooks_GetAll, Webhooks_Get, Webhooks_Put, Webhooks_MultipulePUT, Webhooks_Del, Webhooks_Delete_Multiple, Webhooks_DelAll]
---

# Subscribe to First American order milestone webhooks

Title and escrow files run for weeks. First American publishes 21 milestone events on the Title &
Settlement API, which is the only way to follow an order without polling — there is no order-status
read operation.

## Before you start

- **Domain whitelisting is required before any event will be delivered.** Contact
  `LVIS.support@firstam.com`. Registering a subscription against an unwhitelisted domain will not
  start delivery.
- Get a bearer token first (`AAuthz_GetToken`); every webhook operation declares `bearerAuth`.

## Steps

1. **Register.** `Webhooks_Post` — `POST /webhooks` with a `Webhook`: `Uri` (your endpoint),
   `Secret` ("secret to be used when generating a message hash"), `Description`, `AppSource`
   (subscription grouping), `Filter` (an `EventType` — scope the subscription rather than taking
   all 21), `Headers` (arbitrary key/value pairs First American will send back; `x-api-key` is the
   documented example) and `Properties.retryCount` (an integer 1–5 — the redelivery ceiling).
   Use `Webhooks_Post_Multiple` (`POST /webhooks/registerWebhooks`) to register several at once.

2. **Verify.** `Webhooks_GetAll` — `GET /webhooks` lists them; `Webhooks_Get` — `GET /webhooks/{id}`
   reads one.

3. **Handle the delivery.** The body is an `EventNotification`: `Id`, `RetryCount` and a
   `Notifications` array of `Event` objects. Each `Event` carries `Action` (one of the 21 event
   types), `ExternalTrackingId` (the id **you** chose when opening the order — join on this) and a
   typed `Message` payload: `DocumentPayload` for document deliveries,
   `FundsDisbursedMessagePayload` for `FundsDisbursed`, `CurativeClearedMessagePayload` for
   `CurativeCleared`, `MessagePayload` for `MessageAdded`, `ServiceResponse` for order status.
   **Respond HTTP 200.** The specification states the notification is considered successfully
   published only once your system returns 200; anything else burns a retry.

4. **Pause instead of deleting** when you need to stop temporarily — the subscription carries
   `IsPaused`. Update with `Webhooks_Put` (`PUT /webhooks`) or `Webhooks_MultipulePUT`
   (`PUT /webhooks/webhookRequests`).

5. **Delete.** `Webhooks_Del` — `DELETE /webhooks/{id}` for one, `Webhooks_Delete_Multiple`
   (`POST /webhooks/webhooksList`) for a list, `Webhooks_DelAll` — `DELETE /webhooks` for **all of
   them**. Never call `Webhooks_DelAll` on a shared App: it takes down every subscription on the
   account, and there is no undo.

## The 21 events

Order: `OrderCreated`, `OrderRejected`, `OrderCancelled`, `OrderClosed`.
Title: `CurativeCleared`, `Curative Pending`.
Documents: `TitleProductDelivered`, `FinalPolicyDelivered`, `DocumentDelivery`, `CDDraft`, `CDBalanced`.
Messaging: `MessageAdded`.
Funds: `FundingApproved`, `FundsDisbursed`, `DisburseHold`.
Signing: `AppointmentScheduled`, `SigningOrdered`, `SigningBorrowerNotified`, `SigningCompleted`, `SigningFailed`.
Recording: `RecordingCompleted`.

Documents arrive as **Base64-encoded PDF** inside the event, not as a URL.

## Caution

The `Secret` field exists for message hashing, but the hash algorithm and the header carrying the
hash are **not documented in the contract**. Do not assume a scheme — ask First American what to
verify before you trust an inbound event, and in the meantime treat the whitelisted-domain plus
your own `Headers` value as the only authentication you actually have.

## See also

- `asyncapi/first-american-financial-title-settlement-webhooks.yml`
- `data-model/first-american-financial-data-model.yml`
