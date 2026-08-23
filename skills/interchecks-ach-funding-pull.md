---
name: interchecks-ach-funding-pull
description: >-
  Pull funds from a customer's bank account with Interchecks ACH Funding Plus — run the
  pre-transaction balance and risk checks, handle the risk-threshold errors, cancel while it is
  still cancellable, and refund it once it is not. Use for deposits, not payouts.
api: Interchecks Payments API v2
generated: '2026-08-23'
method: generated
source: https://docs-v2.interchecks.com/docs/funding-transactions
operations:
  - get-access-token
  - create-bank-widget
  - get-bank-accounts
  - create-transaction
  - get-transaction
  - update-transaction
  - get-ach-settlement-report
---

# Pull funds by ACH (ACH Funding Plus)

This is the DEBIT direction: money moves *from* the recipient's bank account *to* you. It is the
one Interchecks rail with a real cancel window, so the sequencing matters.

## 1. Have a linked bank account

Same as a payout: `create-bank-widget` (Plaid Link, 59-minute URL) or a Plaid processor token.
Read `account_id` from `get-bank-accounts`.

## 2. Create the funding transaction — `create-transaction`

```json
{
  "recipient_id": "...",
  "account_id": "...",
  "type": "DEBIT",
  "method": "ACH_FUNDING_PLUS",
  "amount": 250.00,
  "reference_id": "your-id",
  "funding_options": {
    "user_segment": "returning",
    "ip_address": "203.0.113.10",
    "user_agent": "..."
  }
}
```

`funding_options.ip_address` and `user_agent` feed fraud mitigation — send them. `user_segment`
lets Interchecks apply a cohort-specific risk threshold.

Send an `Idempotency-Key`. `102` = identical request in flight (wait, do not re-key), `409` =
already processed.

Same-day processing requires submission before **2:45PM Eastern** on a business day.

## 3. Handle the pre-transaction checks

Before the pull, Interchecks runs velocity, Plaid balance and risk-score checks. These fail with
`422` and a specific code — each needs a different response, and two of them have documented
bypasses:

| `error_code` | what to do |
|---|---|
| `ERR_ACCOUNT_BALANCE_UNAVAILABLE` | Plaid balance unavailable — you may re-issue with `balance_check: false` |
| `ERR_ACCOUNT_RISK_METRICS_UNAVAILABLE` | Plaid risk data unavailable — you may re-issue with `risk_check: false` |
| `ERR_ACCOUNT_BALANCE_THRESHOLD` | balance below the required threshold — do not bypass, decline |
| `ERR_ACCOUNT_RISK_THRESHOLD` | consumer risk threshold exceeded — decline |
| `ERR_ACCOUNT_RISK_BANK_BALANCE_THRESHOLD` | bank risk exceeded **and** balance multiplier unmet; requirement starts at 200% and rises 100% per `PROCESSING` transaction |
| `ERR_ACCOUNT_AUTH_REQUIRED` / `ERR_ACCOUNT_RE_AUTH_REQUIRED` | send the user back through the bank widget |
| `ERR_ACCOUNT_RISK_LEVEL_NOT_CONFIGURED` / `ERR_PROVIDER_ACCOUNT_MISCONFIGURATION` | your configuration, not the user's — contact tech@interchecks.com |

Risk errors return a `funding_options_response` object with the configured threshold and the
actual value, so you can log why rather than just that.

Bypassing `balance_check` or `risk_check` moves the loss risk onto you. Do it as a deliberate
policy, never as a retry reflex.

## 4. Cancel while you still can — `update-transaction`

`PATCH /api/v2/{payer_id}/transactions/{transaction_id}` with status `CANCELLED`.

**Window: only while the transaction is `PROCESSING` or `RETRY`.** Once it has been sent to the
financial institution, or is otherwise in an immutable status, you get `422`. This is the only
cancel window in the API — after it closes, your only route is a refund.

## 5. Returns and retries

R01 (insufficient funds) returns on `ACH_FUNDING_PLUS` are **automatically retried** at a
configured interval; when retries run out you get `ERR_ACH_RETRY_ATTEMPTS_EXHAUSTED`. Other NACHA
return codes arrive as a `FAILED` webhook. R05, R07, R10, R11 and R51 can arrive up to **60
calendar days** after settlement — a funded account is not unconditionally yours for two months.

## 6. Refund — `create-transaction` with `ACH_REFUND`

To give settled funds back, create a **new** transaction with `type: CREDIT`,
`method: ACH_REFUND`, and `originating_transaction_id` referencing the original. The original
transaction moves to `REFUNDED` or `REFUNDED_PARTIAL`. Same-day if submitted before 2:45PM
Eastern; Interchecks does not publish an outer limit for how long after settlement a refund may
be raised.

## Testing

Sandbox amount triggers: `$9.02` `ERR_ACCOUNT_BALANCE_UNAVAILABLE`, `$9.03`
`ERR_ACCOUNT_BALANCE_THRESHOLD`, `$9.04` `ERR_ACCOUNT_RISK_METRICS_UNAVAILABLE`, `$9.05`
`ERR_ACCOUNT_RISK_THRESHOLD`, `$9.06` `ERR_ACCOUNT_AUTH_REQUIRED`, `$9.07`
`ERR_ACCOUNT_RE_AUTH_REQUIRED`, `$9.01` a 503. Use `create-account-plaid` in the test harness to
auto-link a bank account without the Plaid Link UI.
