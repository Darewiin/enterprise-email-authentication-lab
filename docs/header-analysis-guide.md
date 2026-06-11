# Phase 5 — Email Header Analysis Guide

## How to Determine if an Email is Trustworthy

Email headers contain the complete authentication trail — every server the email passed through, every check performed, and every result. This guide explains how to read them and make a trust determination.

---

## Key Headers to Understand

| Header | Purpose |
|--------|---------|
| `Authentication-Results` | Summary of SPF, DKIM, and DMARC checks — most important for security analysis |
| `Return-Path` | Actual sender address used for SPF checking — may differ from visible From: |
| `From` | Visible sender shown to the user — what attackers spoof |
| `Received` | Each server the email passed through — read bottom to top (oldest first) |
| `DKIM-Signature` | Cryptographic signature with algorithm, domain, selector, and signed fields |
| `X-Forefront-Antispam-Report` | Microsoft's spam filtering verdict and reason codes |
| `X-MS-Exchange-Organization-SCL` | Spam Confidence Level (-1 = not spam, 9 = definite spam) |

---

## The 5-Step Trustworthiness Checklist

### Step 1 — Check Authentication-Results First

This is the summary verdict. Look for:
```
Authentication-Results: mx.google.com;
  spf=pass smtp.mailfrom=yourdomain.com;
  dkim=pass header.d=yourdomain.com;
  dmarc=pass action=none header.from=yourdomain.com;
  compauth=pass reason=100
```

- **SPF + DKIM + DMARC all pass** = strong trust signal
- **Any failure** = investigate further, do not automatically trust
- **compauth=pass** = Microsoft's composite score passed (M365 tenants)

### Step 2 — Check Return-Path vs From:

The Return-Path is what SPF checks. The From: is what you see. They should share the same domain.

```
From: CEO <ceo@yourcompany.com>
Return-Path: ceo@yourcompany.com    ← LEGITIMATE (domains match)

From: CEO <ceo@yourcompany.com>
Return-Path: random@attacker.com    ← RED FLAG (spoofing indicator)
```

A mismatch between Return-Path and From: domains is the most common indicator of a spoofing or phishing attempt.

### Step 3 — Check the Received Chain

Read the Received headers from bottom to top — that's chronological order. Ask:
- Does the originating server make sense for the claimed sender?
- Gmail sending from `mail-dl1-x122c.google.com` = expected ✅
- Company email arriving from a random VPS in an unexpected country = suspicious ❌

Example from lab (inbound Gmail → Northgate):
```
Hop 1: Originated at Gmail (mail-dl1-x122c.google.com)
Hop 2: BL02EPF0001A108.namprd05 → MN2PR15CA0048.outlook.office365.com
Hop 3: DM6PR19MB4326 → DM6PR19MB4357 (internal Microsoft routing)
Total: 18 seconds
```

### Step 4 — Check DKIM-Signature Domain

The `d=` field in the DKIM-Signature must match the From: domain for DKIM alignment to pass:

```
DKIM-Signature: v=1; a=rsa-sha256; d=gmail.com; s=20251104;
```

- `d=gmail.com` and From: is `@gmail.com` → DKIM alignment passes ✅
- `d=attacker.com` and From: is `@yourcompany.com` → DKIM alignment fails ❌

### Step 5 — Check Microsoft-Specific Headers (M365 Tenants)

For emails received by Microsoft 365 tenants, check:

| Header | Value | Meaning |
|--------|-------|---------|
| `SCL` | -1 | Not spam (bypassed filter) |
| `SCL` | 1-4 | Low spam probability |
| `SCL` | 5-9 | High spam probability |
| `SFV:NSPM` | — | Spam Filter Verdict: Not Spam |
| `SFV:SPM` | — | Spam Filter Verdict: Spam |
| `BCL` | 0-9 | Bulk Complaint Level (higher = more likely bulk mail) |

---

## Real Header Analysis — Lab Examples

### Inbound Email (Gmail → Northgate) — Fully Authenticated
```
Authentication-Results: spf=pass smtp.mailfrom=gmail.com;
  dkim=pass header.d=gmail.com;
  dmarc=pass action=none header.from=gmail.com;
  compauth=pass reason=100

X-MS-Exchange-Organization-SCL: 1
X-Forefront-Antispam-Report: SFV:NSPM
```
**Verdict: TRUSTWORTHY** — All three protocols pass, Microsoft marks as not spam.

### Outbound Email (Northgate → Gmail) — SPF/DKIM Pass, DMARC Missing
```
SPF: PASS with IP 2a01:111:f403:c107:0:0:0:9
DKIM: 'PASS' with domain northgsolutions.onmicrosoft.com
DMARC: 'FAIL'
```
**Verdict: LIKELY LEGITIMATE** — SPF and DKIM pass, DMARC fails only because no record is published (DNS limitation). Alignment analysis confirms both SPF and DKIM domains match the From: domain.

---

## ARC (Authenticated Received Chain)

The lab headers revealed two ARC seals — a more advanced authentication concept:

```
ARC-Seal: i=1; d=google.com     ← Google added first seal
ARC-Seal: i=2; d=microsoft.com  ← Microsoft verified Google's seal
```

ARC preserves authentication results through email forwarding. When an email is forwarded, SPF often breaks (because the forwarding server wasn't in the original SPF). ARC allows the final recipient to see the original authentication results before forwarding.

---

## What This Demonstrates

| Skill | How It Maps |
|-------|-------------|
| Header analysis | Core skill for email security support roles |
| Authentication interpretation | Understanding SPF/DKIM/DMARC results |
| Spoofing detection | Identifying mismatches in sender fields |
| Microsoft 365 expertise | Reading Microsoft-specific antispam headers |
| Investigation methodology | Systematic approach to email security incidents |
