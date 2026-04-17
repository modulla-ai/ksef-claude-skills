# KSeF Claude Code Skills

Claude Code skills for integrating with **KSeF API 2.0** — the Polish National e-Invoice System (Krajowy System e-Faktur).

> Skills created and maintained by [modulla.ai](https://modulla.ai) — simple KSeF integration for Polish businesses.

---

## What are Claude Code Skills?

[Claude Code Skills](https://docs.anthropic.com/en/docs/claude-code/skills) are markdown reference guides that Claude loads on demand when you're working on a relevant task. Instead of re-explaining KSeF's API quirks every session, the skill is always there.

## Available Skills

| Skill | Description |
|-------|-------------|
| `ksef-auth` | Authentication with KSeF token — challenge flow, token refresh, public key certificates |
| `ksef-send-invoice` | Sending invoices — online session, AES-256-CBC encryption, polling for KSeF number |
| `ksef-generate-xml` | Generating FA(2) invoice XML — structure, KOR (corrective invoices), XSD validation |
| `ksef-list-invoices` | Querying invoices — sent, received, UPO download, session status |

Both **test** (`api-test.ksef.mf.gov.pl`) and **production** (`api.ksef.mf.gov.pl`) environments are covered. The API flow is identical — only the base URL differs.

## Installation

Copy the skill folders to your Claude Code skills directory:

```bash
# Clone the repo
git clone https://github.com/modulla-ai/ksef-claude-skills.git

# Copy skills to Claude Code
cp -r ksef-claude-skills/ksef-auth ~/.claude/skills/
cp -r ksef-claude-skills/ksef-send-invoice ~/.claude/skills/
cp -r ksef-claude-skills/ksef-generate-xml ~/.claude/skills/
cp -r ksef-claude-skills/ksef-list-invoices ~/.claude/skills/
```

Or install all at once:

```bash
git clone https://github.com/modulla-ai/ksef-claude-skills.git
cp -r ksef-claude-skills/ksef-* ~/.claude/skills/
```

Skills are automatically discovered by Claude Code from `~/.claude/skills/`.

## Usage

Once installed, Claude will automatically use the relevant skill when you ask about KSeF. You can also invoke explicitly:

```
Send this invoice to KSeF...
Generate FA(2) XML for this invoice...
List my received invoices from KSeF...
Authenticate with KSeF using my token...
```

## What's covered

- **Auth flow:** challenge → RSA-OAEP encrypt token → poll → redeem → accessToken + refreshToken
- **Encryption:** AES-256-CBC with PKCS#7 padding, IV sent at session open (not prepended to ciphertext)
- **Two public keys:** `KsefTokenEncryption` (auth) vs `SymmetricKeyEncryption` (session AES key)
- **Invoice sending:** 5 required hash fields, polling for KSeF number, session lifecycle
- **FA(2) XML:** element order, KOR corrective invoices, VAT rate codes, XSD validation
- **Invoice listing:** sessions, received invoices, UPO download, rate limits

## API Version

These skills are written for **KSeF API 2.0** (mandatory from February 1, 2026).

## Contributing

PRs welcome — especially for:
- FA(3) schema support
- Batch sessions
- FA_RR (VAT RR invoices)

## License

MIT

---

Built by [modulla.ai](https://modulla.ai) — KSeF integration that just works.
