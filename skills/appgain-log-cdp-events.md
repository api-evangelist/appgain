---
name: Log events and purchases into the Appgain CDP
description: Write custom user events and revenue into Appgain's customer data platform, read them back, and understand which credential each path needs.
api: postman/appgain-omnichannel.postman_collection.json
generated: '2026-08-13'
method: generated
source: >-
  Grounded in Appgain's public Postman collection 4679101/T17KeScV
  (postman/appgain-omnichannel.postman_collection.json, folder EventLogging),
  https://docs.appgain.io/core-concepts/CDP/ and
  https://docs.appgain.io/core-concepts/user-profiles/.
operations:
  - "POST https://{account_subdomain}.appgain.io/user_log_event/{projectId}/{userId} — logEvent"
  - "GET https://api.appgain.io/user_log_event/{projectId}/{userId} — get user logged events"
  - "POST {Appbackend ServerURL}/functions/logPurchase — logPurchase"
  - "POST {Appbackend ServerURL}/functions/getUserInfo — getUserInfo"
---

# Log events and purchases into the Appgain CDP

Events are the spine of Appgain: they populate user profiles, define segments, and fire
automation journeys. Get the write path right and everything downstream works.

## 1. Two credentials, two hosts — know which you are on

| Path | Host | Credential |
|---|---|---|
| `logEvent`, `get user logged events` | account subdomain / `api.appgain.io` | `appApiKey` (project-scoped) |
| `logPurchase`, `getUserInfo` | `{Appbackend ServerURL}` (Parse Server) | `x-parse-application-id` + `x-parse-master-key` |

The Parse **master key** is an unrestricted admin credential that bypasses Parse
class-level permissions and ACLs. It is not an equivalent of the project API key. Keep
it server-side only, never in a browser, a mobile binary, or an agent context that a
user can read.

## 2. Log a custom event

```
POST https://{account_subdomain}.appgain.io/user_log_event/{projectId}/{userId}
Content-Type: application/json
appApiKey: <project api key>
```

```json
{
  "action": "click",
  "type": "tt05",
  "value": {
    "campaign_id": "<campaign id>",
    "campaign_name": "company test",
    "extra_params": {"k1": 1, "k2": 5, "k5": 10}
  }
}
```

Name custom events for the behaviour, not the UI, and keep names stable — segments and
journeys are built on the exact string. Appgain's own documented examples are
`product_viewed`, `add_to_cart`, `checkout_started`, `subscription_started`,
`loan_application_submitted`.

Do **not** re-emit system events. Appgain already captures `smart link open`,
`smart link matching`, `smart link deeplink opened`, `appInstalled`, `appUninstalled`,
`appOpen`, `email received/opened/unsubscribe/resubscribe`, `SMS received`,
`appPush received/open/dismiss/conversion`, `webPush received`,
`automator trigger/received/open/conversion`, `purchase` and `landing page open`.

## 3. Log revenue

```
POST {Appbackend ServerURL}/functions/logPurchase
  ?userId=<id>&name=<product>&currency=<ccy>&amount=<n>&platform=<android|ios|web>
  &smartlink_id=<id of the smartlink that acquired the user>
x-parse-application-id: <parse app id>
x-parse-master-key: <parse master key>
```

Always pass `smartlink_id` when you have it — it is what connects revenue back to the
acquisition campaign, and it is what feeds `ltv`, `ordersNum`, `lastOrderValue` and
`lastOrderDate` on the user profile.

## 4. Read back

```
GET https://api.appgain.io/user_log_event/{projectId}/{userId}?project_sub_domain={subdomain}&appApiKey=<key>
```

There are **no pagination parameters** — no limit, cursor, offset or page. For a
high-volume user the response is whatever the server decides to return.

Profile reads go through Parse:
`POST {Appbackend ServerURL}/functions/getUserInfo?userId=<id>` with the Parse headers.

## 5. Failure modes

- `401 {"status": "failed", "message": "app API key or user authentication required"}`
  — missing or wrong project key.
- There is no idempotency key. A retried `logEvent` or `logPurchase` writes a second
  event, which double-counts revenue and can re-fire a journey. Make the caller
  idempotent on your side: record the id you sent and do not resend on ambiguity.
- No rate limits or rate-limit headers are published. Batch writes with a deliberate
  delay rather than bursting.
