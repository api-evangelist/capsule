---
name: capsule-sync-contacts
description: Incrementally synchronise Capsule contacts (parties) into an external system, including picking up deletions, without tripping the rate limit.
api: Capsule Parties API
operations:
  - GET /parties
  - GET /parties/{partyId}
  - GET /parties/search
  - GET /parties/deleted
  - GET /parties/{partyId}/people
generated: '2026-09-05'
method: generated
source: openapi/capsule-parties-api-openapi.yml + https://developer.capsulecrm.com/v2/overview/
---

# Synchronise Capsule contacts

Capsule's contact record is a **party**, and a party is either a `person` or an
`organisation`. There is no separate contacts endpoint.

## Before you start

- Base URL: `https://api.capsulecrm.com/api/v2`
- Send `Authorization: Bearer {token}` and `Accept: application/json`
- Budget: **4,000 requests per hour per Capsule user**. At `perPage=100` a full
  page costs one request, so a 60,000-contact Growth account is 600 requests.

## Steps

1. **Page the full list.** `GET /parties?perPage=100&embed=tags,fields`.
   `perPage` maxes at 100; `page` is 1-based. Do not construct the next URL
   yourself - follow the RFC 5988 `Link` header with `rel="next"` until it is
   absent.
2. **Expand only what you need.** `embed` is the expansion parameter. Each
   extra embed enlarges the response; it does not cost extra requests.
3. **Resolve organisation membership.** For a party whose `type` is
   `organisation`, `GET /parties/{partyId}/people` returns its people. This is
   one request per organisation - fold it into your rate budget before you
   start, or skip it and rebuild membership from each person's own
   organisation reference.
4. **Pick up deletions.** `GET /parties/deleted` returns the ids of parties
   removed since the point you ask about. A full re-list will never show you a
   deletion, so a synchroniser that skips this endpoint silently accumulates
   ghost records.
5. **Fetch single records only on demand.** `GET /parties/{partyId}` is for
   filling a gap, not for walking the set - one request per contact will
   exhaust the hourly quota on any real account.

## Rate limiting

Read `X-RateLimit-Remaining` on every response. On `429`, wait until the epoch
second in `X-RateLimit-Reset`. **Capsule does not send `Retry-After`**, so a
generic backoff library that only understands `Retry-After` will busy-retry
into the wall.

## Do not poll

If you want changes rather than a snapshot, subscribe a REST Hook to
`party/created`, `party/updated` and `party/deleted` instead of re-listing.
See `asyncapi/capsule-rest-hooks-webhooks.yml`.

## Errors

`401` means the token expired (OAuth access tokens live ~7 days); refresh and
retry. `403` means the token lacks the scope. `422` returns `message`,
`resource` and `field` - read `field` to find the offending property. There
are no named error codes to branch on.
