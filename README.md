# Airtime API — Developer & Integration Guide

**OAuth 2.0 · Version 1.0**

A RESTful HTTP API for purchasing and dispatching airtime to mobile numbers.
This guide is intended for developers and technical teams at
enterprise clients who are integrating the Airtime API into their applications,
platforms, or business workflows.

---

## Document Information

| Field              | Value                                              |
| ------------------ | -------------------------------------------------- |
| API Name           | Airtime API                                        |
| Authentication     | OAuth 2.0 (Password Grant and/or Client Credentials) |
| Version            | v1                                                 |
| Supported Formats  | JSON                                               |
| Transport          | HTTPS only (TLS 1.2+)                              |
| Owner              | Implementation Team                                |
| Last Updated       | See [Changelog](#changelog)                        |

---

## Table of Contents

1. [Overview](#1-overview)
2. [Getting Started](#2-getting-started)
3. [Base URL & Environments](#3-base-url--environments)
4. [Authentication](#4-authentication)
   - [4.1 Obtain an Access Token (Password Grant)](#41-obtain-an-access-token-password-grant)
   - [4.2 Obtain an Access Token (Client Credentials)](#42-obtain-an-access-token-client-credentials)
   - [4.3 Refresh an Access Token](#43-refresh-an-access-token)
   - [4.4 Using the Access Token](#44-using-the-access-token)
5. [API Reference](#5-api-reference)
   - [5.1 Send Airtime](#51-send-airtime)
6. [Status & Response Codes](#6-status--response-codes)
7. [Error Handling](#7-error-handling)
8. [Rate Limits & Quotas](#8-rate-limits--quotas)
9. [Code Samples](#9-code-samples)
10. [Best Practices](#10-best-practices)
11. [Frequently Asked Questions](#11-frequently-asked-questions)
12. [Glossary](#12-glossary)
13. [Changelog](#13-changelog)
14. [Support](#14-support)

---

## 1. Overview

The Airtime API lets your systems programmatically top up mobile numbers (airtime and
related products where provisioned) through a secure, token-authenticated HTTPS
interface. Typical use cases include staff airtime allowances, customer reward top-ups,
agent float distribution, and automated operational top-ups.

**Key characteristics**

- **RESTful** — predictable, resource-oriented endpoints over HTTPS.
- **JSON** — all request and response bodies are JSON.
- **OAuth 2.0** — access is authorized with short-lived bearer tokens.
- **Stateless** — each request is authenticated independently via the `Authorization`
  header.
- **Float-based** — successful requests debit your institutional float balance.

**Integration flow at a glance**

1. Retrieve your API credentials from the developer portal / account manager.
2. Exchange those credentials for an **access token** (Section 4).
3. Call the **Send Airtime** endpoint with the access token in the `Authorization` header
   (Section 5.1).
4. When the token expires, obtain a new one (refresh or re-authenticate).

**Internal processing steps (informational)**

A successful call typically runs this pipeline:

1. Validate institution  
2. Resolve gateway profile  
3. Resolve MNO from MSISDN  
4. Debit institutional float  
5. Dispatch to upstream biller / telco gateway  

Drive integration logic off `response_status` / `overall_status`, not off the human-readable
strings inside the `response` object.

---

## 2. Getting Started

Before making your first request, ensure you have completed the following:

| # | Prerequisite |
| - | ------------ |
| 1 | An active account provisioned by your account manager. |
| 2 | OAuth 2.0 credentials — at minimum `client_id` (and `client_secret` if using client credentials). |
| 3 | Account username and password **or** client credentials, depending on grant type. |
| 4 | A gateway profile linked to a valid **institution**. |
| 5 | A funded institutional **float** balance sufficient to cover the top-ups you intend to send. |
| 6 | Channel ID (`chid`) issued for your integration. |

> **Where to find credentials:** Log in to the developer portal and open
> **Developer → Settings**, or obtain them from your account manager. Keep these values
> confidential.

---

## 3. Base URL & Environments

All API endpoints are relative to the base URL for your environment. Always use HTTPS;
plain HTTP requests are rejected.

| Environment | Base URL (example) |
| ----------- | ------------------ |
| Production  | `https://nenasasa.com/api/v1/` |
| Staging     | Provided by your account manager (e.g. host-specific staging URL) |

> Replace the base URL with the environment-specific host issued to you if you have a
> dedicated or sandbox endpoint.

---

## 4. Authentication

The API uses **OAuth 2.0**. Supported grant types depend on how your application was
provisioned:

| Grant | Typical use |
| ----- | ----------- |
| `password` | Resource owner (username + password) |
| `client_credentials` | Server-to-server / machine clients |
| `refresh_token` | Renew access without full re-login (where issued) |

| Token | Purpose | Typical Lifetime |
| --------------- | --------------------------------------------------- | --------------------- |
| `access_token` | Presented as a Bearer token to authorize API calls. | Often 36,000 s (10 hours); always use `expires_in` from the response |
| `refresh_token` | Used to obtain a new access token (when issued). | Provider-defined |

> **Security:** Treat all tokens and credentials as secrets. Store them server-side,
> never embed them in client-side code, mobile apps, or public repositories, and always
> transmit them over TLS.

### 4.1 Obtain an Access Token (Password Grant)

**Endpoint**

```
POST /o/token/
```

Full URL example: `https://nenasasa.com/api/v1/o/token/`

**Request Headers**

| Name | Value | Required |
| -------------- | ------------------ | -------- |
| `Content-Type` | `application/json` | Yes |

**Request Body Parameters**

| Name | Type | Required | Description |
| ------------ | ------ | -------- | ------------------------------------------------------------- |
| `client_id` | String | Yes | OAuth 2.0 client identifier. |
| `username` | String | Yes | Your account username. |
| `password` | String | Yes | Your account password. |
| `grant_type` | String | Yes | Must be `password`. |

**Sample Request**

```json
{
  "client_id": "your_client_id",
  "username": "your_username",
  "password": "your_password",
  "grant_type": "password"
}
```

**Sample Response** — `200 OK`

```json
{
  "access_token": "9U0XnjLEmP7vvl2JRJb0u3umSGu6rK",
  "expires_in": 36000,
  "token_type": "Bearer",
  "scope": "read write groups",
  "refresh_token": "PxXEeFTh87phN5SseDQMJ9mGFL83YA"
}
```

### 4.2 Obtain an Access Token (Client Credentials)

Use this when your application was issued a `client_id` and `client_secret` for
server-to-server access.

**Request Body Parameters**

| Name | Type | Required | Description |
| --------------- | ------ | -------- | -------------------------------- |
| `client_id` | String | Yes | OAuth 2.0 client identifier. |
| `client_secret` | String | Yes | OAuth 2.0 client secret. |
| `grant_type` | String | Yes | Must be `client_credentials`. |

**Sample Request**

```json
{
  "client_id": "your_client_id",
  "client_secret": "your_client_secret",
  "grant_type": "client_credentials"
}
```

**Sample Response** — `200 OK`

```json
{
  "access_token": "2wICE4dU4aAozW9Sy9KuwZczDpLqcE",
  "expires_in": 36000,
  "token_type": "Bearer",
  "scope": "read write groups"
}
```

> Note: Client-credentials tokens may not include a `refresh_token`. Request a new token
> when `expires_in` elapses.

### 4.3 Refresh an Access Token

When a refresh token was issued with the password grant, you can renew the access token
without re-submitting the password.

**Request Body Parameters**

| Name | Type | Required | Description |
| --------------- | ------ | -------- | ----------------------------------------------- |
| `client_id` | String | Yes | OAuth 2.0 client identifier. |
| `grant_type` | String | Yes | Must be `refresh_token`. |
| `refresh_token` | String | Yes | The refresh token from the previous response. |

**Sample Request**

```json
{
  "client_id": "your_client_id",
  "grant_type": "refresh_token",
  "refresh_token": "PxXEeFTh87phN5SseDQMJ9mGFL83YA"
}
```

### 4.4 Using the Access Token

Include the access token as a Bearer token in the `Authorization` header of every
authenticated request:

```
Authorization: Bearer <access_token>
```

Example:

```
Authorization: Bearer 9U0XnjLEmP7vvl2JRJb0u3umSGu6rK
```

---

## 5. API Reference

### 5.1 Send Airtime

Purchases and dispatches airtime to a destination MSISDN. The amount is debited from your
institutional float when the debit step succeeds.

**Endpoint**

```
POST /SEND AIRTIME/
```

Full URL example: `https://nenasasa.com/api/v1/SEND AIRTIME/`

> The path segment is literally `SEND AIRTIME` (including the space). If your HTTP client
> requires URL encoding, the space becomes `%20` (i.e. `.../api/v1/SEND%20AIRTIME/`).

**Request Headers**

| Name | Value | Required |
| --------------- | ------------------------ | -------- |
| `Content-Type` | `application/json` | Yes |
| `Authorization` | `Bearer <access_token>` | Yes |

**Request Body Parameters**

| Name | Type | Required | Description |
| ------------------- | ------ | -------- | ------------------------------------------------------------------ |
| `chid` | String | Yes | Channel unique ID identifying the sending channel/profile. |
| `msisdn` | String | Yes | Destination number (international / normalized MSISDN preferred). |
| `amount` | String / Number | Yes | Airtime amount to send (e.g. `"10.00"`). |
| `reference` | String | Recommended | Your unique transaction reference for reconciliation and support. |
| `client_reference` | String | Optional | Alternate client-side reference if used by your integration. |
| `institution_id` | String / Number | Conditional | Required only if your gateway profile does not already have an institution bound. When the profile has an institution, the API validates or captures it automatically. |

**Sample Request**

```json
{
  "chid": "13",
  "msisdn": "2547XXXXXXXX",
  "amount": "10.00",
  "reference": "AIRTIME-2026-0001",
  "institution_id": "765"
}
```

**Response Body Parameters**

| Name | Type | Description |
| ----------------- | ------ | ---------------------------------------------------------------- |
| `response` | Object | Per-step processing detail (informational). |
| `action_id` | Number | Identifier for the last executed action/step. |
| `response_status` | String | Application status for the request. `00` indicates success. |
| `overall_status` | String | Overall status for the operation. `00` indicates success. |
| `last_response` | String | Human-readable summary of the outcome. |
| `response_message` | String / null | Optional machine-oriented message (may be present). |

**Typical `response` object keys** (informational; may vary by configuration)

| Name | Description |
| ---------------------- | ------------------------------------------------------- |
| `validate_institution` | Institution validated or captured. |
| `get_gateway_profile` | Gateway profile resolved. |
| `check_mno` | Mobile network resolved from MSISDN. |
| `debit_float` | Float debit result (amount / balance messaging). |
| `biller` | Upstream biller / telco dispatch result. |

**Sample Response** — success path (`200 OK`, application `00` or async `09`)

```json
{
  "response": {
    "validate_institution": "Institution Validated",
    "get_gateway_profile": "Got Gateway Profile",
    "check_mno": "Safaricom subscriber verified successfully.",
    "debit_float": "Float Debited with: 10.00 balance: 99990.00",
    "biller": "Async Biller Submitted: ok"
  },
  "action_id": 133,
  "response_status": "09",
  "overall_status": "09",
  "last_response": "Async Biller Submitted: ok",
  "response_message": "safaricom_mno"
}
```

> **Note:** `09` indicates the request was accepted for **asynchronous** upstream
> processing. `00` indicates synchronous success where the product is configured for
> realtime. Always treat both as accepted outcomes unless your account manager specifies
> otherwise. Drive automation off `response_status` / `overall_status`, not off free-text
> strings inside `response`.

---

## 6. Status & Response Codes

The API communicates outcomes at two levels: the **HTTP status code** of the response,
and the **application status codes** (`response_status` / `overall_status`) in the JSON
body.

**HTTP status codes**

| Code | Meaning | Notes |
| ----- | ---------------------- | ------------------------------------------------------------ |
| `200` | OK | Request processed. Inspect the body for application status. |
| `400` | Bad Request | Malformed JSON or missing/invalid parameters. |
| `401` | Unauthorized | Missing, invalid, or expired access token. |
| `403` | Forbidden | Token lacks required scope or account not permitted. |
| `429` | Too Many Requests | Rate limit exceeded; retry after backoff. |
| `500` | Internal Server Error | Unexpected server error; retry later or contact support. |
| `503` | Service Unavailable | Temporary outage or maintenance; retry with backoff. |

**Application status codes** (`response_status`, `overall_status`)

| Code | Meaning | Typical cause / action |
| ------ | -------- | ----------------------------------------------------------- |
| `00` | Success | Request processed successfully (including realtime biller success). |
| `09` | Accepted (async) | Submitted for asynchronous upstream processing. |
| `03` | Institution mismatch | `institution_id` does not match the profile institution. |
| `25` | Not found / missing config | Profile institution missing, product/item not found, etc. |
| `30` | Invalid MSISDN / MNO | Number could not be validated or mapped to a network. |
| `51` | Insufficient float | Top up institutional float and retry. |
| `92` | Product not found | No biller product configured for this service / filters. |
| `96` | System / processing error | Inspect `last_response`; contact support if persistent. |
| Other non-`00` | Failure | Inspect `last_response` and correct configuration or payload. |

> Because a `200 OK` can still carry a non-`00` application status (for example,
> insufficient float), always check both the HTTP code **and** the body status.

---

## 7. Error Handling

Build your integration to handle failures gracefully.

| Scenario | Signal | Recommended handling |
| ---------------------------- | ---------------------------------------- | ----------------------------------------------------------------- |
| Expired / invalid token | HTTP `401` | Refresh or re-authenticate, then retry once. |
| Missing / invalid parameters | HTTP `400`, non-`00` status | Do not retry blindly; fix the payload first. |
| Profile has no institution | `25`, institution message | Have the gateway profile linked to an institution, or pass a valid `institution_id` only if your account allows it. |
| Insufficient float | Non-`00` (e.g. `51`), float message | Top up the institutional float, then resend. |
| Product / biller not found | `92` | Confirm service and product configuration with the implementation team. |
| Rate limited | HTTP `429` | Exponential backoff and retry. |
| Server / gateway error | HTTP `5xx` or `96` | Retry with backoff; alert support if persistent. |

**Recommended retry strategy**

- Retry only on transient failures (`429`, `5xx`, network timeouts, selected `96` cases).
- Use **exponential backoff with jitter** and cap the number of attempts.
- Make requests **idempotent** on your side using a stable `reference` / `client_reference`
  so retries do not create unintended duplicate top-ups.
- Do not retry on `400` / `403` / permanent `25` / `92` without correcting the request or
  configuration.

---

## 8. Rate Limits & Quotas

Formal per-second rate limits are provisioned per account and may be adjusted for your
volume tier. Design defensively:

- Respect `429 Too Many Requests` and back off before retrying.
- For large volumes, queue and spread requests on your side.
- Monitor institutional **float balance**; depleted float causes failures even when you
  are within rate limits.

> Contact your account manager to confirm throughput and quotas for your account.

---

## 9. Code Samples

Replace placeholder values with your credentials, channel ID, institution (if required),
and recipient data.

### 9.1 cURL

**Get an access token (password grant)**

```bash
curl -X POST "https://nenasasa.com/api/v1/o/token/" \
  -H "Content-Type: application/json" \
  -d '{
    "client_id": "your_client_id",
    "username": "your_username",
    "password": "your_password",
    "grant_type": "password"
  }'
```

**Get an access token (client credentials)**

```bash
curl -X POST "https://nenasasa.com/api/v1/o/token/" \
  -H "Content-Type: application/json" \
  -d '{
    "client_id": "your_client_id",
    "client_secret": "your_client_secret",
    "grant_type": "client_credentials"
  }'
```

**Send airtime**

```bash
curl -X POST "https://nenasasa.com/api/v1/SEND%20AIRTIME/" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -d '{
    "chid": "13",
    "msisdn": "2547XXXXXXXX",
    "amount": "10.00",
    "reference": "AIRTIME-2026-0001",
    "institution_id": "765"
  }'
```

### 9.2 Python (`requests`)

```python
import requests

BASE_URL = "https://nenasasa.com/api/v1"


def get_access_token():
    resp = requests.post(
        f"{BASE_URL}/o/token/",
        json={
            "client_id": "your_client_id",
            "username": "your_username",
            "password": "your_password",
            "grant_type": "password",
        },
        timeout=30,
    )
    resp.raise_for_status()
    return resp.json()["access_token"]


def send_airtime(access_token, msisdn, amount, reference, institution_id=None):
    body = {
        "chid": "13",
        "msisdn": msisdn,
        "amount": str(amount),
        "reference": reference,
    }
    if institution_id is not None:
        body["institution_id"] = str(institution_id)

    resp = requests.post(
        f"{BASE_URL}/SEND%20AIRTIME/",
        headers={"Authorization": f"Bearer {access_token}"},
        json=body,
        timeout=60,
    )
    resp.raise_for_status()
    return resp.json()


if __name__ == "__main__":
    token = get_access_token()
    result = send_airtime(
        token,
        msisdn="2547XXXXXXXX",
        amount="10.00",
        reference="AIRTIME-2026-0001",
        institution_id="765",
    )
    print(result.get("response_status"), result.get("last_response"))
```

### 9.3 Node.js (`fetch`)

```javascript
const BASE_URL = "https://nenasasa.com/api/v1";

async function getAccessToken() {
  const res = await fetch(`${BASE_URL}/o/token/`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      client_id: "your_client_id",
      username: "your_username",
      password: "your_password",
      grant_type: "password",
    }),
  });
  if (!res.ok) throw new Error(`Auth failed: ${res.status}`);
  const data = await res.json();
  return data.access_token;
}

async function sendAirtime(accessToken, { msisdn, amount, reference, institutionId }) {
  const body = {
    chid: "13",
    msisdn,
    amount: String(amount),
    reference,
  };
  if (institutionId != null) body.institution_id = String(institutionId);

  const res = await fetch(`${BASE_URL}/SEND%20AIRTIME/`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${accessToken}`,
    },
    body: JSON.stringify(body),
  });
  if (!res.ok) throw new Error(`Send failed: ${res.status}`);
  return res.json();
}

(async () => {
  const token = await getAccessToken();
  const result = await sendAirtime(token, {
    msisdn: "2547XXXXXXXX",
    amount: "10.00",
    reference: "AIRTIME-2026-0001",
    institutionId: "765",
  });
  console.log(result.response_status, result.last_response);
})();
```

### 9.4 PHP (cURL)

```php
<?php
$baseUrl = "https://nenasasa.com/api/v1";

function post_json($url, $payload, $headers = []) {
    $ch = curl_init($url);
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode($payload),
        CURLOPT_HTTPHEADER => array_merge(["Content-Type: application/json"], $headers),
    ]);
    $response = curl_exec($ch);
    curl_close($ch);
    return json_decode($response, true);
}

// 1. Get token
$auth = post_json("$baseUrl/o/token/", [
    "client_id" => "your_client_id",
    "username" => "your_username",
    "password" => "your_password",
    "grant_type" => "password",
]);
$token = $auth["access_token"];

// 2. Send airtime
$result = post_json("$baseUrl/SEND%20AIRTIME/", [
    "chid" => "13",
    "msisdn" => "2547XXXXXXXX",
    "amount" => "10.00",
    "reference" => "AIRTIME-2026-0001",
    "institution_id" => "765",
], ["Authorization: Bearer $token"]);

echo $result["response_status"] . " " . $result["last_response"];
```

---

## 10. Best Practices

- **Cache and reuse tokens.** Reuse a valid access token until near expiry; then refresh
  or re-authenticate.
- **Protect credentials.** Store `client_id`, secrets, passwords, and tokens in a secure
  server-side store. Never ship them in client-side apps or public repos.
- **Use consistent MSISDN formatting.** Prefer international format (e.g. `2547…`) for
  reliable MNO resolution.
- **Send a unique `reference`.** Use it for idempotency, reconciliation, and support.
- **Ensure institution binding.** Prefer gateway profiles that already have an
  institution; only rely on `institution_id` in the body if that is how your account is
  configured.
- **Fund float before bulk runs.** Monitor balance; top up before campaigns.
- **Handle both status layers.** Check HTTP status and body `response_status` /
  `overall_status`.
- **Treat `00` and `09` as accepted** unless your account team defines a stricter rule.
- **Log outcomes.** Persist `reference`, `action_id`, `response_status`, and
  `last_response` for audit and support.

---

## 11. Frequently Asked Questions

**How long is an access token valid?**  
Use the `expires_in` value from the token response. Do not hard-code lifetimes.

**Do I need a new token for every top-up?**  
No. Reuse a valid token across many requests.

**What does `response_status` `00` mean?**  
Success for the processed path (including realtime upstream success where configured).

**What does `09` mean?**  
The request was accepted for asynchronous upstream processing.

**I got HTTP `200` but a non-`00` status. Why?**  
The API accepted the HTTP request but the business operation failed (float, product,
institution, MSISDN, etc.). Read `last_response`.

**Do I always need `institution_id`?**  
Only if the gateway profile behind your token has no institution. If the profile already
has one, the API can capture or validate it automatically.

**What format should recipient numbers use?**  
International MSISDN for the destination country (e.g. Kenya `2547…`).

**What is `chid`?**  
Channel unique ID for the sending channel/profile provisioned on your account.

**Why did float debit report “No float amount to debit”?**  
Usually missing amount mapping, float product configuration, or account float setup.
Confirm amount field, institution float, and product configuration with the
implementation team.

---

## 12. Glossary

| Term | Definition |
| --------------- | -------------------------------------------------------------------------- |
| MSISDN | Recipient mobile number, preferably in international format. |
| Float | Prepaid institutional balance debited for successful airtime requests. |
| Access token | Short-lived bearer credential used to authorize API calls. |
| Refresh token | Credential used to obtain a new access token without full re-login (when issued). |
| Institution | Organisation entity bound to the gateway profile / float. |
| `chid` | Channel unique ID identifying the sending channel/profile. |
| Biller | Upstream product/gateway path used to fulfil the airtime request. |
| `reference` | Client-supplied transaction reference for tracking and support. |

---

## 13. Changelog

| Version | Date | Changes |
| ------- | ------------ | -------------------------------------------------- |
| 1.0 | 17 Aug 2026 | Initial Airtime API guide (OAuth 2.0, Send Airtime). |

---

## 14. Support

For credential provisioning, institution/float setup, product configuration, throughput
or quota changes, or integration assistance, contact your account manager or the
Implementation Team through your standard support channel.

When raising an issue, include:

- Environment and base URL  
- Approximate timestamp  
- Request `reference`  
- `response_status` / `overall_status` and `last_response`  
- Whether the call was password grant or client credentials  

---

*This document is provided for integration purposes. Endpoint paths, quotas, product
coverage, and pricing are account-specific; confirm the exact values issued to your
account before going to production.*
