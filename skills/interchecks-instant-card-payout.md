---
name: interchecks-instant-card-payout
description: >-
  Push funds to a recipient's debit card in real time using Interchecks INSTANT_DEPOSIT (Visa/
  Mastercard OCT) — capture the card without taking on PCI scope, verify it, send the payout and
  read the network action code when it declines.
api: Interchecks Payments API v2
generated: '2026-08-23'
method: generated
source: openapi/interchecks-payments-api-v2.json
operations:
  - get-access-token
  - create-card-widget
  - add-card-account
  - get-card-accounts
  - verify-card-account
  - get-card-account-verify
  - create-transaction
  - get-transaction
---

# Instant payout to a debit card

## 1. Capture the card without touching it — `create-card-widget`

`POST /api/v2/{payer_id}/widgets/accounts/card` returns a hosted URL (default **15 minutes**).
Embed it in an iFrame or modal. Interchecks offers this specifically so the card number never
reaches your systems, which keeps you out of PCI scope.

Pass `widget_params.target_origin_url` and the child frame will `postMessage`:

```json
{"source": "IC", "type": "CARD", "result": "SUCCESS", "widget_id": "<widget_id>"}
```

Or pass `redirect_url` instead for a plain redirect. The `PAYMENT_ACCOUNT` webhook is the
authoritative signal; it carries the masked `card_number` and the `account_id`.

If you are already PCI compliant you can call `add-card-account`
(`POST /api/v2/{payer_id}/accounts/{recipient_id}/cards`) with the raw PAN instead.

## 2. Verify the card — `verify-card-account`

`POST /api/v2/{payer_id}/accounts/{recipient_id}/cards/{account_id}/verification` checks name,
address and CVV against the issuer. Read the result with `get-card-account-verify`, or the
history with `get-card-account-verifications`. The result object carries `network`, `pan`, `cvv`,
`avs.street`, `avs.zip`, `ani.first_name`, `ani.last_name` — each a match result, not a boolean.

Depending on client configuration this validation may run automatically when the card is added.

## 3. Send the payout — `create-transaction`

`POST /api/v2/{payer_id}/transactions`:

```json
{
  "recipient_id": "...",
  "account_id": "...",
  "type": "CREDIT",
  "method": "INSTANT_DEPOSIT",
  "amount": 125.00,
  "reference_id": "your-id"
}
```

Send an `Idempotency-Key` header. `409` means already processed; `102` means the identical
request is still in flight — wait, do not re-key.

## 4. Read the result carefully

**A decline comes back as HTTP 200.** The call succeeded; the authorization did not:

```json
{"http_status": 200, "error_code": "ERR_OCT_DECLINE", "error_message": "15:No such issuer"}
```

`error_message` is `<action code>:<message>`. The action code is the Visa/Mastercard network
response code — 67 are published, and codes **76-89 are private-use and mean different things on
Visa and on Mastercard**, so branch on `network` before interpreting those. Full table in
`errors/interchecks-decline-codes.yml`.

A successful transaction returns `network_approval_code`.

`UNKNOWN` is a real terminal-ish status here, not an error: on a Visa timeout or "in process"
workflow the transaction is neither confirmed nor failed. You only receive it if your payer
account is configured to allow `UNKNOWN`; otherwise it is reported as `PAID` until the final
status is known. Do not retry an `UNKNOWN` — you will double-pay. Wait for the webhook.

## 5. There is no undo

`INSTANT_DEPOSIT` has **no documented reversal operation**. Once approved, the money is gone.
Validate the amount and recipient before step 3. (The inbound sibling `INSTANT_FUNDING` — an AFT
pull — *can* be reversed via `update-transaction` while `PAID`; the outbound OCT cannot.)

## Testing

Sandbox test BINs are `451530` and `531260`; anything else returns `ERR_CARD_INELIGIBLE`, and a
BIN not starting with 4 or 5 returns `ERR_INVALID_CARD_DETAILS`. Never use a live debit card in
sandbox. Amount `$9.02` forces a decline, `$9.04` a payer NSF, `$9.05` a card velocity error,
`$9.06`/`$9.07` an `UNKNOWN`. Full list in `sandbox/interchecks-sandbox.yml`.
