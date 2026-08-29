---
name: splio-manage-consent-and-suppression
description: >-
  Handle consent, opt-out, suppression and erasure in Splio — list subscribe/unsubscribe, bulk
  opt-out, email and SMS blacklists, and GDPR contact deletion. Use when implementing a preference
  centre, honouring an unsubscribe, or servicing a data-subject request against a Splio universe.
api: Splio Customer Platform API
base_url: https://api.splio.com
generated: '2026-08-29'
method: generated
source: >-
  openapi/splio-customer-platform-openapi.json,
  https://splio.com/en/personal-data-protection-policy/,
  https://dev-scp.splio.com/docs/storage-value-for-lists-subscriptions
operations:
- POST /authenticate
- GET /data/v1/lists
- POST /data/contacts/{id}/lists/subscribe
- POST /data/contacts/{id}/lists/unsubscribe
- POST /data/contacts/optout
- GET /data/blacklists/emails
- GET /data/blacklists/cellphones
- POST /data/blacklists/{channel}
- DELETE /data/blacklists/{channel}/{source}
- DELETE /data/contacts/{id}
- DELETE /data/contacts/bulk
---

# Consent, suppression and erasure in Splio

Splio is an EU processor operating under GDPR with servers "located exclusively in the European
Union" and a named DPO at `dpo@splio.com`. The operations below are the ones that carry legal weight.
Get the reversibility right before you run any of them at scale.

## 0. Authenticate

`POST /authenticate` → 24-hour JWT.

## 1. List subscription — reversible, symmetric

- `GET /data/v1/lists` enumerates the universe's membership lists.
- `POST /data/contacts/{id}/lists/subscribe` — subscribe to one or more.
- `POST /data/contacts/{id}/lists/unsubscribe` — the exact inverse.

These are a clean reversible pair. No window is documented on either, which means Splio does not
state a deadline — not that one does not exist. Do not promise a customer a reversal window Splio
has not published.

## 2. Global opt-out — NOT symmetric

`POST /data/contacts/optout` opts contacts out of **all** lists, up to 1000 per call.

There is no bulk re-opt-in. Undoing this means calling
`POST /data/contacts/{id}/lists/subscribe` once per contact per list, and you can only do that if
you recorded which lists they were on beforehand. **Snapshot list membership before opting anyone
out.** After the call, the information needed to reverse it is gone.

## 3. Blacklists — reversible, per channel

- `POST /data/blacklists/{channel}` adds touchpoints (email addresses, cellphones).
- `DELETE /data/blacklists/{channel}/{source}` removes them.
- `GET /data/blacklists/emails` and `/cellphones` read the current suppression state.

Both write operations can return **207 Multi-Status** — "Blacklisting touchpoint(s) is partially
successful" / "Deleting touchpoint(s) from blacklist is partially successful". A 2xx here is not a
guarantee that a given address is suppressed. For anything consent-related, read the per-item
results and verify with a `GET`.

## 4. Erasure — irreversible, and asynchronous

- `DELETE /data/contacts/{id}` — one contact.
- `DELETE /data/contacts/bulk` — up to 1000, **asynchronous**.

No restore, no undelete, no documented soft-delete retention period. The bulk form is asynchronous
and there is no job-status endpoint, so you cannot even observe the point of no return.

Before running erasure:

1. Confirm the identifier against `GET /universes/unique-key` — the universe's contact key may be
   email, cellphone or a custom field, and deleting on the wrong key deletes the wrong person.
2. Read the contact back with `GET /data/contacts/{individual_id}` and match it against the request.
3. Export what you need for your own audit trail. Splio will not have it afterwards.
4. Delete.

Deleting a contact does not undo their commercial history in a useful sense either — orders
reference contacts, and a refund in Splio is modelled as a compensating negative order, not a
deletion. See `conventions/splio-conventions.yml`.

## 5. What is not in the API

Double opt-in (shipped October 2025) and the contact preference centre (April 2025) are application
features, not endpoints. If your flow needs them, they are configured in Splio, and the API sees only
their effect on list membership.

## Rate limits

Contact endpoints run at **20 requests/second**, blacklists at the default **10 RPS**, both **per
client IP**. Splio warns that ignoring 429 plus the rate-limit headers "may experience data loss" —
on a consent pipeline that is a compliance failure, not a retry.

See `errors/splio-problem-types.yml` and `conformance/splio-conformance.yml`.
