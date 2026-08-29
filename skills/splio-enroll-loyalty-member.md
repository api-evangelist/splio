---
name: splio-enroll-loyalty-member
description: >-
  Enroll a customer in a Splio loyalty program and take them through their first reward — create the
  contact, create the membership, credit points, check eligibility, grant and burn a reward. Use when
  a brand needs a customer put on a card, given points, or issued a reward through the Splio Customer
  Platform API.
api: Splio Customer Platform API
base_url: https://api.splio.com
generated: '2026-08-29'
method: generated
source: >-
  openapi/splio-customer-platform-openapi.json, https://dev-scp.splio.com/docs/generic-concepts,
  https://dev-scp.splio.com/docs/loyalty-integration-guidelines
operations:
- POST /authenticate
- GET /universes/unique-key
- POST /data/contacts
- POST /loyalty/members
- GET /loyalty/programs
- GET /loyalty/members/{id}/balance
- POST /loyalty/v1/members/{card_code}/credit
- POST /loyalty/rewards/eligibility/{card_code}
- POST /loyalty/v1/rewards/{id}/grant
- POST /loyalty/v1/reward-attributions/{id}/burn
- POST /loyalty/members/{card_code}/tier_change
---

# Enroll a loyalty member in Splio

Splio's specs declare **no `operationId` on any operation**, so every step below is named by
method + path. That is the only stable handle the contract gives you.

## 0. Authenticate first — and once

`POST /authenticate` with `{"api_key": ..., "password": ...}` returns a JWT valid for **24 hours**.
Send it as `Authorization: Bearer <jwt>` on every later call. Splio explicitly warns against
authenticating per request. Cache the token and refresh on a 24-hour schedule.

## 1. Find out what a contact is keyed on — do not assume

`GET /universes/unique-key` returns whether this universe keys contacts on email, cellphone, or a
custom field. This is per-universe and it changes the meaning of every contact payload you send.
Read it before your first write, not after your first 400.

## 2. Create the contact

`POST /data/contacts`. Returns 201 on success. Custom fields are typed — send a string into a date
field and you get a 400 with `error: wrong_type` and the field named in `error_key`. To discover the
available fields first, call `GET /data/fields/contacts`.

For a backfill use `POST /data/contacts/bulk` (max **1000** per call). It can return **207
Multi-Status** — a success status with failures inside it. Always read the per-item results; do not
treat 207 as done.

## 3. Create the membership

`POST /loyalty/members` attaches the contact to a program and mints a **card code**. From here on,
loyalty operations address the member by `card_code`, not by the contact's unique key — Splio keeps
two identities for one person and gives you no operation to resolve between them. Store the mapping
yourself.

Use `GET /loyalty/programs` to pick the program, and `GET /loyalty/programs/{id}/tiers` to see the
ladder.

## 4. Credit points

`POST /loyalty/v1/members/{card_code}/credit` accepts `q_points_quantity`,
`nq_points_quantity` (qualifying and non-qualifying), a `context` string, and — importantly — an
**`idempotency_key`** in the request body.

Send a UUID in `idempotency_key` on every credit. This is one of only three operations in the whole
Splio estate that supports it, and it is the one where a retry costs the brand real money. Note the
semantics: a repeat with the same key is **refused**, not replayed. You will not get the original
response back, so record the outcome of the first call yourself.

The call returns **202**, not 200 — the credit is accepted asynchronously and there is no job-status
endpoint. Confirm with `GET /loyalty/members/{id}/balance`.

## 5. Grant a reward

`POST /loyalty/rewards/eligibility/{card_code}` first — it returns what this member can actually
claim, given points, tier and validity windows. Then `POST /loyalty/v1/rewards/{id}/grant`, which
also accepts an **`idempotency_key`**. Use it.

A grant produces a **reward attribution** — the instance the member holds. `GET
/loyalty/reward-attributions/{id}` reads one back; `GET /loyalty/v2/contacts/{id}/rewards` lists
everything a contact has been granted and used.

## 6. Burn

`POST /loyalty/v1/reward-attributions/{id}/burn` consumes the attribution at redemption.

**This is irreversible.** There is no un-burn, void or restore operation anywhere in the API.
Neither is there a debit endpoint for points. Before burning, confirm the redemption really
happened — a re-grant is a new reward, not a correction.

## Errors

Splio returns `{"status": 400, "errors": [{"error_key", "error", "error_description"}]}` for
validation failures — but a bare JSON string `"Authentication failed."` for 401 and
`"Method Not Allowed."` for 405. Handle both shapes; a client that unconditionally reads
`body.errors[0]` will crash on the first expired token.

## Rate limits

Loyalty endpoints fall under the default **10 requests/second**, limited **per client IP address**
(not per key). On 429 read `rateLimit-reset` for the seconds to wait — there is no `Retry-After`.
Splio warns that not handling 429 "may experience data loss".

See `conventions/splio-conventions.yml`, `errors/splio-problem-types.yml` and
`rate-limits/splio-rate-limits.yml`.
