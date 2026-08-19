---
name: beeketing-recover-abandoned-checkouts
description: List abandoned ShopBase checkouts, preview the recovery email or SMS, send it, and cancel a scheduled send.
api: ShopBase Admin API
host: https://{shop}.onshopbase.com
scopes: [read_orders, write_orders]
operations:
  - count-checkouts
  - retrieves-a-list-of-checkouts
  - retrieves-a-checkout
  - retrieves-a-list-of-checkout-timelines
  - get-email-preview
  - send-recovery-email
  - cancel-recovery-email
  - get-sms-preview
  - send-recovery-sms
  - cancel-recovery-sms
  - update-checkout-note
generated: '2026-08-13'
method: generated
source: openapi/beeketing-shopbase-admin-openapi.json
---

# Recover abandoned ShopBase checkouts

Base `https://{shop}.onshopbase.com/admin`. This flow sends messages to real
buyers — preview before you send, and never retry a send blindly.

**Scope note.** ShopBase documents `read_checkouts` / `write_checkouts`, but the
published spec declares `read_orders` / `write_orders` on every checkout
operation. Request the order scopes; add the checkout scopes only if a call
returns 403.

## Steps

1. **Size the queue.** `GET /admin/checkouts/count.json` (`count-checkouts`).

2. **List abandoned checkouts.** `GET /admin/checkouts.json`
   (`retrieves-a-list-of-checkouts`), paged with `page` and `limit`. Fetch one
   with `GET /admin/checkouts/{checkout_id}.json` (`retrieves-a-checkout`).

3. **Read the history before acting.**
   `GET /admin/checkouts/{checkout_id}/timeline.json`
   (`retrieves-a-list-of-checkout-timelines`) shows what has already been sent.
   With no idempotency key on this API, the timeline is the only defence against
   messaging the same buyer twice.

4. **Preview.**
   `GET /admin/checkouts/{checkout_id}/send-recovery-email/get-review.json`
   (`get-email-preview`) or
   `.../send-recovery-sms/get-review.json` (`get-sms-preview`).

5. **Send.**
   `POST /admin/checkouts/{checkout_id}/send-recovery-email.json`
   (`send-recovery-email`) or
   `.../send-recovery-sms.json` (`send-recovery-sms`).

6. **Cancel a scheduled send** if the buyer converts first:
   `POST /admin/checkouts/{checkout_id}/cancel-send-recovery-email.json`
   (`cancel-recovery-email`) or `.../cancel-send-recovery-sms.json`
   (`cancel-recovery-sms`).

7. **Annotate.** `PUT /admin/checkouts/{checkout_id}/update-note.json`
   (`update-checkout-note`).

## Rules

- **A send is not idempotent.** A timeout on step 5 may still have delivered the
  message. Re-read the timeline (step 3) before retrying — an accidental second
  send is a deliverability and consent problem, not just a duplicate row.
- **Rate limit.** 30-token leaky bucket, 2/second leak per shop;
  `X-Sb-Shop-Api-Call-Limit` reports `current/30`, exhaustion returns 429.
- **Errors.** `errors` / `error` in the body; 403 means a missing scope.
- **Events.** `checkouts/create`, `checkouts/update`, `checkouts/delete` and
  `orders/create` webhooks tell you when a checkout converts —
  `asyncapi/beeketing-webhooks.yml`.
