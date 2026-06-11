# Phase 2 — SPF Implementation

## What is SPF?

SPF (Sender Policy Framework) is a DNS TXT record that lists which mail servers are authorized to send email on behalf of your domain. When a receiving server gets an email claiming to be from your domain, it checks the SPF record: "Is this server on the approved list?"

- Sending server is in the SPF record → **SPF PASS**
- Sending server is NOT in the record → **SPF FAIL**

Without SPF, anyone can send email pretending to be from your domain using any mail server.

---

## The Northgate Solutions SPF Record

```
v=spf1 include:spf.protection.outlook.com -all
```

### Syntax Breakdown

| Part | Meaning |
|------|---------|
| `v=spf1` | Identifies this as an SPF record, version 1 |
| `include:spf.protection.outlook.com` | Authorizes all Microsoft Exchange Online servers |
| `-all` | Hard fail — reject any server not listed |

The `include:spf.protection.outlook.com` mechanism expands to cover all of Microsoft's outbound mail server IP ranges. When Northgate sends email, the sending IP (`2a01:111:f403:c107::9`) falls within this range, so SPF passes.

---

## DNS Limitation Note

This lab uses a Microsoft-managed `.onmicrosoft.com` domain. Microsoft pre-configures the SPF record — it cannot be modified. MXToolbox's Email Health check defaulted to evaluating `onmicrosoft.com` (the root domain) instead of `northgsolutions.onmicrosoft.com` (the lab subdomain).

Running the check directly against the full subdomain returned more accurate results. Both screenshots are included in the screenshots folder to document this behavior.

> In a production environment with a custom domain, the administrator would control the SPF record directly in their DNS provider (Cloudflare, GoDaddy, Azure DNS) and could add or remove authorized senders as needed.

---

## Test Results

### Successful SPF Pass
Email sent from `jmartinez@northgsolutions.onmicrosoft.com` to personal Gmail:
```
SPF: PASS with IP 2a01:111:f403:c111:0:0:0:5
```
Gmail confirmed the sending IP belongs to Microsoft's authorized range.

### Header Analysis (MXToolbox)
- SPF Alignment: ✅
- SPF Authenticated: ✅
- Delivered via: outbound.protection.outlook.com → mx.google.com

---

## Common SPF Failure Scenarios

### Scenario 1 — Third-Party Sender Not in SPF
A company uses Mailchimp for marketing emails but never added Mailchimp's servers to the SPF record. Recipients see `SPF: FAIL` and emails land in spam.

**Fix:** Add `include:servers.mcsv.net` to the SPF record:
```
v=spf1 include:spf.protection.outlook.com include:servers.mcsv.net -all
```

### Scenario 2 — Too Many DNS Lookups
SPF has a hard limit of 10 DNS lookups. Organizations using many third-party senders (Salesforce, HubSpot, Zendesk, Mailchimp) often hit this limit, causing SPF to fail silently.

**Fix:** Use an SPF flattening service or consolidate senders into fewer include mechanisms.

### Scenario 3 — Soft Fail vs Hard Fail
`~all` (soft fail) delivers the email but flags it. `-all` (hard fail) rejects it. Many organizations use `~all` because they're not confident their SPF is complete, which weakens protection.

**Fix:** Audit all authorized senders, then switch from `~all` to `-all` for full enforcement.

---

## Business Impact

- Without SPF, attackers can send spoofed emails from any server claiming to be your domain
- SPF failures cause legitimate emails to land in spam or be rejected
- Misconfigured SPF is one of the most common email delivery issues in enterprise IT support
- Mimecast and similar services evaluate SPF as part of their inbound filtering decisions

---

## What This Demonstrates

| Skill | How It Maps |
|-------|-------------|
| DNS record interpretation | Reading and understanding SPF syntax |
| Email authentication troubleshooting | Diagnosing SPF failures |
| Exchange Online administration | Understanding Microsoft's mail infrastructure |
| Enterprise email security | Foundation for DMARC enforcement |
