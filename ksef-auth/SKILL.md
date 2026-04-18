---
name: ksef-auth
description: Use when authenticating with KSeF API 2.0 (Polish e-invoice system) using a KSeF token or company certificate (XAdES-BES), or when accessToken is missing/expired and needs refreshing.
---

# KSeF — Uwierzytelnianie (API 2.0)

## Overview

KSeF API 2.0 uses JWT-based authentication. Two methods are supported:
1. **KSeF Token** — for test environment and token-based prod access
2. **Certificate (XAdES-BES)** — for production environment using a company certificate

Implementation: `/root/ksef/ksef_client.py` → `KSeFClient.authorise()`, `authorise_cert()`, `refresh_access_token()`

## Method 1: KSeF Token

```
POST /auth/challenge
  → challenge, timestampMs

encrypt("{token}|{timestampMs}") with RSA-OAEP SHA-256 (KsefTokenEncryption cert)

POST /auth/ksef-token
  body: { challenge, contextIdentifier: {type:"Nip", value:nip}, encryptedToken }
  → authenticationToken.token, referenceNumber

poll GET /auth/{referenceNumber}  (bearer: authToken)
  status.code: 100=pending, 200=ok, 4xx/5xx=error
  wait 2s between polls, max 20 attempts

POST /auth/token/redeem  (bearer: authToken, empty body)
  → accessToken.token, refreshToken.token
```

## Method 2: Certificate (XAdES-BES) — Production

Used for production auth with a company certificate (`.pem`/`.crt` + `.key`).

```
POST /auth/challenge
  → challenge (UUID string), timestampMs

Build XAdES-BES XML signature:
  - SignedInfo contains: challenge, nip, timestampMs
  - Signed with company private key
  - Certificate included in KeyInfo

POST /auth/cert
  body: { challenge, contextIdentifier: {type:"Nip", value:nip}, signature: "<XAdES XML>" }
  → authenticationToken.token, referenceNumber

poll GET /auth/{referenceNumber}  (bearer: authToken)
  → wait for code=200

POST /auth/token/redeem  (bearer: authToken, empty body)
  → accessToken.token, refreshToken.token
```

### XAdES-BES Implementation Notes

```python
# Library: endesive (pip install endesive)
from endesive.xmlsig import sign

# ECDSA key: signature must be raw R||S bytes (not DER/ASN.1)
# endesive returns DER for ECDSA — convert manually:
from cryptography.hazmat.primitives.asymmetric.utils import decode_dss_signature
import base64

r, s = decode_dss_signature(der_sig)
raw_sig = r.to_bytes(32, "big") + s.to_bytes(32, "big")
sig_b64 = base64.b64encode(raw_sig).decode()

# RSA key: standard PKCS#1 or PSS — no conversion needed

# verifyCertificateChain=false for test/self-signed certs
```

### Required SignedInfo fields (exact element names)

```xml
<SignedInfo>
  <Challenge>{challenge}</Challenge>
  <Nip>{nip}</Nip>
  <Timestamp>{timestampMs}</Timestamp>
</SignedInfo>
```

## Token Lifetimes

| Token | Lifetime | Notes |
|-------|----------|-------|
| challenge | 10 min | Must use before expiry |
| authenticationToken | short | Use only to redeem |
| accessToken | minutes | JWT, short-lived |
| refreshToken | up to 7 days | Treat as secret |

## Refresh (no re-auth needed)

```
POST /auth/token/refresh  (bearer: refreshToken, empty body)
  → new accessToken.token
```

Use `refresh_access_token()` when accessToken expires mid-session.

## Public Key Certificates

```
GET /security/public-key-certificates
  → list of {certificate (DER base64), usage: ["KsefTokenEncryption"|"SymmetricKeyEncryption"]}
```

- **KsefTokenEncryption** → encrypt `{token}|{timestampMs}` during token auth
- **SymmetricKeyEncryption** → encrypt AES key during session open

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Encrypting only token (no timestamp) | Must encrypt `{token}|{timestampMs}` exactly |
| Using wrong cert for auth | Auth uses KsefTokenEncryption, NOT SymmetricKeyEncryption |
| Not polling for status | Auth is async — must poll until code=200 |
| Reusing accessToken after expiry | Use refreshToken to get new one |
| ECDSA signature in DER format | KSeF expects raw R\|\|S bytes (64 bytes for P-256), not ASN.1 DER |
| Using cert auth in test env | Test env supports token auth; cert auth is for production |

## Base URLs

- Test: `https://api-test.ksef.mf.gov.pl/api/v2`
- Production: `https://api.ksef.mf.gov.pl/api/v2`
