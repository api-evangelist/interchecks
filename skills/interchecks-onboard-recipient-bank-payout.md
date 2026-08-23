---
name: interchecks-onboard-recipient-bank-payout
description: >-
  Onboard a new payee to Interchecks and pay them by ACH — create the recipient, link a bank
  account (Plaid widget or processor token), then create an ACH transaction and settle its
  outcome from the webhook. Use when a payout must reach a US bank account.
api: Interchecks Payments API v2
generated: '2026-08-23'
method: generated
source: openapi/interchecks-payments-api-v2.json
operations:
  - get-access-token
  - create-recipient
  - create-bank-widget
  - get-bank-accounts
  - create-transaction
  - get-transaction
  - update-transaction
---

# Onboard a recipient and pay them by ACH

Base URL `https://prod.api.interchecks.io` (sandbox `https://test.api.interchecks.io`).
Every business path is `/api/v2/{payer_id}/...`; `payer_id` is your aggregator or payer id.

## 1. Get a token — `get-access-token`

`POST /api/v2/oauth2/token` with `Authorization: Basic base64(clientId:secret)` and
`grant_type=client_credentials` (form-encoded). The response `access_token` is a JWT valid for
`expires_in` **900 seconds**. Send it as `Authorization: Bearer <access_token>` on everything
below and refresh before it expires — there is no refresh token.

## 2. Create the recipient — `create-recipient`

`POST /api/v2/{payer_id}/recipients`. Supply `first_name`/`last_name` (or `company_name`),
`email`, and your own `reference_id`. Keep the `reference_id`: it is the only way to find this
recipient again without storing `recipient_id`, via `get-recipient-by-reference-id`.

A recipient cannot be deleted through the API. Do not create speculative recipients.

## 3. Link a bank account

Two paths, pick one:

- **Hosted widget** — `create-bank-widget` (`POST /api/v2/{payer_id}/widgets/accounts/bank`)
  returns a short-lived URL (valid **59 minutes**) to embed in an iFrame. Pass `widget_params`
  with `redirect_url`; the bank widget signals completion by redirect only (it appends
  `widgetId`, `accountId`, and `errorCode`), not by `postMessage`.
- **Plaid processor token** — if you already run Plaid Link yourself, hand the processor token
  straight to `POST /api/v2/{payer_id}/accounts/{recipient_id}/banks/plaid-processor-token`.

Either way, wait for the `PAYMENT_ACCOUNT` webhook before you assume an account exists. Then
read it back with `get-bank-accounts` to get `account_id`.

## 4. Create the transaction — `create-transaction`

`POST /api/v2/{payer_id}/transactions` with `recipient_id`, `account_id`, `type: CREDIT`,
`method`, `amount`, and your `reference_id`.

Pick the method deliberately — this decides whether you can undo it:

| method | speed | reversible? |
|---|---|---|
| `ACH_SAME_DAY` | same day if submitted before 2:45PM Eastern on a business day | no cancel; only a later `ACH_REFUND` |
| `ACH_STANDARD` | 3-5 business days | no cancel; only a later `ACH_REFUND` |
| `RTP` | up to 30 seconds round trip | **no** — irrevocable |

**Always send an `Idempotency-Key` header.** A unique caller-generated value. If the same key is
replayed you get `409` (already processed) or `102` (still processing). On `102`, wait and poll —
do **not** re-submit with a fresh key, or you will pay twice.

## 5. Settle the outcome

The initial response is `PROCESSING`, not final. The `TRANSACTION` webhook carries the terminal
status. Verify it first (see `interchecks-verify-webhook`), then act on
`transaction_status`: `PAID`, `FAILED`, `RETRY`, `CANCELLED`, `PENDING_KYC`, `APPROVAL_REQUIRED`.

If you must poll instead, use `get-transaction` or `get-transaction-by-reference-id`.

## 6. If it fails

`FAILED` webhooks carry `error_code` and `error_message`. ACH returns arrive as NACHA reason
codes — `R01` insufficient funds, `R02` account closed, `R03` no account (also surfaced as
`ERR_BANK_ACCOUNT_R03`). See `errors/interchecks-decline-codes.yml` for all 22 with their return
timeframes; a return can arrive up to **60 calendar days** later for R05/R07/R10/R11/R51, so an
ACH payout is not truly final on the day it settles.

Bank status can also change out from under you: watch for `PLAID_REAUTH_REQUIRED` and
`PLAID_PERMISSION_REVOKED` on the `PAYMENT_ACCOUNT` webhook and re-run the widget before the
next payout.

## Cancelling

Only `ACH_FUNDING_PLUS` (a pull, not a payout) can be cancelled, via `update-transaction` to
`CANCELLED`, and only while `PROCESSING` or `RETRY`. Once sent to the financial institution you
get `422`. Outbound ACH credits have **no cancel operation** — treat step 4 as the point of no
return.
