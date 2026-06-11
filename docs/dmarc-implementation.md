# Phase 4 — DMARC Implementation

## What is DMARC?

DMARC (Domain-based Message Authentication, Reporting & Conformance) ties SPF and DKIM together. It tells receiving servers: "If an email claims to be from my domain and authentication fails, here's what to do with it."

DMARC also introduces **alignment** — the domain in the visible From: header must match the domain validated by SPF or DKIM. This prevents attackers from passing SPF/DKIM with their own domain while spoofing yours in the visible From: field.

---

## DMARC Policies

| Policy | Action | Use Case |
|--------|--------|----------|
| `p=none` | Monitor only — deliver the email, send reports | Start here — collect data without risk |
| `p=quarantine` | Send failing emails to spam/junk | Medium enforcement — visible impact |
| `p=reject` | Block failing emails entirely | Full enforcement — the end goal |

---

## What the DMARC Record Would Be

Due to the DNS limitation of `.onmicrosoft.com` domains (see below), the DMARC record could not be published. In a production environment with a custom domain, the following records would be configured progressively:

### Stage 1 — Monitoring (Week 1)
```
v=DMARC1; p=none; rua=mailto:dmarc@northgsolutions.onmicrosoft.com
```

### Stage 2 — Soft Enforcement (Weeks 2-3)
```
v=DMARC1; p=quarantine; pct=50; rua=mailto:dmarc@northgsolutions.onmicrosoft.com
```

### Stage 3 — Full Enforcement (Week 4+)
```
v=DMARC1; p=reject; rua=mailto:dmarc@northgsolutions.onmicrosoft.com
```

### Record Field Breakdown

| Field | Meaning |
|-------|---------|
| `v=DMARC1` | DMARC version 1 |
| `p=none/quarantine/reject` | Policy to apply on failure |
| `pct=50` | Apply policy to 50% of failing emails (gradual rollout) |
| `rua=mailto:` | Email address to receive aggregate reports |

---

## DNS Limitation

MXToolbox DMARC lookup for `northgsolutions.onmicrosoft.com` returned:

```
No DMARC Record found for sub-domain.
Organization Domain of this sub-domain is: onmicrosoft.com
Inbox Receivers will apply onmicrosoft.com DMARC record to mail
sent from northgsolutions.onmicrosoft.com
```

And for the organizational domain:
```
No DMARC Record found for onmicrosoft.com
```

MXToolbox also confirmed: **"Your DNS hosting provider is Microsoft Corporation"**

This means:
1. Microsoft manages DNS for all `.onmicrosoft.com` domains centrally
2. The `_dmarc.northgsolutions.onmicrosoft.com` TXT record cannot be added
3. Receiving servers looking for a DMARC policy fall back to `onmicrosoft.com` — which also has no record
4. Without a DMARC record, receiving servers have no policy to enforce — they evaluate SPF and DKIM but take no action on failure

> In a production environment with a custom domain (e.g., `northgatesolutions.com`), the administrator would add the DMARC TXT record directly in their DNS provider. The authentication concepts demonstrated here apply identically.

---

## DMARC Decision Process

```
Incoming email claiming to be from northgsolutions.onmicrosoft.com
                          ↓
              Check SPF — does it PASS?
     Does the SPF domain ALIGN with the From: domain?
                          ↓
             Check DKIM — does it PASS?
     Does the DKIM domain ALIGN with the From: domain?
                          ↓
   If SPF passes WITH alignment → DMARC PASS
   If DKIM passes WITH alignment → DMARC PASS
   If BOTH fail or neither aligns → DMARC FAIL
                          ↓
      Apply policy: none / quarantine / reject
```

---

## Manual Alignment Analysis

For outbound test emails (Northgate → Gmail):

**SPF Alignment:**
- Return-Path: `jmartinez@northgsolutions.onmicrosoft.com`
- From: `jmartinez@northgsolutions.onmicrosoft.com`
- ✅ Domains match — SPF alignment passes

**DKIM Alignment:**
- DKIM signing domain (d=): `northgsolutions.onmicrosoft.com`
- From: domain: `northgsolutions.onmicrosoft.com`
- ✅ Domains match — DKIM alignment passes

**DMARC Conclusion:**
Both SPF and DKIM pass with alignment. If a DMARC record existed with `p=none`, DMARC would PASS. The only reason DMARC fails is the missing DNS record — not an authentication failure.

---

## Microsoft 365 Safety Net

Even without a published DMARC record, Microsoft 365 provides protection through:

- **Spoof Intelligence:** ON — detects and flags spoofed senders
- **Mailbox Intelligence:** ON — learns from user behavior to identify impersonation
- **Composite Authentication (compauth):** Microsoft's proprietary authentication score that combines SPF, DKIM, DMARC, and behavioral signals

These controls act as a safety net, but a published DMARC `p=reject` policy remains the industry-recommended standard for full protection.

---

## What This Demonstrates

| Skill | How It Maps |
|-------|-------------|
| DMARC policy design | Understanding progressive enforcement |
| DNS record management | Knowing what records to create and why |
| Alignment analysis | Diagnosing why DMARC passes or fails |
| Microsoft 365 security | Understanding built-in protections |
