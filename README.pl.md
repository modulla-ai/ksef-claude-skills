# KSeF Skills dla Claude Code

Skills dla Claude Code do integracji z **KSeF API 2.0** — Krajowym Systemem e-Faktur.

> Stworzone i utrzymywane przez [modulla.ai](https://modulla.ai) — prosta integracja z KSeF dla polskich firm.

---

## Czym są Claude Code Skills?

[Skills](https://docs.anthropic.com/en/docs/claude-code/skills) to pliki Markdown, które Claude ładuje automatycznie gdy pracujesz nad powiązanym zadaniem. Zamiast tłumaczyć za każdym razem jak działa KSeF API — skill jest zawsze pod ręką.

## Dostępne skille

| Skill | Opis |
|-------|------|
| `ksef-auth` | Uwierzytelnianie tokenem KSeF — flow challenge, odświeżanie tokenu, certyfikaty klucza publicznego |
| `ksef-send-invoice` | Wysyłka faktur — sesja online, szyfrowanie AES-256-CBC, polling numeru KSeF |
| `ksef-generate-xml` | Generowanie XML FA(2) — struktura, faktury korygujące (KOR), walidacja XSD |
| `ksef-list-invoices` | Pobieranie faktur — wystawione, otrzymane, pobieranie UPO, status sesji |

Skille obejmują środowisko **testowe** (`api-test.ksef.mf.gov.pl`) i **produkcyjne** (`api.ksef.mf.gov.pl`). Flow API jest identyczny — różni się tylko base URL.

## Instalacja

```bash
# Sklonuj repo
git clone https://github.com/modulla-ai/ksef-claude-skills.git

# Skopiuj skille do Claude Code
cp -r ksef-claude-skills/ksef-* ~/.claude/skills/
```

Skille są automatycznie wykrywane przez Claude Code z katalogu `~/.claude/skills/`.

## Użycie

Po instalacji Claude automatycznie używa odpowiedniego skilla. Możesz też wywołać wprost:

```
Wyślij tę fakturę do KSeF...
Wygeneruj XML FA(2) dla tej faktury...
Pobierz listę otrzymanych faktur z KSeF...
Uwierzytelnij się w KSeF moim tokenem...
```

## Co jest pokryte

- **Flow uwierzytelniania:** challenge → szyfrowanie RSA-OAEP tokenu → polling → redeem → accessToken + refreshToken
- **Szyfrowanie:** AES-256-CBC z paddingiem PKCS#7, IV wysyłany przy otwieraniu sesji (nie dołączany do zaszyfrowanych danych)
- **Dwa klucze publiczne:** `KsefTokenEncryption` (auth) vs `SymmetricKeyEncryption` (klucz AES sesji)
- **Wysyłka faktury:** 5 wymaganych pól z hashami, polling numeru KSeF, cykl życia sesji
- **XML FA(2):** kolejność elementów, faktury korygujące KOR, kody stawek VAT, walidacja XSD
- **Pobieranie faktur:** sesje, faktury otrzymane, pobieranie UPO, limity zapytań

## Wersja API

Skille napisane dla **KSeF API 2.0** (obowiązkowe od 1 lutego 2026).

## Contributing

PR-y mile widziane — szczególnie:
- Obsługa schematu FA(3)
- Sesje wsadowe (batch)
- FA_RR (faktury VAT RR)

## Licencja

MIT

---

Zbudowane przez [modulla.ai](https://modulla.ai) — integracja z KSeF, która po prostu działa.
