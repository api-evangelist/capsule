---
name: capsule-manage-opportunity
description: Create and advance a Capsule sales opportunity through a pipeline safely, given that Capsule has no idempotency keys and no undo.
api: Capsule Opportunities API
operations:
  - GET /opportunities
  - POST /opportunities
  - GET /opportunities/{opportunityId}
  - PUT /opportunities/{opportunityId}
  - DELETE /opportunities/{opportunityId}
  - GET /opportunities/search
  - GET /parties/{partyId}/opportunities
  - GET /opportunities/{opportunityId}/parties
  - POST /opportunities/{opportunityId}/parties/{partyId}
generated: '2026-09-05'
method: generated
source: openapi/capsule-opportunities-api-openapi.yml + https://developer.capsulecrm.com/v2/overview/
---

# Manage a Capsule opportunity

## Safety rules that govern every step

- **There is no idempotency key.** Capsule documents no `Idempotency-Key`
  header on any write. If a `POST /opportunities` times out, retrying creates
  a **second** opportunity. Search before you retry (step 1), never retry
  blind.
- **There is no undo.** No restore, revert or rollback operation exists in the
  Capsule v2 API, and no recovery window is published. A `DELETE` and an
  overwriting `PUT` are both one-way as far as the API is concerned.
- Because of both of the above, treat `DELETE /opportunities/{opportunityId}`
  as an operation requiring explicit human confirmation.

## Steps

1. **Check it does not already exist.** `GET /opportunities/search?q=<name>`,
   or `GET /parties/{partyId}/opportunities` to see everything already open
   against that contact. This is your idempotency substitute.
2. **Resolve the party.** An opportunity belongs to a party. Get the
   `partyId` first - `GET /parties/search?q=...`.
3. **Resolve the pipeline and milestone.** The opportunity's position is a
   `milestone` inside a `pipeline`. Multiple pipelines exist only on Growth
   plans and above; on Free and Starter there is exactly one.
4. **Create.** `POST /opportunities` with the body wrapped in an
   `opportunity` key:
   `{"opportunity": {"party": {"id": 123}, "name": "...", "milestone": {"id": 4}}}`.
   Expect `201 Created` plus a `Location` header carrying the new resource
   URL. Record the id immediately - it is your only handle.
5. **Advance it.** `PUT /opportunities/{opportunityId}` with only the fields
   that change. Capsule merges: fields you omit are left alone. Moving the
   `milestone` fires an `opportunity/moved` webhook; closing it fires
   `opportunity/closed`.
6. **Close-lost.** Set the milestone to the lost one and supply a
   `lostReason`. Read the account's lost reasons first rather than inventing
   a string.
7. **Link extra parties.** `POST /opportunities/{opportunityId}/parties/{partyId}`
   attaches an additional contact.

## Long-running responses

A `202 Accepted` means the work is queued. Poll the URL in the `Location`
header for job status rather than assuming the change landed.
