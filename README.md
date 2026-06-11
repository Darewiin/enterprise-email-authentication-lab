# Enterprise Email Authentication Lab

> A hands-on email security lab built on Microsoft 365 Exchange Online, demonstrating SPF, DKIM, and DMARC configuration, email header analysis, and security investigation methodology.

![Exchange Online](https://img.shields.io/badge/Exchange_Online-0078D4?style=flat-square&logo=microsoftoutlook&logoColor=white)
![Microsoft 365](https://img.shields.io/badge/Microsoft_365-0078D4?style=flat-square&logo=microsoft&logoColor=white)
![SPF](https://img.shields.io/badge/SPF-Configured-10b981?style=flat-square)
![DKIM](https://img.shields.io/badge/DKIM-Enabled-10b981?style=flat-square)
![DMARC](https://img.shields.io/badge/DMARC-Documented-f59e0b?style=flat-square)

---

## Overview

This project demonstrates practical understanding of enterprise email authentication using the same technology stack behind services like **Mimecast**, **Proofpoint**, and **Microsoft Defender for Office 365**. Built on the existing Northgate Solutions Microsoft 365 tenant from the [Unified Endpoint Management Lab](https://github.com/Darewiin/unified-endpoint-management-lab).

### What's implemented

- **Mail flow analysis** — MX record verification, message tracing in Exchange admin center, hop-by-hop delivery analysis
- **SPF configuration** — Record verified, syntax documented, pass/fail behavior analyzed
- **DKIM signing** — Enabled in Microsoft Defender portal, signatures verified in email headers, before/after comparison documented
- **DMARC documentation** — Progressive enforcement path documented (none → quarantine → reject), alignment analysis completed, DNS limitation explained
- **Email header analysis** — Inbound and outbound headers analyzed using Google Admin Toolbox, authentication results interpreted, trustworthiness methodology created
- **Security investigation scenarios** — Three realistic investigation write-ups: successful authentication, SPF failure troubleshooting, and spoofing attempt

### Before / After State


| Protocol | Phase 1 (Before) | Phase 3+ (After) |
|----------|-----------------|-----------------|
| SPF | ✅ Pass | ✅ Pass |
| DKIM | ❌ Not signing | ✅ Pass |
| DMARC | ❌ Fail (no record) | ⚠️ No record (DNS limitation) |

<img width="1210" height="632" alt="phase4-dmarc-no-record" src="https://github.com/user-attachments/assets/aede6f31-9ff6-40eb-947e-01b0b25f5db3" />

<img width="1512" height="408" alt="phase3-dkim-mxtoolbox" src="https://github.com/user-attachments/assets/ea2314e3-98bf-4371-8a54-e341699dcf6b" />

> **Note:** DMARC could not be fully implemented due to a DNS limitation of `.onmicrosoft.com` domains. See [Phase 4 documentation](docs/dmarc-implementation.md) for full explanation and what the record would look like in production.

---

## Lab Environment

| Component | Details |
|-----------|---------|
| **Platform** | Microsoft 365 E5 |
| **Email service** | Exchange Online |
| **Domain** | northgsolutions.onmicrosoft.com |
| **DNS provider** | Microsoft Corporation (managed) |
| **Tools used** | MXToolbox, Google Admin Toolbox, Microsoft Defender portal, Exchange admin center |
| **Builds on** | [UEM Lab](https://github.com/Darewiin/unified-endpoint-management-lab) — same Northgate Solutions tenant |

---

## Key DNS Limitation

This lab uses a Microsoft-managed `.onmicrosoft.com` domain. Microsoft controls the DNS centrally, meaning SPF and DMARC records cannot be modified or added directly. In a production environment with a custom domain (e.g., `northgatesolutions.com`), the administrator would have full DNS control and could configure all three records independently.

This limitation was encountered and documented throughout the project — including MXToolbox confirming **"Your DNS hosting provider is Microsoft Corporation"** and the DMARC lookup showing no record exists for either `northgsolutions.onmicrosoft.com` or the organizational domain `onmicrosoft.com`.

---

## Repository Structure

```
├── README.md
├── LICENSE
├── .gitignore
└── docs/
    ├── mail-flow-overview.md
    ├── spf-implementation.md
    ├── dkim-implementation.md
    ├── dmarc-implementation.md
    ├── header-analysis-guide.md
    ├── investigation-scenarios.md
    └── screenshots/
        ├── phase1-*.png
        ├── phase2-*.png
        ├── phase3-*.png
        ├── phase4-*.png
        └── phase5-*.png
```

---

## Authentication Flow

```
Outbound (Northgate → Gmail)
┌─────────────────────────────────────────────────────────┐
│  Jordan Martinez sends email via Exchange Online        │
│                          ↓                              │
│  Exchange Online SMTP server signs with DKIM            │
│  Sending IP: 2a01:111:f403:c107::9 (Microsoft)         │
│                          ↓                              │
│  Gmail receives → checks SPF → PASS                    │
│  Gmail verifies DKIM signature → PASS                   │
│  Gmail checks DMARC → FAIL (no record published)        │
│  Delivered in 4 seconds                                 │
└─────────────────────────────────────────────────────────┘

Inbound (Gmail → Northgate)
┌─────────────────────────────────────────────────────────┐
│  Personal Gmail sends to jmartinez@northgsolutions...   │
│                          ↓                              │
│  Originated at Gmail (mail-dl1-x122c.google.com)       │
│                          ↓                              │
│  Microsoft checks SPF → PASS (gmail.com authorized)    │
│  Microsoft verifies DKIM → PASS (d=gmail.com)          │
│  Microsoft checks DMARC → PASS (Google has p=none)     │
│  compauth=pass reason=100                               │
│  Delivered in 18 seconds via TLS_AES_256_GCM_SHA384    │
└─────────────────────────────────────────────────────────┘
```

---

## Certifications This Prepares You For

| Exam | Title | Relevance |
|------|-------|-----------|
| **SC-900** | Security Fundamentals | Email authentication, anti-phishing, Zero Trust |
| **MS-102** | M365 Administrator | Exchange Online, mail flow, email authentication policies |
| **SC-200** | Security Operations | Email threat investigation, incident response methodology |

---

## How to Reproduce

1. Sign up for a [Microsoft 365 E5 trial](https://www.microsoft.com/en-us/microsoft-365/enterprise/e5)
2. Verify your MX record at [MXToolbox](https://mxtoolbox.com)
3. Enable DKIM at [security.microsoft.com](https://security.microsoft.com) → Email authentication settings
4. Send test emails and analyze headers using [Google Admin Toolbox](https://toolbox.googleapps.com/apps/messageheader)
5. Follow the phase-by-phase documentation in [`docs/`](docs/)

---

## Related Project

This lab complements the **[Unified Endpoint Management Lab](https://github.com/Darewiin/unified-endpoint-management-lab)** — together they demonstrate both endpoint security (device compliance, Conditional Access, identity management) and communication security (email authentication, mail flow, spoofing prevention).

---

## License

This project is licensed under the [MIT License](LICENSE).
