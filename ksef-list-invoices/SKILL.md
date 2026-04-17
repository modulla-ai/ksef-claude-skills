---
name: ksef-list-invoices
description: Use when querying, listing, or downloading invoices from KSeF API 2.0, including sent invoices, received invoices, session status, or UPO documents.
---

# KSeF — Pobieranie faktur i statusów (API 2.0)

## Overview

KSeF API 2.0 provides separate endpoints for sent (via sessions) and received invoices. UPO (Urzędowe Potwierdzenie Odbioru) is generated per session after closing.

Implementation: `/root/ksef/ksef_client.py` → `query_invoices()`, `get_invoice_xml()`, `wait_for_ksef_number()`

## Sent Invoices (wystawione)

```
GET /sessions?sessionType=online  (bearer: accessToken)
  → { sessions: [{referenceNumber, dateCreated, totalInvoiceCount, ...}] }

GET /sessions/{sessionRef}/invoices  (bearer: accessToken)
  → { invoices: [{referenceNumber, ksefNumber, status, ...}] }

GET /sessions/{sessionRef}/invoices/{invoiceRef}  (bearer: accessToken)
  → { status: {code}, ksefNumber, upoDownloadUrl }
```

Filter sessions by `dateCreated` client-side (no server-side date filter on sessions endpoint).

## Received Invoices (otrzymane)

```
GET /invoices/received?dateFrom=...&dateTo=...&pageOffset=0&pageSize=100
  (bearer: accessToken)
  → { invoices: [...] }
```

Note: May not be available in test environment — handle gracefully.

## Invoice Status Codes

| code | meaning |
|------|---------|
| 100 | In progress (processing) |
| 200 | OK — KSeF number assigned |
| 4xx | Validation error |
| 5xx | System error |

## Download UPO

```
GET /sessions/{sessionRef}/invoices/{invoiceRef}
  → upoDownloadUrl  (pre-signed URL, time-limited)

GET {upoDownloadUrl}  (no auth header needed)
  → UPO XML bytes
```

## Session Status & Bulk UPO

```
GET /sessions/{sessionRef}  (bearer: accessToken)
  → { status, totalInvoiceCount, processedCount, errorCount }

# Bulk UPO for entire session available after session is closed
GET /sessions/{sessionRef}/upo  (bearer: accessToken)
  → bulk UPO XML
```

## Reference Number Format (internal convention)

Our system uses `{sessionRef}|{invoiceRef}` as a combined identifier:
```python
# To retrieve UPO:
session_ref, invoice_ref = ksef_reference_number.split("|", 1)
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Polling without delay | Wait 3s between status polls to avoid rate limiting |
| Using upoDownloadUrl without time check | URL is pre-signed and expires — use immediately |
| Querying received in test env | Endpoint may not exist — wrap in try/except |
| Not paginating large result sets | Use pageOffset + pageSize for >100 invoices |

## Environments

| | Base URL |
|-|---------|
| Test | `https://api-test.ksef.mf.gov.pl/api/v2` |
| Production | `https://api.ksef.mf.gov.pl/api/v2` |

Test env may have missing endpoints (e.g. `/invoices/received`) — handle with try/except.

## Rate Limits

Limits apply per context + IP (authenticated) or per IP (public).
Published limits: `/tmp/ksef-docs/limity/limity-api.md`
Test environment has less restrictive limits than production.
