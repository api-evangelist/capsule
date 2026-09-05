---
name: capsule-subscribe-webhooks
description: Subscribe to Capsule REST Hooks so an agent reacts to CRM changes instead of polling, and handle the delivery guarantees Capsule actually offers.
api: Capsule REST Hooks
operations:
  - POST /resthooks
  - GET /resthooks
  - GET /resthooks/{restHookId}
  - DELETE /resthooks/{restHookId}
generated: '2026-09-05'
method: generated
source: https://developer.capsulecrm.com/v2/overview/avoid-polling-with-rest-hooks + asyncapi/capsule-rest-hooks-webhooks.yml
---

# Subscribe to Capsule REST Hooks

Capsule's stated position is that integrations should not poll. REST Hooks are
the supported alternative.

## Prerequisites you cannot work around

- Only a **registered OAuth application** can manage subscriptions. A personal
  access token will not do it.
- Only an **account administrator** can manage hooks.
- **Maximum 20 subscriptions per account.** One hook is one event name, so
  twenty is the ceiling on distinct events you can watch - budget them.

## Steps

1. **Stand up an HTTPS receiver.** `targetUrl` must be publicly reachable and
   accept `POST` with a JSON body.
2. **Subscribe.** `POST /api/v2/resthooks` with `targetUrl`, `event`, and an
   optional `description`. One request per event.
3. **Pick the events.** Nineteen exist:
   `party/created|updated|deleted`,
   `kase/created|updated|deleted|closed|moved`,
   `opportunity/created|updated|deleted|closed|moved`,
   `task/created|updated|completed`,
   `user/created|updated|deleted`.
   Note `kase` is the wire name for **Project** - the product renamed Cases to
   Projects in 2022 and the API did not follow.
4. **Handle the payload.** The body is
   `{"event": "<name>", "payload": [ ...entities... ]}`. **`payload` is an
   array** - one delivery can carry several affected records. Iterate it;
   never index `[0]`.
5. **Verify what you can.** Capsule publishes **no signature or shared secret**
   for webhook deliveries. Treat the `targetUrl` itself as the only secret:
   make it unguessable, serve it over TLS, and re-fetch the entity from the
   API before acting on anything consequential rather than trusting the
   posted body.
6. **Return quickly.** Failed deliveries retry 5 times - at 5s, 1m, 5m, 15m,
   15m - roughly 36 minutes in total. After that the hook is marked dead and
   **automatically unsubscribed**, silently. Monitor `GET /api/v2/resthooks`
   and re-subscribe if a hook disappears.
7. **Unsubscribe deliberately.** `DELETE /api/v2/resthooks/{restHookId}`, or
   have your receiver return `410 Gone` to unsubscribe itself.

## Backfill

REST Hooks tell you about changes from now on. Combine them with
`capsule-sync-contacts` for the initial load and with the `/deleted` change
feeds to reconcile anything missed during an outage longer than the 36-minute
retry window.
