---
name: ksef-generate-xml
description: Use when generating FA(2) or FA(3) invoice XML for KSeF, including corrective invoices (KOR), or when validating XML structure against the official XSD schema.
---

# KSeF — Generowanie XML faktury (FA(2)/FA(3))

## Overview

KSeF accepts invoices in FA(2) and FA(3) XML schemas published by Ministerstwo Finansów. XML must validate against the official XSD before sending.

Implementation: `/root/ksef/xml_generator.py` → `generate_fa_xml()`, `generate_kor_xml()`
Schemas (local): `/tmp/ksef-docs/faktury/schemy/FA/`

## Key Differences FA(2) vs FA(3)

| Feature | FA(2) | FA(3) |
|---------|-------|-------|
| Schema version | 1-0E | 1-0E |
| systemCode | FA (2) | FA (3) |
| Status | Current standard | Newer, also supported |
| Our implementation | ✅ xml_generator.py | ❌ not yet implemented |

## FA(2) Structure (required element order)

```xml
<Faktura xmlns="http://crd.gov.pl/wzor/2023/06/29/12648/">
  <Naglowek>
    <KodFormularza kodSystemowy="FA (2)" wersjaSchemy="1-0E">FA</KodFormularza>
    <WariantFormularza>2</WariantFormularza>
    <DataWytworzeniaFa>2026-01-15T10:00:00Z</DataWytworzeniaFa>
    <SystemInfo>NazwaSystemu</SystemInfo>
  </Naglowek>
  <Podmiot1>  <!-- Sprzedawca (seller) -->
    <DaneIdentyfikacyjne><NIP>1234567890</NIP><Nazwa>...</Nazwa></DaneIdentyfikacyjne>
    <Adres>...</Adres>
  </Podmiot1>
  <Podmiot2>  <!-- Nabywca (buyer) -->
    <DaneIdentyfikacyjne><NIP>...</NIP><Nazwa>...</Nazwa></DaneIdentyfikacyjne>
    <Adres>...</Adres>
  </Podmiot2>
  <Fa>
    <KodWaluty>PLN</KodWaluty>
    <P_1>2026-01-15</P_1>   <!-- data wystawienia -->
    <P_2>FV/01/2026</P_2>   <!-- numer faktury -->
    <RodzajFaktury>VAT</RodzajFaktury>   <!-- VAT | KOR | ZAL | ROZ | UPR | KOR_ZAL | KOR_ROZ -->
    <!-- KOR only — must come immediately after RodzajFaktury: -->
    <PrzyczynaKorekty>Błędna cena</PrzyczynaKorekty>
    <DaneFaKorygowanej>
      <DataWystFaKorygowanej>2026-01-01</DataWystFaKorygowanej>
      <NrFaKorygowanej>FV/01/2026</NrFaKorygowanej>
      <NrKSeFFaKorygowanej>1234567890-20260101-XXXX</NrKSeFFaKorygowanej>
    </DaneFaKorygowanej>
    <!-- end KOR section -->
    <FaWiersz>...</FaWiersz>  <!-- one per line item -->
    <Rozliczenie>...</Rozliczenie>  <!-- tax summary -->
    <Platnosc>...</Platnosc>  <!-- payment terms -->
  </Fa>
</Faktura>
```

## KOR (Corrective Invoice) — Critical Element Order

**MUST follow this exact order inside `<Fa>`:**
1. `<RodzajFaktury>KOR</RodzajFaktury>`
2. `<PrzyczynaKorekty>` — reason for correction
3. `<DaneFaKorygowanej>` — original invoice data (date, number, KSeF number)
4. Then remaining Fa elements

XSD validation will fail if order is wrong.

## VAT Rates (P_12 in FaWiersz)

| stawka_vat | P_12 value |
|------------|------------|
| "23" | 23 |
| "8" | 8 |
| "5" | 5 |
| "0" | 0 |
| "zw" | ZW |

## XSD Validation (before sending)

```python
from lxml import etree
schema_path = "/tmp/ksef-docs/faktury/schemy/FA/schemat_FA(2)_v1-0E.xsd"
schema = etree.XMLSchema(etree.parse(schema_path))
doc = etree.fromstring(xml_bytes)
schema.assertValid(doc)  # raises if invalid
```

## Environments

XML schema and validation rules are **identical** for test and production.
Test invoices (sent to `api-test.ksef.mf.gov.pl`) have no legal effect — safe for development.
Production invoices are legally binding documents.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| KOR elements in wrong order | RodzajFaktury → PrzyczynaKorekty → DaneFaKorygowanej (strict) |
| Missing NrKSeFFaKorygowanej | Required for KOR — KSeF number of original invoice |
| Wrong namespace | Must match schema exactly: `http://crd.gov.pl/wzor/2023/06/29/12648/` |
| Not validating before send | Always validate against XSD — KSeF rejects invalid XML asynchronously |
