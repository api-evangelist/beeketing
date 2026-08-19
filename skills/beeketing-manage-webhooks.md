---
name: beeketing-manage-webhooks
description: Register, inspect, update and delete ShopBase webhook subscriptions so an integration reacts to store events instead of polling.
api: ShopBase Admin API
host: https://{shop}.onshopbase.com
scopes: []
operations:
  - retrieves-a-list-of-webhooks
  - create-webhook
  - get-webhook-by-id
  - update-webhook
  - delete-webhook-by-id
generated: '2026-08-13'
method: generated
source: openapi/beeketing-shopbase-admin-openapi.json
---

# Manage ShopBase webhook subscriptions

Base `https://{shop}.onshopbase.com/admin`. Webhooks are the correct way to stay
in sync with a ShopBase store: the Admin API's leaky bucket (30 tokens, 2/second)
makes polling expensive, and the pagination offset ceiling of 100,000 makes a
full re-scan impossible on a large store.

## Steps

1. **List what is already registered.** `GET /admin/webhooks.json`
   (`retrieves-a-list-of-webhooks`). Do this before creating anything — there is
   no idempotency key, so a retried create yields two subscriptions and two
   deliveries per event.

2. **Subscribe.** `POST /admin/webhooks.json` (`create-webhook`) with the topic
   and your HTTPS callback address. Valid topics come from the published catalog,
   not from guesswork — the full list is in
   `asyncapi/beeketing-webhooks.yml`:

   - carts: `carts/create`, `carts/update`
   - checkouts: `checkouts/create`, `checkouts/update`, `checkouts/delete`
   - collections: `collections/create`, `collections/update`
   - fulfillments: `fulfillments/create`, `fulfillments/update`
   - orders: `orders/create`, `orders/updated`, `orders/paid`, `orders/fulfilled`,
     `orders/partially_fulfilled`, `orders/cancelled`, `orders/delete`
   - transactions: `order_transactions/create`
   - products: `products/create`, `products/update`, `products/delete`
   - refunds: `refunds/create`
   - shop / app: `shop/update`, `app/uninstalled`
   - themes: `themes/create`, `themes/publish`, `themes/update`, `themes/delete`

3. **Inspect one.** `GET /admin/webhooks/{webhook_id}.json`
   (`get-webhook-by-id`).

4. **Change the endpoint or topic.** `PUT /admin/webhooks/{webhook_id}.json`
   (`update-webhook`).

5. **Unsubscribe.** `DELETE /admin/webhooks/{webhook_id}.json`
   (`delete-webhook-by-id`).

## Rules

- **Always subscribe to `app/uninstalled`.** It is the only signal that a
  merchant has removed your app; without it you keep state for a store you can no
  longer call.
- **Deliveries are not a contract.** ShopBase publishes no AsyncAPI document, no
  delivery-retry policy and no replay endpoint. Treat your consumer as
  at-least-once and de-duplicate on the payload id.
- **Reconcile, do not trust.** Pair each topic with the matching read operation
  (`retrieves-a-specific-order`, `get-detail-of-a-product`) and re-fetch the
  resource rather than acting on the webhook body alone.
- **Errors.** `errors` / `error` fields in the JSON body; 429 on rate-limit
  exhaustion. See `errors/beeketing-problem-types.yml`.
