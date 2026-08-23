---
name: interchecks-verify-webhook
description: >-
  Verify and process an Interchecks webhook — decode the x-verification RS256 JWT, fetch the
  validation key by kid, confirm the SHA-256 body hash and the 5-minute freshness window, then
  dispatch on webhook_type. Use before acting on any Interchecks event.
api: Interchecks Payments API v2
generated: '2026-08-23'
method: generated
source: https://docs-v2.interchecks.com/docs/webhooks
operations:
  - get-access-token
---

# Verify an Interchecks webhook before you trust it

Interchecks is asynchronous by design: the terminal state of every ACH, RTP and card transaction
arrives by webhook, not in the API response. That makes webhook verification a money-safety
control, not a formality.

## 1. Verify before parsing

Every webhook carries `x-verification`, an **RS256 JWT** whose payload is:

```json
{"iat": 1657569050, "request_body_sha256_hash": "B8B0...3666"}
```

1. Decode the JWT header and read `kid`.
2. `GET /api/v2/{payer_id}/webhooks/get_validation_key/{kid}` — returns the RSA public key as a
   JWK (`kty`, `e`, `n`). Cache by `kid`; keys rotate.
3. Verify the JWT signature with that key. **If it fails, drop the webhook.**
4. SHA-256 the raw request body and compare to `request_body_sha256_hash` (case-insensitive).
   **If they differ, drop the webhook.**
5. Reject if `iat` is more than **5 minutes** old.

## 2. Deduplicate

`x-webhook-id` matches the `Aws-Api-Gateway-Requestid` returned by the originating create
request. Use it as the idempotency key on your side, and to correlate the event back to the API
call that caused it.

Interchecks retries a non-2xx **twice within 15 seconds, then repeats the cycle within the next
2 hours** — so you will see duplicates. Return 2xx quickly and process out of band.

## 3. Dispatch on `webhook_type`

**`PAYMENT`** — `payment_id`, `payment_status`, optional `reference_id`.

**`TRANSACTION`** — `transaction_id`, `transaction_status`, `type`, `method`,
`webhook_created_date`, `network_approval_code` (Instant Deposit), optional `reference_id`. The
error variant adds `error_code` and `error_message`. Statuses: `APPROVAL_REQUIRED`, `CANCELLED`,
`FAILED`, `PAID`, `PENDING_KYC`, `PROCESSING`, `RETRY`, `REFUNDED`, `REFUNDED_PARTIAL`,
`REVERSED`, `REVERSAL_PENDING`, `UNKNOWN`.

**`PAYMENT_ACCOUNT`** — account lifecycle, always with `widget_id`, `account_id`, `recipient_id`.
Three things here need action, not logging:

- `account_status: BANK_ACCOUNT_BLOCKED` — blocked by fraud controls. Stop transacting.
- `account_status: PLAID_REAUTH_REQUIRED` — Plaid authorization is about to expire. Re-run the
  bank widget **before** the next payout, not after the failure.
- `account_status: PLAID_PERMISSION_REVOKED` — the user revoked Plaid. A new bank widget
  authorization is required before transacting.

A card account created with verification carries a `verification_result` object
(`network`, `pan`, `cvv`, `avs.street`, `avs.zip`, `ani.first_name`, `ani.last_name`).

## 4. Configuration note

Webhook URLs are set in the Developer area of the Interchecks Portal, at the aggregator or the
payer level. If your API account was created at the aggregator level, webhooks fire against the
**aggregator's** settings — not each payer's.

## 5. Do not treat absence as success

`PROCESSING` with no follow-up webhook is not a payment. Reconcile with `get-transaction` /
`get-transaction-by-reference-id`, and against `get-ach-settlement-report` and
`get-payer-oct-settlement-report` for the day.
