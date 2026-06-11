# Phase 3 — DKIM Implementation

## What is DKIM?

DKIM (DomainKeys Identified Mail) adds a cryptographic digital signature to every outgoing email. The sending server signs the email with a private key, and the receiving server verifies it using a public key published in DNS.

Think of it like a wax seal on a letter — it proves:
1. The email genuinely came from the claimed domain
2. The email was not modified in transit

---

## How DKIM Works

```
1. Exchange Online signs the outgoing email with Northgate's private key
          ↓
2. The DKIM-Signature header is added to the email
          ↓
3. Receiving server (Gmail) retrieves the public key from DNS
   using the selector: selector1-northgsolutions-onmicrosoft-com._domainkey
          ↓
4. Gmail verifies the signature against the public key
          ↓
5. If valid → DKIM PASS. If modified or key mismatch → DKIM FAIL
```

---

## Configuration

DKIM was enabled in the Microsoft Defender portal:

**Path:** security.microsoft.com → Email & collaboration → Policies & rules → Threat policies → Email authentication settings → DKIM tab

**Status:** "Signing DKIM signatures for this domain" — actively signing all outbound email.

**CNAME Records (Publish CNAMEs section):**
- Host Name: `selector1._domainkey`
- Host Name: `selector2._domainkey`

> Microsoft manages two selectors to enable zero-downtime key rotation. When DKIM keys are rotated, Microsoft switches between selector1 and selector2 without any service interruption.

---

## Before / After Comparison

### Phase 2 (Before DKIM enabled)
MXToolbox analysis showed:
- DKIM Alignment: ❌
- DKIM Authenticated: ❌

### Phase 3 (After DKIM enabled)
MXToolbox analysis showed:
- DKIM Alignment: ✅
- DKIM Authenticated: ✅

Gmail "Show original" confirmed:
```
DKIM: 'PASS' with domain northgsolutions.onmicrosoft.com
```

---

## DKIM-Signature Header Analysis

The full DKIM-Signature header from a test email:

```
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed;
  d=northgsolutions.onmicrosoft.com; s=selector1-northgsolutions-onmicrosoft-com;
  h=From:Date:Subject:Message-ID:Content-Type:MIME-Version:X-MS-Exchange-SenderADCheck;
  bh=<body hash>;
  b=<cryptographic signature>
```

### Field Breakdown

| Field | Value | Meaning |
|-------|-------|---------|
| `v=1` | 1 | DKIM version 1 |
| `a=rsa-sha256` | rsa-sha256 | RSA signing with SHA-256 hash |
| `c=relaxed/relaxed` | relaxed/relaxed | Canonicalization — allows minor whitespace changes in headers and body |
| `d=` | northgsolutions.onmicrosoft.com | The domain that signed the email |
| `s=` | selector1-northgsolutions-onmicrosoft-com | Selector — tells receivers which public key to retrieve from DNS |
| `h=` | From:Date:Subject:... | Which headers were included in the signature |
| `bh=` | [hash value] | SHA-256 hash of the email body — detects body tampering |
| `b=` | [signature] | The actual RSA cryptographic signature |

---

## Why DKIM Matters

- **Tamper detection:** Any modification to the signed headers or body after sending invalidates the DKIM signature
- **Domain authentication:** Proves the email was sent by a server holding the private key for the signing domain
- **DMARC requirement:** DMARC requires either SPF or DKIM to pass WITH alignment — DKIM is the more reliable of the two because it travels with the email through forwarding chains
- **Mimecast relevance:** DKIM verification is a core function of email security gateways — understanding it is essential for Technical Support roles

---

## What This Demonstrates

| Skill | How It Maps |
|-------|-------------|
| Public key cryptography | Understanding how DKIM signing works |
| Microsoft Defender administration | Configuring email authentication settings |
| Email header analysis | Reading and interpreting DKIM-Signature fields |
| Before/after testing | Validating configuration changes with evidence |
