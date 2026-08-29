---
name: splio-sync-ecommerce-data
description: >-
  Feed an e-commerce catalogue and its sales history into the Splio Customer Platform — contacts,
  products, stores, orders, abandoned carts and refunds — and keep them in sync. Use when connecting
  a storefront, POS or data warehouse to Splio through the REST API rather than a packaged connector.
api: Splio Customer Platform API
base_url: https://api.splio.com
generated: '2026-08-29'
method: generated
source: >-
  openapi/splio-customer-platform-openapi.json,
  https://dev-scp.splio.com/docs/integration-guidelines,
  https://dev-scp.splio.com/docs/how-to-manage-refunds,
  https://dev-scp.splio.com/docs/how-to-choose-your-contact-unique-key
operations:
- POST /authenticate
- GET /universes/unique-key
- GET /data/fields/contacts
- POST /data/contacts/bulk
- PATCH /data/contacts/{individual_id}
- POST /data/v1/products
- POST /data/v1/stores
- POST /data/v1/orders
- PATCH /data/v3/orders/{id}
- GET /data/v2/orders
- GET /data/v1/orders/abandoned
- POST /events/
---

# Sync e-commerce data into Splio

## Before anything: three identifier systems, no bridge

Splio addresses the same customer three ways and gives you nothing that maps between them:

| Entity | Identified by |
|---|---|
| Contact | the universe's `unique_key` — email, cellphone **or** a custom field |
| Order / Product / Store | **your** `external_id` (`ext_id` for stores) |
| Loyalty member | `card_code` |

Call `GET /universes/unique-key` first and hold the answer. Maintain your own mapping table; the API
will not do it for you.

## 1. Authenticate

`POST /authenticate` → JWT, valid 24 hours, sent as `Authorization: Bearer <jwt>`. Refresh on a
timer, never per call.

## 2. Discover the schema before you write

`GET /data/fields/contacts`, `/orders`, `/order-lines`, `/products`, `/stores` return the custom
fields defined in this universe and their types. Create anything missing with `POST /data/fields`.
Writing an untyped or wrongly-typed value returns 400 `wrong_type` naming the field.

## 3. Load in dependency order

Stores and products before orders — an order references `store_external_id` and `product_id`, and a
dangling reference is not resolved retroactively.

1. `POST /data/v1/stores` for each point of sale.
2. `POST /data/v1/products` for the catalogue.
3. `POST /data/contacts/bulk` for customers, **1000 per call**.
4. `POST /data/v1/orders` for purchases and abandoned carts — the same endpoint creates both; the
   cart is an order with abandoned status.

`POST /data/contacts/bulk` returns **207 Multi-Status** on partial failure. A 2xx does not mean the
batch applied. Read every per-item result and re-drive the failures.

## 4. Rate-limit the loader properly

Contacts and sales data are capped at **20 requests/second**; everything else at **10 RPS**. The
bucket is **per client IP address**, so parallel workers behind one NAT gateway share it and a
horizontally-scaled loader will throttle itself.

Watch `rateLimit-remaining` and back off before you hit zero. On 429 the body is
`{"message": "API rate limit exceeded"}` — a different envelope from every other error — and the
wait is in `rateLimit-reset`, in seconds. There is no `Retry-After`.

None of the sales-data writes support `idempotency_key`; only the loyalty credit/grant and the
Messaging API do. A retried `POST /data/v1/orders` is a second order. Make your loader durable on
your own side: record what you sent, and use `GET /data/v2/orders/{id}` to check before re-sending.

## 5. Refunds are negative orders, not deletions

Splio is explicit: *"Refunds are registered like orders, they are not a distinct data entity."*
Create a **new** order with:

- price: the negative of the original price
- quantity: `0` (it cannot be negative)
- total price: the negative of the refunded amount

`DELETE /data/orders/{id}` exists but is a data-hygiene delete, not a business reversal — deleting
the original order destroys the purchase history the segmentation depends on. Do not use it to
model a return.

Refunds then need excluding from segments deliberately: build a target filter on the order scope for
negative totals and include it in your contact filters.

## 6. Behavioural events

Custom interactions go to the Interactions API — `POST /events/` for one,
`POST /events/bulk` for many, same bearer token, `https://api.splio.com`. Page views, cart and
purchase events from the storefront itself are better sent through Splio's published GTM tags.

## 7. Incremental sync

- `GET /data/v2/orders` lists orders **with status** — prefer it to the v1 path, which omits status.
- `GET /data/v1/orders/abandoned` lists carts.
- `PATCH /data/v3/orders/{id}` is the current edit path (v1 and v2 read paths still exist; v3 is
  the write path — the versions are per-resource and do not move together).
- `PATCH /data/contacts/{individual_id}` for contact deltas.

## Alternative: don't hand-roll it

If the source is Shopify, Magento 2, PrestaShop or anything Make can reach, Splio publishes a
packaged connector, and the Datahub SFTP/CSV path handles bulk file loads with a declarative
configuration file. Reach for the API when none of those fit.

See `conventions/splio-conventions.yml`, `data-model/splio-data-model.yml` and
`rate-limits/splio-rate-limits.yml`.
