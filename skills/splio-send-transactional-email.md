---
name: splio-send-transactional-email
description: >-
  Send a transactional or triggered email through Splio Edition 2025 — upload a design with the
  Content API, then send it with the Messaging API or launch it as a one-shot campaign with the
  Campaign API. Use when a third-party system needs to trigger email through Splio, or when
  migrating off the legacy Launch or Trigger APIs.
api: Splio Messaging API, Splio Content API, Splio Campaign API
base_url: https://api.splio.com
generated: '2026-08-29'
method: generated
source: >-
  openapi/splio-messaging-api-openapi.json, openapi/splio-content-api-openapi.json,
  openapi/splio-campaign-api-v1-openapi.json,
  https://dev-scp.splio.com/docs/migrate-from-launch-api-to-content-campaign-apis-splio-edition-2025
operations:
- POST /authenticate
- POST /v1/create-email-design
- POST /messages
- POST /v1/one-shot
- GET /v1/one-shot/{id}
- POST /v1/one-shot/stop/{id}
---

# Send email through Splio Edition 2025

Three separate APIs, three separate base URLs, one bearer token.

| Purpose | Base URL | Operation |
|---|---|---|
| Create content | `https://api.splio.com/content` | `POST /v1/create-email-design` |
| Send transactional | `https://api.splio.com/messaging/v1/{universe_id}` | `POST /messages` |
| Launch a campaign | `https://api.splio.com/campaign-api` | `POST /v1/one-shot` |

The Messaging base URL is templated on `universe_id` — your account identifier goes in the path, not
a header.

## 0. Authenticate

`POST /authenticate` against `https://api.splio.com` → a JWT good for 24 hours, used on all three.

## 1. Create the design

`POST /v1/create-email-design` accepts either an HTML file or a ZIP, and returns the design's
**UUID**. The ZIP structure is fixed:

```
design.zip
├── index.html or mail.html    (required, at the ROOT)
└── images/                    (optional)
```

Image references in the HTML must be relative (`images/filename.ext`). Supported formats: PNG, JPG,
JPEG, AVIF, GIF, JFIF, WEBP, WOFF.

Keep the returned UUID — both send paths take it.

## 2a. Transactional: POST /messages

Send to `https://api.splio.com/messaging/v1/{universe_id}/messages`. Body carries `channel`
(currently `email` only), `recipients`, `properties`, and — critically — **`idempotency_key`**.

Splio's own wording: *"This key must be used if you want to avoid duplicates. When receiving 2 calls
with the same idempotency key, we refuse the second call."* **Refused, not replayed.** You will not
get the original message back, so log the first result. Set a UUID on every send.

Two personalisation modes:

- **Contact already in Splio** — personalise from Splio's own fields with Liquid.
- **Contact external to Splio** — pass the personalisation data in the request.

Rate limit: **8 requests/second**, lower than every other endpoint, and per client IP.

**There is no unsend.** No recall, no cancel, no void. `POST /messages` is the point of no return
for that recipient.

## 2b. Campaign: POST /v1/one-shot

For a marketing send to a target rather than named recipients. `GET /v1/one-shot/{id}` reads it back;
`POST /v1/one-shot/stop/{id}` stops one in flight and returns **202**.

`stop` is the only reversal in the sending surface, and Splio does not document how long after
launch it remains effective, nor what happens to messages already handed to the MTA. Treat it as
best-effort mitigation, not an undo.

Neither `POST /v1/one-shot` nor `stop` supports `idempotency_key`. A retried launch is a second
campaign to the same audience.

## 3. Migrating off the legacy APIs

- **Launch API → Content + Campaign.** The legacy call fetched content by URL and sent in one step.
  Content must now be created through the Content API to get an `email_design_id`. Splio publishes
  no sunset date.
- **Trigger API → Messaging API.** Deprecated October 2024. Still reachable after an SE2025 upgrade,
  but for design changes only; new feeds must go to `POST /messages`. Again, no sunset date.
- **SMTP Relay is unaffected** — Splio states explicitly that nothing needs changing there.

Neither legacy path is marked `deprecated: true` anywhere in the specs, so a generated client will
not warn you.

## Watch for change

There is no developer changelog — `dev-scp.splio.com/changelog` 404s. API changes are announced in
the marketing-facing Help Center product-update stream at
https://helpcenter.splio.com/kb/en/splio-product-updates-405810, dated by month, with no
breaking/non-breaking labels. Poll it.

See `lifecycle/splio-lifecycle.yml` and `changelog/splio-changelog.yml`.
