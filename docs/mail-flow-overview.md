# Phase 1 — Mail Flow Overview

## How Email Travels Through an Enterprise

Understanding mail flow is the foundation of email security. Every email authentication protocol (SPF, DKIM, DMARC) operates at a specific point in this journey.

---

## The Mail Flow Path

```
Sender (Outlook / Gmail)
        ↓
Sender's mail server (Exchange Online / Google SMTP)
        ↓
DNS Lookup → MX Record query: "Where do I send mail for this domain?"
        ↓
SMTP Transfer (port 25, TLS encrypted)
        ↓
Recipient's mail server (Exchange Online / Gmail)
        ↓
Authentication checks:
  → SPF:   Is this server authorized to send for this domain?
  → DKIM:  Was this email signed by the domain's private key?
  → DMARC: Do SPF/DKIM results align with the From: domain?
        ↓
Delivery decision: Inbox / Junk / Quarantine / Reject
```

---

## Key Concepts

### SMTP (Simple Mail Transfer Protocol)
The protocol used to transmit email between servers. Port 25 is used for server-to-server transfer. Port 587 is used for client-to-server submission (when you click Send in Outlook).

### MX Record (Mail Exchanger)
A DNS record that tells other mail servers where to deliver email for your domain. The Northgate Solutions MX record points to Microsoft's Exchange Online servers:
```
northgsolutions.onmicrosoft.com → northgsolutions-onmicrosoft-com.mail.protection.outlook.com
```

### DNS (Domain Name System)
The "phone book" of the internet. All three email authentication records (SPF, DKIM, DMARC) are published as DNS TXT records that receiving servers look up to validate incoming email.

### Exchange Online
Microsoft's cloud email service included in Microsoft 365. It handles:
- Inbound mail reception and spam filtering
- Outbound mail sending and DKIM signing
- Mail flow rules and policies
- Message tracing and audit logging

---

## Lab Verification

### MX Record Confirmed
MXToolbox lookup of `northgsolutions.onmicrosoft.com` confirmed the MX record points to Microsoft's mail protection servers. All inbound email flows through Exchange Online before reaching user mailboxes.

### Message Trace
Exchange admin center → Mail flow → Message trace confirmed:
- Test email from Jordan Martinez delivered to personal Gmail in **9 seconds**
- Test email from personal Gmail delivered to Jordan Martinez in **18 seconds**
- Both directions passing through Microsoft's mail protection infrastructure

### Before State (Phase 1 Baseline)
Gmail's "Show original" on the first test email revealed:
- **SPF: PASS** — Microsoft's servers are pre-authorized
- **DKIM: absent** — not yet configured
- **DMARC: FAIL** — no record published

This baseline was used as the starting point for Phases 2-4.

---

## What This Demonstrates

| Skill | How It Maps |
|-------|-------------|
| Understanding mail flow | Foundation for troubleshooting email delivery issues |
| MX record verification | Standard first step in any email investigation |
| Message tracing | Core skill for Exchange Online administrators |
| Header interpretation | Used daily in email security support roles |
