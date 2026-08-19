---
name: beeketing-refund-an-order
description: Calculate a ShopBase refund before creating it, then record the refund and verify the resulting transaction.
api: ShopBase Admin API
host: https://{shop}.onshopbase.com
scopes: [read_orders, write_orders]
operations:
  - retrieves-a-specific-order
  - retrieves-a-list-of-transactions
  - retrieves-a-list-of-refunds-for-an-order
  - calculate-the-refund
  - create-the-refund
  - retrieve-a-specific-refund
  - retrieves-a-specific-transaction
generated: '2026-08-13'
method: generated
source: openapi/beeketing-shopbase-admin-openapi.json
---

# Refund a ShopBase order

Base `https://{shop}.onshopbase.com/admin`. Token in `APP_ACCESS_TOKEN`, or HTTP
Basic for a private app. This flow moves money — read the safety rules first.

## Steps

1. **Load the order.** `GET /admin/orders/{order_id}.json`
   (`retrieves-a-specific-order`).

2. **Read what has already been refunded.**
   `GET /admin/orders/{order_id}/refunds.json`
   (`retrieves-a-list-of-refunds-for-an-order`). There is no idempotency key on
   this API, so this read is the guard against a double refund.

3. **Calculate first.** `POST /admin/orders/{order_id}/refunds/calculate.json`
   (`calculate-the-refund`) with the line items and quantities. This returns the
   refundable amounts, shipping and taxes without moving money. Never skip it —
   compute the amount yourself only if you intend to override the API's own
   arithmetic.

4. **Create the refund.** `POST /admin/orders/{order_id}/refunds.json`
   (`create-the-refund`) using the transactions the calculate step returned.
   Requires `write_orders`.

5. **Verify.** `GET /admin/orders/{order_id}/refunds/{refund_id}.json`
   (`retrieve-a-specific-refund`), and
   `GET /admin/orders/{order_id}/transactions.json`
   (`retrieves-a-list-of-transactions`) to confirm the refund transaction landed.

## Rules

- **A failed create is ambiguous.** A timeout or 5xx on step 4 does not tell you
  whether the refund was recorded. Re-run step 2 before retrying, always.
- **Rate limit.** 30-token leaky bucket, 2/second leak, per shop.
  `X-Sb-Shop-Api-Call-Limit` carries `current/30`; a 429 means back off.
- **Errors.** `errors` / `error` fields in the JSON body. 422 is a well-formed
  request with semantic problems — read the field, do not retry blindly.
- **Events.** `refunds/create` and `order_transactions/create` webhooks confirm
  the outcome asynchronously — `asyncapi/beeketing-webhooks.yml`.
