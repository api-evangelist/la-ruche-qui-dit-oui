---
name: la-ruche-qui-dit-oui-checkout-and-pay-basket
description: >-
  Read a member's basket and orders, confirm a basket into a payment hand-off,
  and retry a failed payment on the La Ruche qui dit Oui! / Food Assembly API.
  Money-moving — requires human confirmation and must never be retried blindly.
api: la-ruche-qui-dit-oui:food-assembly-api
generated: '2026-07-19'
method: generated
source: openapi/la-ruche-qui-dit-oui-api-openapi.yml
operations:
  - listOrders
  - confirmBasket
  - repayOrder
---

# Check out and pay a basket

## Safety rule — read this first

`confirmBasket` and `repayOrder` move money, and **the API publishes no
idempotency mechanism**. There is no idempotency key header, no request-replay
semantics, and no documented deduplication. A retried call has no protection
against generating a duplicate payment hand-off.

Therefore:

- **Never retry either operation automatically.** Not on timeout, not on 5xx,
  not on a dropped connection.
- **Always get explicit human confirmation** before the first call.
- If a call's outcome is unknown, do not re-issue it. Re-read state with
  `listOrders` and inspect the order's `state` before doing anything else.

## Understanding order state

Orders carry a numeric `state` — this is the authoritative lifecycle value. They
also carry a coarser string `status` (`basket` or `order`) that overlaps with it;
prefer `state`.

| `state` | Meaning |
|---|---|
| 1 | Cart — ongoing order, member is still shopping |
| 2 | Cart locked — member clicked Pay, payment form generated on the PSP side |
| 3 | Pending — payment in progress, awaiting PSP feedback |
| 4 | Confirmed — payment successful |
| 5 | Shipped — distribution over, member received the order |
| 7 | Canceled — payment denied or canceled |

States 6 and 8 are marked *n/a* in the provider's documentation.

## Steps

1. **Read current state** with `listOrders` — `GET /orders/` with
   `Authorization: Bearer <token>`. Returns `{count, orders: [...]}`. There is
   no pagination, so you receive every order; for a long-tenured member this
   response can be large.

   Each order carries `id`, `createdAt`, `status`, `state`, `totalPrice`
   (integer minor units + currency), `countItems`, `items`, `distributionId`
   and an inline `distribution` object. The shape of `items[]` is elided in the
   provider's own documentation — do not assume its fields.

2. **Decide which operation applies.**
   - `state: 1` (Cart) → confirm it with `confirmBasket`.
   - `state: 7` (Canceled) or a previously failed payment → retry with
     `repayOrder`.
   - `state: 2` or `3` → a payment is already in flight. **Stop.** Do not
     confirm again; wait and re-read.
   - `state: 4` or `5` → already paid. Nothing to do.

3. **Confirm the basket** with `confirmBasket` —
   `POST /distributions/{id}/basket/confirm/` where `{id}` is the
   **distribution** id. Note the plural `distributions` segment here; the
   product-listing route uses the singular form.

   Body requires three return URLs: `urlAccept`, `urlDecline`, `urlCancel`.

4. **Or retry payment** with `repayOrder` — `POST /orders/{id}/payments/` where
   `{id}` is the **order** id. Same three-URL body. The response is the full
   order object with a fresh payment hand-off.

5. **Hand off to the payment provider.** Neither operation charges the card. The
   response carries `paymentForm` (pre-rendered HTML) and `paymentRequest`
   (`url` plus a signed `data` payload) to POST to the payment service provider
   — the documentation's examples show Ogone. An agent cannot complete this
   step; surface the form or URL to the human and stop.

6. **Confirm the outcome** by re-reading `listOrders` and checking that `state`
   advanced to `4`.

## Errors

`400` on either payment operation returns the order envelope:
`{"error", "details", "orderUuid"}`. The only documented code is
`MissingUserInformation` — "Order cannot be placed because some user information
are missing". The documentation does not say which fields are missing; the
member must complete their profile in the web application. `orderUuid` is the
only correlation identifier the API surfaces, so capture it for support.

`401` with `WWW-Authenticate: Bearer` means the token is missing or expired —
see the authenticate-and-read-membership skill.

See `errors/la-ruche-qui-dit-oui-problem-types.yml` and
`conventions/la-ruche-qui-dit-oui-conventions.yml`.

## Known drift

The provider's documentation dates from April 2017 and other documented routes
now 404 on the live host. Verify these routes resolve before attempting any
money-moving call.
