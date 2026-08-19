---
name: ematic-solutions-esp-list-subscriptions
description: >-
  Add or remove an email address on the ESP list connected to an Ematic Solutions account, through the
  Core API, with the consent obligation Ematic places on the caller.
api: Ematic Solutions Core API
api_reference: https://kb.ematicsolutions.com/restful-api/core-api.html
operations:
  - createSubscription
  - cancelSubscription
generated: '2026-08-13'
method: generated
source: https://kb.ematicsolutions.com/restful-api/core-api.html
---

# Managing ESP list subscriptions through the Ematic Core API

## Before anything else: consent is your obligation

Ematic's developer guide states plainly that adding or removing email addresses from an ESP list
**without the user's consent is prohibited**. Do not wire these two operations to an autonomous agent
loop. Both write to a third-party marketing system and one of them causes real email to be sent to a
real person.

## Setup

Resolve the host from your Ematic API key suffix and authenticate with both keys:

```
POST https://<suffix>-api.ematicsolutions.com:8084/v2/subscribe
Authorization: ematic-apikey=<ematic-key>,esp-apikey=<esp-key>
Content-Type: application/json
```

## Subscribe — `createSubscription`, `POST /subscribe`

Only email subscriptions are supported. `subscribe_email_addr` carries the address to subscribe; it
does not have to match the identifying `email`, but Ematic recommends they match.

```json
{
  "uuid": "11279900",
  "email": "user@example.com",
  "subscribe_email_addr": "user@example.com",
  "first_name": "John",
  "last_name": "Smith",
  "clientEnv": {
    "country": "SG", "language": "en-SG", "currency": "SGD",
    "device_os_family": "Windows", "device_os_version": "10"
  }
}
```

`first_name` and `last_name` are optional.

| Status | Meaning | Do |
|---|---|---|
| `200` | Subscription already exists | Treat as a no-op success — **not** as a create |
| `201` | Subscribed | The body may carry a `coupon` field; surface it if your flow promises one |
| `400` | Bad request | Check `subscribe_email_addr` and that `uuid` or `email` is present |
| `401` | `invalid apikeys` | Fix the Authorization header |
| `500` | ESP rejected it | Read `esp_code` and `mandrill_code` against your ESP's own error reference |

## Unsubscribe — `cancelSubscription`, `DELETE /subscribe`

Same envelope, but the address goes in `unsubscribe_email_addr`. Names are not needed.

```json
{
  "uuid": "11279900",
  "email": "user@example.com",
  "unsubscribe_email_addr": "user@example.com"
}
```

| Status | Meaning | Do |
|---|---|---|
| `200` | User is not subscribed | No-op success |
| `204` | Unsubscribed | Done — there is no response body |
| `400` | Bad request | Check the body |
| `401` | `invalid apikeys` | Fix the Authorization header |
| `500` | ESP error | Read `esp_code` |

## Retry carefully

There is no idempotency key on either operation. A `500` leaves you unable to tell whether the ESP
applied the change, so re-read state in the ESP rather than blind-retrying, especially on unsubscribe
— a failed unsubscribe that you do not resolve is a compliance problem, not just a bug.
