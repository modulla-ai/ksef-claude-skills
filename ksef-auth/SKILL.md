---
name: ksef-auth
description: Use when authenticating with KSeF API 2.0 (Polish e-invoice system) using a KSeF token, or when accessToken is missing/expired and needs refreshing.
---

# KSeF — Uwierzytelnianie (API 2.0)

## Overview

KSeF API 2.0 uses JWT-based authentication. Auth is a separate, independent process from opening a session. One access token can be used for multiple sessions.

Implementation: `/root/ksef/ksef_client.py` → `KSeFClient.authorise()` and `refresh_access_token()`

## Auth Flow (KSeF Token method)

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

- **KsefTokenEncryption** → encrypt `{token}|{timestampMs}` during auth
- **SymmetricKeyEncryption** → encrypt AES key during session open

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Encrypting only token (no timestamp) | Must encrypt `{token}|{timestampMs}` exactly |
| Using wrong cert for auth | Auth uses KsefTokenEncryption, NOT SymmetricKeyEncryption |
| Not polling for status | Auth is async — must poll until code=200 |
| Reusing accessToken after expiry | Use refreshToken to get new one |

## Base URLs

- Test: `https://api-test.ksef.mf.gov.pl/api/v2`
- Production: `https://api.ksef.mf.gov.pl/api/v2`
