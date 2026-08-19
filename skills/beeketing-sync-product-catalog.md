---
name: beeketing-sync-product-catalog
description: Page a ShopBase product catalog, then create or update products, variants, images and collection membership.
api: ShopBase Admin API
host: https://{shop}.onshopbase.com
scopes: [read_products, write_products]
operations:
  - get-count-of-all-products
  - get-all-products
  - get-detail-of-a-product
  - create-a-product
  - update-a-product
  - patch-multiple-product
  - get-all-variants-of-a-product
  - create-a-variant-of-a-product
  - update-a-variant-of-a-product
  - get-all-images-of-a-product
  - create-a-product-image
  - update-multi-product-images-for-multi-variants
  - retrieve-a-list-of-custom-collections
  - create-a-collect
generated: '2026-08-13'
method: generated
source: openapi/beeketing-shopbase-admin-openapi.json
---

# Sync a ShopBase product catalog

Base `https://{shop}.onshopbase.com/admin`. Requires `read_products` to read and
`write_products` to write; a write scope implicitly grants the matching read
scope.

## Steps

1. **Size the job.** `GET /admin/products/count.json`
   (`get-count-of-all-products`).

2. **Page the catalog.** `GET /admin/products.json` (`get-all-products`) with
   `page` and `limit`. **Hard ceiling:** once `page * limit` exceeds 100,000 the
   API returns 429, so for a large catalog narrow the query rather than paging
   deeper — `GET /admin/products/collections.json`
   (`get-products-by-collection-id`), or the vendor/type/tag facets at
   `/admin/products/vendors.json`, `/admin/products/types.json` and
   `/admin/products/tags.json`.

3. **Create or update.** `POST /admin/products.json` (`create-a-product`) or
   `PUT /admin/products/{product_id}.json` (`update-a-product`). For a bulk field
   change use `PATCH /admin/products.json` (`patch-multiple-product`).

4. **Variants.** `GET /admin/products/{product_id}/variants.json`
   (`get-all-variants-of-a-product`), then
   `POST` the same path (`create-a-variant-of-a-product`) or
   `PUT /admin/variants/{variant_id}.json` (`update-a-variant-of-a-product`).
   Note the asymmetry: variants are created under the product but updated at the
   top level.

5. **Images.** `POST /admin/products/{product_id}/images.json`
   (`create-a-product-image`), then bind images to variants with
   `PUT /admin/products/{product_id}/images.json`
   (`update-multi-product-images-for-multi-variants`).

6. **Collection membership.** List collections with
   `GET /admin/collections/custom.json`
   (`retrieve-a-list-of-custom-collections`), then join a product to one with
   `POST /admin/collects.json` (`create-a-collect`). Smart collections are
   rule-based and are not joined through Collect.

## Rules

- **Rate limit.** 30-token leaky bucket, 2 requests/second leak, per shop. A full
  catalog sync will hit it — read `X-Sb-Shop-Api-Call-Limit` and pace to the leak
  rate rather than sprinting into 429s.
- **No idempotency key.** A retried `create-a-product` creates a second product.
  Key your sync on your own external id and re-read before you re-create.
- **Errors.** `errors` / `error` in the body; 403 means a missing scope.
- **Events.** `products/create`, `products/update`, `products/delete` and
  `collections/*` webhooks replace polling — `asyncapi/beeketing-webhooks.yml`.
