---
name: beeketing-fulfill-an-order
description: Find an unfulfilled ShopBase order, create a fulfillment for its line items, then complete or cancel it.
api: ShopBase Admin API
host: https://{shop}.onshopbase.com
scopes: [read_orders, write_orders, write_fulfillments]
operations:
  - retrieves-a-list-of-orders
  - retrieves-a-specific-order
  - receive-a-list-of-all-fulfillmentservices
  - retrieves-fulfillments-associated-with-an-order
  - create-fulfillment
  - update-fulfillment
  - complete-a-fulfillment
  - cancel-a-fulfillment
generated: '2026-08-13'
method: generated
source: openapi/beeketing-shopbase-admin-openapi.json
---

# Fulfill a ShopBase order

Every call goes to `https://{shop}.onshopbase.com/admin/...` where `{shop}` is the
store handle. Send the per-store token in the `APP_ACCESS_TOKEN` header (public
app, OAuth 2.0 authorization code) or HTTP Basic credentials (private app). All
paths end in `.json`.

## Steps

1. **Find the order.** `GET /admin/orders.json` (`retrieves-a-list-of-orders`).
   Page with `page` and `limit`; keep `page * limit` under 100,000 or the call
   returns 429. Then `GET /admin/orders/{order_id}.json`
   (`retrieves-a-specific-order`) for the full line items.

2. **Check what is already fulfilled.**
   `GET /admin/orders/{order_id}/fulfillments.json`
   (`retrieves-fulfillments-associated-with-an-order`). There is **no idempotency
   key** on this API — this read is the only thing standing between a retry and a
   duplicate fulfillment.

3. **Resolve the fulfillment service** if the items ship through one:
   `GET /admin/fulfillment_services.json`
   (`receive-a-list-of-all-fulfillmentservices`).

4. **Create the fulfillment.** `POST /admin/orders/{order_id}/fulfillments.json`
   (`create-fulfillment`) with the line items and tracking details. Requires
   `write_orders`.

5. **Move it to its end state.**
   `POST /admin/orders/{order_id}/fulfillments/{fulfillment_id}/complete.json`
   (`complete-a-fulfillment`), or
   `.../open.json` (`mark-a-fulfillment-as-open`) to reopen, or
   `.../cancel.json` (`cancel-a-fulfillment`) to cancel.
   Tracking edits go through `PUT .../fulfillments/{fulfillment_id}.json`
   (`update-fulfillment`).

## Rules

- **Rate limit.** Leaky bucket, size 30, leaking 2 requests/second per shop.
  Read `X-Sb-Shop-Api-Call-Limit` (format `22/30`) on every response and back off
  before it fills. On 429, wait and retry.
- **Errors.** Detail arrives in the `errors` or `error` field of the body, not as
  `application/problem+json`. 403 means the token lacks the scope, 402 means the
  shop is frozen, 423 means the shop is locked. See
  `errors/beeketing-problem-types.yml`.
- **Retries are not safe.** Re-read step 2 before re-issuing step 4.
- **Events.** Subscribe to `fulfillments/create` and `fulfillments/update` rather
  than polling — see `asyncapi/beeketing-webhooks.yml`.
