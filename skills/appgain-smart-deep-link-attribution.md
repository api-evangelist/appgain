---
name: Create an Appgain smart deep link and read its attribution
description: Create a smart deep link with per-platform targets and tracking pixels, upload the media it references, and follow the attribution chain from link to install to purchase.
api: postman/appgain-omnichannel.postman_collection.json
generated: '2026-08-13'
method: generated
source: >-
  Grounded in Appgain's public Postman collection 4679101/T17KeScV
  (postman/appgain-omnichannel.postman_collection.json),
  https://docs.appgain.io/smartDeepLink/gettingStarted/ and
  https://docs.appgain.io/core-concepts/CDP/. Operations are named by the
  collection's request name, method and URL — Appgain publishes no OpenAPI.
operations:
  - "POST https://{account_subdomain}.appgain.io/apps/{projectId}/smartlinks — create SmartLink"
  - "POST https://{account_subdomain}.appgain.io/apps/{projectId}/smartlinksbulk — create Smart Link Bulk"
  - "POST https://{account_subdomain}.appgain.io/{projectId}/upload — upload photo"
  - "GET https://{account_subdomain}.appgain.io/{projectId}/upload — list media files"
  - "GET https://api.appgain.io/user_log_event/{projectId}/{userId} — get user logged events"
---

# Create an Appgain smart deep link and read its attribution

A smart deep link is Appgain's acquisition and attribution unit. Its id propagates onto
the user profile (`smartlink_id`), onto events, and onto purchases — so getting the link
right is what makes the revenue numbers downstream meaningful.

## 1. Host

Smart links and media do **not** live on `api.appgain.io`. They live on your account
subdomain: `https://{account_subdomain}.appgain.io`. Note the path split — smart links
sit under `/apps/{projectId}/…`, media sits under `/{projectId}/…`. Authenticate with
the `appApiKey` header.

## 2. Upload the creative first

```
POST https://{account_subdomain}.appgain.io/{projectId}/upload
appApiKey: <key>
Content-Type: multipart/form-data
form field: photo=@file.png   (or video=@file.mp4)
```

`GET` the same URL lists the library. Use the returned URL as the smart link's `image`.

## 3. Create the link

From *create SmartLink*:

```
POST https://{account_subdomain}.appgain.io/apps/{projectId}/smartlinks
accept: application/json
Content-Type: application/json
appApiKey: <key>
```

```json
{
  "name": "Product SKU",
  "description": "Product description",
  "targates": {
    "android": {"primary": "myapp://product/SKU", "fallback": "https://site/productSKU"},
    "ios":     {"primary": "myapp://product/SKU", "fallback": "https://site/productSKU"},
    "web": "https://site/"
  },
  "params": [{"k1": "v1"}, {"k2": "v2"}],
  "launch_page": {"header": "Please Wait..."},
  "image": "https://site/img/SKU.png",
  "web_push_subscription": false,
  "tag": "product",
  "media_source": "AppShare",
  "FBpixel_tracking": true,
  "twitter_pixel_tracking": true,
  "google_ads_pixel_tracking": true,
  "linkedIn_pixel_tracking": true
}
```

Two things to get right:

- The targets key is spelled **`targates`**. That is Appgain's spelling, not a typo to fix.
- `primary` is the app URI scheme; `fallback` is where the user goes when the app is not
  installed. Omitting `fallback` strands users without the app.

For bulk, `POST /apps/{projectId}/smartlinksbulk?email=<address>&userId=<id>` with a CSV
as multipart `file`.

## 4. Follow the attribution chain

| Stage | System event | Carries |
|---|---|---|
| user opens the link | `smart link open` | SDL URL params, location, header, userId, smartlink id |
| install detected from the link | `smart link matching` | same, plus deferred-deeplink extras |
| app opened via the link | `smart link deeplink opened` | same |
| install from any source | `appInstalled` | `tracked` vs `organic`, smartlink id |
| revenue | `purchase` | amount, currency, `smartlink_id` |

After matching, the user profile carries `smartlink_id`, `reinstall_source`,
`reinstallcount` and `additional_params`. Read a user's events with:

```
GET https://api.appgain.io/user_log_event/{projectId}/{userId}?project_sub_domain={subdomain}&appApiKey=<key>
```

Note this operation accepts the key as a **query parameter**. Prefer the header where
both are accepted — a key in a URL ends up in proxy and gateway logs.

## 5. Verify before you scale

Appgain publishes a manual end-to-end procedure for deferred deep-link attribution at
<https://docs.appgain.io/mobileattribution/HowToTest/>. There is no automated fixture or
simulator, so run it on a device before trusting campaign attribution numbers.
