---
name: ksef-send-invoice
description: Use when sending an invoice (FA, KOR, FA_RR) to KSeF API 2.0, opening an online session, encrypting XML, waiting for a KSeF number after submission, or handling duplicate invoice errors.
---

# KSeF — Wysyłka faktury (sesja interaktywna, API 2.0)

## Overview

Every invoice must be AES-256-CBC encrypted before sending. Encryption is mandatory in API 2.0 (both interactive and batch). The AES key is generated fresh per session and sent RSA-encrypted to KSeF when opening the session.

Implementation: `/root/ksef/ksef_client.py` → `KSeFClient.open_session()`, `send_invoice()`, `terminate_session()`

## Full Flow

```
1. authorise()  →  accessToken
   (see ksef-auth skill)

2. Generate AES-256 key (32 bytes) + IV (16 bytes) randomly

3. POST /sessions/online  (bearer: accessToken)
   body: {
     formCode: { systemCode:"FA (2)", schemaVersion:"1-0E", value:"FA" },
     encryption: {
       encryptedSymmetricKey: base64(RSA-OAEP-encrypt(aes_key, SymmetricKeyEncryption_cert)),
       initializationVector: base64(iv)
     }
   }
   → referenceNumber (sessionRef), validUntil (12h lifetime)

4. For each invoice:
   a. encrypted = AES-256-CBC(PKCS7-padded XML, key, iv)
   b. POST /sessions/online/{sessionRef}/invoices  (bearer: accessToken)
      body: {
        invoiceHash:           base64(SHA256(original_xml)),
        invoiceSize:           len(original_xml),
        encryptedInvoiceHash:  base64(SHA256(encrypted)),
        encryptedInvoiceSize:  len(encrypted),
        encryptedInvoiceContent: base64(encrypted)
      }
      → referenceNumber (invoiceRef)

5. poll GET /sessions/{sessionRef}/invoices/{invoiceRef}  (bearer: accessToken)
   status.code: 200=ok → ksefNumber assigned
   wait 3s between polls

6. POST /sessions/online/{sessionRef}/close  (bearer: accessToken)
   → triggers async UPO generation
```

## Supported formCode schemas

| systemCode | schemaVersion | value |
|------------|---------------|-------|
| FA (2) | 1-0E | FA |
| FA (3) | 1-0E | FA |
| FA_RR(1) | 1-1E | FA_RR |
| PEF(3) | 2-1 | PEF |

## Encryption Details

```python
# AES-256-CBC with PKCS#7 padding
padded = pkcs7_pad(xml_bytes, block_size=16)
encrypted = AES_CBC_encrypt(padded, key=aes_key, iv=aes_iv)
# IV is sent at session open — NOT prepended to ciphertext
```

## Error 440 — Duplikat faktury

If the same invoice number is submitted again, KSeF returns error code **440** (Duplikat faktury). The response contains the **original KSeF number** from the first submission.

```python
# Response body when error 440:
# {
#   "status": { "code": 440, "description": "Duplikat faktury" },
#   "extensions": { "originalKsefNumber": "1234567890-20260101-XXXX-YYYY-ZZ" }
# }

# Handling:
if status_code == 440:
    original_number = response["status"].get("extensions", {}).get("originalKsefNumber")
    # Treat as success — return the original KSeF number
    return original_number
```

This is idempotent: safe to retry — returns the same KSeF number.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Prepending IV to ciphertext | IV is sent separately at session open — raw ciphertext only in invoiceContent |
| Wrong cert for AES key encryption | Use SymmetricKeyEncryption cert, NOT KsefTokenEncryption |
| Hash of encrypted, not original | Both hashes required: original XML hash AND encrypted bytes hash |
| Sending unencrypted XML | API 2.0 requires encryption — no plaintext option |
| Not closing session | Close triggers UPO generation — always close when done |
| Reusing AES key across sessions | Generate fresh key + IV for each new session |
| Treating error 440 as failure | Duplikat means invoice already exists — extract originalKsefNumber and continue |

## Environments

| | Base URL |
|-|---------|
| Test | `https://api-test.ksef.mf.gov.pl/api/v2` |
| Production | `https://api.ksef.mf.gov.pl/api/v2` |

API flow is **identical** in both — only `base_url` changes (set in `KSeFClient.__init__`).
Test env: use test NIP + test token, invoices don't have legal effect.

## Session Lifecycle

- Session valid for **12 hours** from creation
- Multiple sessions can be open simultaneously under one auth
- Closing is lightweight (async) — call even if all invoices sent
