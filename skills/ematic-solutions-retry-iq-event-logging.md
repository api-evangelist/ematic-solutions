---
name: ematic-solutions-retry-iq-event-logging
description: >-
  Record e-commerce product events (browse, cart, checkout, conversion) into Ematic Solutions so
  Retry-iQ can act on them, using the Core API from a server rather than the Ematic.js browser tracker.
api: Ematic Solutions Core API
api_reference: https://kb.ematicsolutions.com/restful-api/core-api.html
operations:
  - logBrowse
  - logCart
  - logCheckout
  - logConversion
  - logBatchMode
generated: '2026-08-13'
method: generated
source: https://kb.ematicsolutions.com/restful-api/core-api.html
---

# Logging product events to the Ematic Core API

Every operationId and path below is taken verbatim from Ematic's published Core API reference.
Ematic publishes no OpenAPI document, so validate against the reference page before shipping.

## 1. Resolve your host first — there is no global base URL

Ematic shards client accounts across data centers. Your Ematic API key has the form
`<hash>-<suffix>`; the suffix names your data center.

```
base = https://<suffix>-api.ematicsolutions.com:8084/v2
# example, for a key ending "-sg1":
base = https://sg1-api.ematicsolutions.com:8084/v2
```

Sending to the wrong shard will not silently work. Note the non-standard port **8084** — confirm your
egress firewall allows it before you debug anything else.

## 2. Authenticate with BOTH keys in one header

```
Authorization: ematic-apikey=<ematic-key>,esp-apikey=<esp-key>
Content-Type: application/json
Accept: application/json
```

Omitting either key returns `401 {"message":"missing apikeys"}`. A bad key returns
`401 {"message":"invalid apikeys"}`.

## 3. Identify the user

At least one of `uuid` (your own identifier for the shopper) or `email` must be present or the request
fails with `400`. Send both whenever you have both.

## 4. Send the event

| Event | Operation | Path |
|---|---|---|
| Product page viewed | `logBrowse` | `POST /log/browse` |
| Cart added/removed/quantity changed | `logCart` | `POST /log/cart` |
| Checkout page opened | `logCheckout` | `POST /log/checkout` |
| Order completed | `logConversion` | `POST /log/conversion` |
| Many events at once | `logBatchMode` | `POST /log/batch` |

```json
{
  "uuid": "11279900",
  "email": "user@example.com",
  "clientEnv": {
    "deviceHeight": 0, "deviceWidth": 0,
    "viewportHeight": 0, "viewportWidth": 0,
    "country": "SG", "language": "en-SG", "currency": "SGD",
    "device_uid": "", "device_model": "",
    "device_os_family": "Windows", "device_os_version": "10"
  },
  "products": [
    {
      "experiment_id": 0,
      "client_product_id": "223",
      "client_category_id": "1",
      "price_number": 20.5,
      "price": "$20.50",
      "name": "T-Shirt",
      "description": "Men's T-Shirt",
      "brand_name": "Acme",
      "image_url": "http://www.mywebstore.com/images/products/product_223.jpg",
      "link": "https://www.mywebstore.com/products/223",
      "misc1": "Blue", "misc2": "XL", "misc3": ""
    }
  ]
}
```

`experiment_id` **must be 0**. `description` is capped at 255 characters. `misc1`–`misc3` are free
slots for your own attributes.

## 5. Cart logs are snapshots, not deltas

This is the rule people get wrong. On every cart change send the **entire** cart:

```
add A            -> products = [A]
add B            -> products = [A, B]
add C            -> products = [A, B, C]
remove B         -> products = [A, C]
```

## 6. Handle the responses

| Status | Meaning | Do |
|---|---|---|
| `201` | Event recorded | Continue |
| `400` | Bad request | Fix the body — usually a missing `uuid`/`email` or a missing required product field |
| `401` | `missing apikeys` / `invalid apikeys` | Fix the Authorization header, confirm you are on the right shard |

## 7. What this API does not give you

There is **no idempotency key**, so a retried POST logs the event twice — dedupe on your side before
resending. There are **no published rate limits** and no `Retry-After` header, so back off
conservatively on your own schedule. There is no read operation: you cannot query back what you sent.
