---
name: interchecks-encrypted-payloads
description: >-
  Enable and test Interchecks envelope encryption — AES-GCM payload with RSA-OAEP SHA-256 key
  wrapping, negotiated by the X-ENCRYPTED header — using the sandbox encryption test harness
  before turning it on against real traffic.
api: Interchecks Payments API v2
generated: '2026-08-23'
method: generated
source: https://docs-v2.interchecks.com/docs/encrypted-requests
operations:
  - get-access-token
  - get-encryption-public-key
  - post-payload-encryption-test-encryption
  - post-payload-encryption-test-decryption
  - post-payload-encryption-test-end-to-end
  - encryption-health-check
---

# Envelope-encrypted requests and responses

Optional, per-payer, on top of TLS. A payer or aggregator can be configured so that request and
response bodies for **any** API call are encrypted end to end. Enablement is a configuration
change on the Interchecks side — talk to tech@interchecks.com first; this skill covers proving
your implementation once it is switched on in sandbox.

## The scheme

Envelope encryption. The body is encrypted with AES; the AES key and IV are encrypted with the
counterparty's RSA public key.

- **AES**: `AES/GCM/NoPadding` by default. To use `AES/CBC/PKCS5Padding`, send
  `X-AES-MODE: CBC_PKCS5PADDING`. To be explicit about the default, send
  `X-AES-MODE: GCM_NOPADDING`.
- **RSA**: `RSA/ECB/OAEPPadding` with **OAEP SHA-256 and MGF1 SHA-256**.

> The MGF1 digest is the trap. Java's default crypto provider resolves
> `RSA/ECB/OAEPWITHSHA-256ANDMGF1PADDING` to **SHA-1** MGF1; BouncyCastle uses SHA-256. Specify
> the MGF1 parameters explicitly rather than relying on the algorithm string.

Interchecks' RSA public key is generated with a 4096-bit modulus and is available in PEM or JWK
form from `get-encryption-public-key`
(`GET /api/v2/{payer_id}/payload/encryption/public-key/{format}`).

## Sending

1. Add `X-ENCRYPTED: <Payer ID>`.
2. AES-encrypt the JSON body with an IV; **Base64** the ciphertext.
3. RSA-encrypt the AES key and the IV with the Interchecks public key; **hex-encode** both.
4. POST this envelope:

```json
{
  "encryptedText": "base64 string",
  "key": "hexadecimal string",
  "kid": "string",
  "ivSpec": "hexadecimal string"
}
```

Note the asymmetry: ciphertext is Base64, key and IV are hex. Getting this backwards is the most
common failure.

**GET requests** have no body — the presence of `X-ENCRYPTED` alone causes the *response* to be
encrypted.

If the body does not map to the envelope shape, Interchecks replies `422` in **plaintext** with
the expected format, so a malformed envelope is self-diagnosing.

## Receiving

Interchecks encrypts responses with **your** public RSA key (which you provide during
onboarding), sends the ciphertext Base64 and the key/IV hex, and identifies the key it used with
a `kid`. Decrypt the AES key and IV with your private key or KMS, then decrypt the body.

## Prove it in sandbox first

Three sandbox-only operations exercise each direction independently, so you can isolate a
failure instead of debugging a live payment:

- `post-payload-encryption-test-decryption` — send an encrypted payload, get plaintext back.
  Proves **your encryption** is correct.
- `post-payload-encryption-test-encryption` — send plaintext JSON, get an encrypted response.
  Proves **your decryption** is correct.
- `post-payload-encryption-test-end-to-end` — encrypted in, encrypted out, no business logic.
  Proves the whole round trip.

`encryption-health-check` (`POST /api/v2/{payer_id}/payload/encryption/health`) confirms the
encryption path is live.

Run all three green before enabling encryption on any operation that moves money.
