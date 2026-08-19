---
name: Send an Appgain omnichannel campaign
description: Send a push, email, SMS, web push or WhatsApp message through Appgain's single /send endpoint, choosing the channel by body key and the audience by receiver shape.
api: postman/appgain-omnichannel.postman_collection.json
generated: '2026-08-13'
method: generated
source: >-
  Grounded in Appgain's own public Postman collection 4679101/T17KeScV
  (postman/appgain-omnichannel.postman_collection.json) and
  https://docs.appgain.io/integrations/n8n-integration/. Appgain publishes no
  OpenAPI, so operations are identified by the collection's request names, HTTP
  method and exact URL rather than by operationId. Every request below appears
  verbatim in that collection or in Appgain's published integration guide.
operations:
  - "POST https://notify.appgain.io/{projectId}/send — send apppush"
  - "POST https://notify.appgain.io/{projectId}/send — send email"
  - "POST https://notify.appgain.io/{projectId}/send — send sms specific usersid"
  - "POST https://notify.appgain.io/{projectId}/send — send webpush"
  - "POST https://notify.appgain.io/{projectId}/send — WHATSAPP / UOWHATSAPP"
  - "POST https://notify.appgain.io/{projectId}/create_whatsapp_template"
---

# Send an Appgain omnichannel campaign

**This sends real messages to real people. Appgain has no test mode, no test API key
and no sandbox host** (`sandbox/appgain-sandbox.yml`). The only isolation available is
a separate Appgain project. Confirm with a human before every send.

## 1. Credentials

- `projectId` — path segment, identifies the tenant.
- `appApiKey` — request header. Appgain's own examples spell it `appApiKey` on
  `api.` / `automator.` and lowercase `appapikey` on `notify.`; HTTP headers are
  case-insensitive, so either works.

Both come from the Appgain dashboard: **Project Settings → API and SDK Keys**.

## 2. One endpoint, six channels

Every channel is the same call. The channel is selected by the **top-level body key**,
not by the path:

| Body key | Channel |
|---|---|
| `appPush` | mobile push |
| `email` | email |
| `SMS` | SMS |
| `webPush` | web push |
| `WHATSAPP` | WhatsApp Business API (vendor-routed) |
| `UOWHATSAPP` | WhatsApp Lite |

```
POST https://notify.appgain.io/{projectId}/send
Content-Type: application/json
appapikey: <project api key>
```

## 3. Choose the audience

Inside the channel object, pick exactly one targeting shape:

- `receivers: [{ "userID": "...", "email": "..."|"mobileNum": "..." }]` — explicit recipients.
- `users_id: "id1,id2"` — a **comma-separated string**, not an array.
- `target_list_name: "..."` — a named SMS/WhatsApp list.
- omit all three — **sends to every user in the project**. Never do this without explicit confirmation.

## 4. Channel bodies

Push (from *send apppush text*):

```json
{"appPush": {"message": "Test Push", "sound": "default", "campaignName": "campName",
  "receivers": [{"userID": "<user id>"}]}, "campaign_name": "campName"}
```

Rich push adds `"type"` — one of `photo`, `gif`, `video`, `webView`, `htmlWebView` —
plus `attachment`, `image-url`, `mutable-content: 1` and `category: "rich-apns"`.
Silent push uses `content-available: 1` with an empty `sound`.

Email (from *send email*):

```json
{"email": {"message": "<h1>Hello</h1>", "template_name": "HTML_Invoice",
  "subject": "test", "type": "HTML",
  "receivers": [{"userID": "<user id>", "email": "<address>"}]}}
```

SMS (from *send sms -to spefic numbers*):

```json
{"SMS": {"message": "Testing team",
  "receivers": [{"userID": "<user id>", "mobileNum": "<e164 number>"}]}}
```

WhatsApp Business requires a template that Meta has already approved. Create it first
with `POST /{projectId}/create_whatsapp_template`, then send with
`{"WHATSAPP": {"vendor": "routemobile", "template_name": "...", "lang_code": "en",
"parameters": {"body": [{"text": "..."}]}, "receivers": [{"mobileNum": "..."}]}}`.
That path additionally carries an `Authorization` bearer JWT and an `authToken` header
whose issuance Appgain does not document — obtain them from the dashboard/account team.

WhatsApp Lite (`UOWHATSAPP`) needs no template but is capped at **300 messages/day**.

## 5. Handle the response — carefully

- Success and failure both return `{"status": ..., "message": ...}`; `status` is
  `"failed"` on `api.`/`automator.` and `"Failed"` on `notify.`. There is no error code.
- **A bad or missing API key on `/send` returns HTTP 500, not 401**, with
  `{"exception": "'NoneType' object has no attribute 'api_key'", ...}`
  (`errors/appgain-problem-types.yml`). Do not treat a 500 from `/send` as transient —
  check the credential first.
- **There is no idempotency key.** Retrying a `/send` sends the campaign again. On any
  ambiguous failure, stop and ask a human; do not auto-retry.
- No rate-limit headers are returned and no limit is published
  (`rate-limits/appgain-rate-limits.yml`). Pace conservatively.

## 6. Verify

Delivery shows up as CDP system events — `appPush received/open/dismiss/conversion`,
`email received/opened`, `SMS received`, `webPush received` — each stamped with
`campaign name` and `campaign id`. Read them per user with
`GET https://api.appgain.io/user_log_event/{projectId}/{userId}`.
