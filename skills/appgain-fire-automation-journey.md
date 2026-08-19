---
name: Fire and cancel an Appgain automation journey
description: Trigger a server-side multi-channel automation journey for one user or a batch, and cancel scheduled messages, using Appgain's automator trigger-point endpoints.
api: postman/appgain-omnichannel.postman_collection.json
generated: '2026-08-13'
method: generated
source: >-
  Grounded in Appgain's public Postman collection 4679101/T17KeScV
  (postman/appgain-omnichannel.postman_collection.json, folder Campaigns/Automation),
  https://docs.appgain.io/Marketing%20Automation/ServerSideAutomation/ and the batch
  v2 form published in https://docs.appgain.io/integrations/n8n-integration/.
operations:
  - "GET https://automator.appgain.io/automessages/{projectId}/firevent/{triggerPointName}/{userId} — fire automessage"
  - "DELETE https://automator.appgain.io/automessages/{projectId}/firevent/{triggerPointName}/{userId} — cancel automessage"
  - "POST https://automator.appgain.io/automessages/{projectId}/firevent/v2/{triggerPointName} — batch fire (users[] body)"
---

# Fire and cancel an Appgain automation journey

An automation journey is configured in the Appgain dashboard — the trigger point, the
ordered messages, each message's channel, and each message's grace period. Your job from
code is only to **fire the trigger point** for a user and, when the reason for it goes
away, **cancel** it.

## 1. The journey must already exist

There is no API to create a journey, a rule, or a segment. Before firing, a human must
have created, in the dashboard:

1. a customer **segment**, and
2. an **automation rule** with a named trigger point and its message steps.

Firing an unknown trigger point does nothing useful.

## 2. Fire for one user

```
GET https://automator.appgain.io/automessages/{projectId}/firevent/{triggerPointName}/{userId}
appApiKey: <project api key>
```

Yes — a `GET` with a side effect. That is what the published collection specifies.

## 3. Fire for a batch (v2)

```
POST https://automator.appgain.io/automessages/{projectId}/firevent/v2/{triggerPointName}
appApiKey: <project api key>
Content-Type: application/json

{"users": [{"userId": "..."}, {"userId": "..."}]}
```

`v2` is the only version segment anywhere in Appgain's surface. Both forms are live and
neither is marked deprecated (`lifecycle/appgain-lifecycle.yml`), so pick one per
integration and stay on it.

## 4. Cancel

```
DELETE https://automator.appgain.io/automessages/{projectId}/firevent/{triggerPointName}/{userId}
appApiKey: <project api key>
```

Cancel when the journey's premise expires — the abandoned cart got purchased, the
subscription was cancelled, the user unsubscribed. Anything already delivered cannot be
recalled.

## 5. What the platform does for you, and what it does not

**Does:** a server-side anti-spam guard will not send the same journey message to the
same user more than once in a day even if you fire the trigger repeatedly, and it stops
the sequence if the user clicks the CTA on an earlier message.

**Does not:** de-duplicate direct `POST /{projectId}/send` calls, and does not provide
an idempotency key. The anti-spam guard is a campaign policy on journey messages only —
never rely on it as a substitute for firing correctly.

## 6. Errors

An invalid key returns `401 {"status": "failed", "message": "Not Authorized Api Key"}`.
There is no error code and no `Retry-After`. Note also that `automator.appgain.io`
reports a `-dev` build in its `x-api-version` response header; treat it as
production-critical but instrument it.

## 7. Verify

The journey emits `automator trigger`, `automator received`, `automator open` and
`automator conversion` events, each carrying the automessage name and id. Read them per
user with `GET https://api.appgain.io/user_log_event/{projectId}/{userId}`.
