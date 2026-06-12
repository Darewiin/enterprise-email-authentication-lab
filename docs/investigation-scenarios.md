# Phase 6 — Security Investigation Scenarios

Three realistic email security scenarios demonstrating investigation methodology for Technical Support Engineer roles. Each scenario follows the standard incident investigation format: Symptoms → Investigation → Findings → Root Cause → Resolution → Prevention.

---

## Scenario A — Successful Authenticated Email

### Situation
A user sends an email to an external contact and wants to confirm it was delivered securely and that authentication is working correctly.

### Symptoms
- Email sent from `jmartinez@northgsolutions.onmicrosoft.com` to external Gmail account
- No bounce message received
- User wants to verify the email appears legitimate to recipients

### Investigation Steps
1. Ask the recipient to open the email and check authentication results via "Show original"
2. Review the Authentication-Results header in the received email
3. Verify SPF, DKIM, and DMARC results
4. Check the DKIM-Signature header to confirm signing domain matches From: domain
5. Run the email headers through MXToolbox or Google Admin Toolbox for visual confirmation

### Findings
```
From: Jordan Martinez <jmartinez@northgsolutions.onmicrosoft.com>
Return-Path: jmartinez@northgsolutions.onmicrosoft.com

SPF: PASS with IP 2a01:111:f403:c107:0:0:0:9
DKIM: PASS with domain northgsolutions.onmicrosoft.com
DMARC: FAIL (no DMARC record published — DNS limitation)

DKIM-Signature: v=1; a=rsa-sha256;
  d=northgsolutions.onmicrosoft.com; s=selector1-northgsolutions-onmicrosoft-com
```

### Root Cause
N/A — email is legitimate. SPF and DKIM both pass. DMARC fails only because no DMARC record is published for the `.onmicrosoft.com` domain (Microsoft manages DNS centrally). This is a known lab environment limitation, not an authentication failure.

### Alignment Analysis
- Return-Path domain = From: domain ✅ → SPF alignment passes
- DKIM d= domain = From: domain ✅ → DKIM alignment passes
- If a DMARC record existed → DMARC would PASS

### Resolution
Email is functioning correctly. Authentication is working. The DMARC failure would be resolved in production by publishing a DMARC record — not possible in this lab due to DNS limitations.

### Conclusion
✅ **Email is trustworthy.** SPF confirms the sending server is authorized. DKIM confirms the message was not modified in transit. Both domains align correctly.

---

## Scenario B — SPF Failure Investigation

### Situation
A user reports that emails they send to external contacts are landing in spam folders. No bounce messages. Recipients confirm the emails arrive but are flagged as suspicious.

### Symptoms
- Outbound emails from specific users consistently going to recipient spam
- No error messages or delivery failures on the sending side
- Recipient spam filter notes: "SPF authentication failed"
- Issue started after the company began using a new marketing platform

### Investigation Steps
1. Ask the affected user which email address and which recipients are affected
2. Ask a recipient to share the email headers showing the SPF result
3. Go to **Exchange admin center → Mail flow → Message trace** — confirm the email was sent through Exchange Online
4. Run **MXToolbox SPF Lookup** on the sending domain — review the current record
5. Identify all services currently sending email on behalf of the domain
6. Compare authorized senders in SPF record against actual sending services
7. Check if the SPF record exceeds 10 DNS lookups (lookup limit)

### Findings
```
Current SPF record:
v=spf1 include:spf.protection.outlook.com -all

Marketing platform sending IP: 198.2.136.64 (Mailchimp)
SPF check result: FAIL — 198.2.136.64 not found in spf.protection.outlook.com
```

The marketing platform was added recently but its servers were never added to the SPF record. Receiving servers checked the SPF record, found the Mailchimp IP was not authorized, and flagged the email as a potential spoof.

### Root Cause
**Missing SPF include for third-party sender.** The organization added a marketing email platform after the SPF record was created. The new platform's servers are not covered by `include:spf.protection.outlook.com`. Receiving servers correctly rejected the emails as unauthorized.

### Resolution
Update the SPF record to include the marketing platform:
```
v=spf1 include:spf.protection.outlook.com include:servers.mcsv.net -all
```

After updating:
1. Wait 30-60 minutes for DNS propagation
2. Send a test email through the marketing platform
3. Verify SPF pass in the recipient's email headers
4. Confirm emails are no longer landing in spam

### Prevention
- **Audit all third-party senders** before finalizing the SPF record
- **Document every service** that sends email on behalf of the domain
- **Monitor SPF lookup count** — stay below 10 to avoid "SPF PermError"
- **Review SPF record** whenever a new email service is added

---

> **Note:** This is a documented hypothetical scenario based on known Business Email Compromise (BEC) attack patterns. Header findings are illustrative — a live spoofing attempt was not generated in this lab environment.

## Scenario C — Spoofing Attempt Investigation

### Situation
A user receives an email appearing to be from the CEO requesting an urgent wire transfer to a new vendor. The request asks for secrecy and to bypass normal approval processes. The user is suspicious and escalates to IT.

### Symptoms
- Email appears to come from `ceo@northgsolutions.onmicrosoft.com`
- Request is unusual: urgent, financial, requests secrecy
- CEO is "traveling" and cannot be reached by phone per the email
- User did not initiate any vendor relationship mentioned in the email

### Investigation Steps
1. **Do NOT click any links or reply to the email**
2. Open the email headers — Outlook web: three dots → View message details
3. Check **From:** vs **Return-Path** — do they share the same domain?
4. Check **Authentication-Results** — did SPF, DKIM, and DMARC pass?
5. Run the sending domain/IP through **MXToolbox** and a **blacklist check**
6. Contact the CEO through a verified channel (phone, Teams) to confirm
7. Run the email through **Microsoft Defender → Explorer** for threat intelligence
8. Check **Exchange admin center → Message trace** for the email's origin

### Findings
```
From: CEO <ceo@northgsolutions.onmicrosoft.com>    ← what user sees
Return-Path: noreply@attacker-domain.com            ← what SPF checks

Authentication-Results:
  spf=fail (attacker-domain.com not authorized)
  dkim=fail (no valid DKIM signature from northgsolutions.onmicrosoft.com)
  dmarc=fail (neither SPF nor DKIM aligns with From: domain)

Received: from mail.attacker-domain.com (45.33.32.156)
  → This IP has no relationship to Microsoft's servers
```

The From: header was manually crafted to show the CEO's address. The actual sending server belongs to an external attacker. SPF, DKIM, and DMARC all fail because the attacker has no access to Northgate's authorized servers or signing keys.

### Root Cause
**Business Email Compromise (BEC) / CEO impersonation attack.** An external attacker spoofed the From: header to impersonate the CEO. The attack exploited the absence of a DMARC reject policy — without `p=reject`, the email was delivered despite failing all authentication checks. Microsoft's Spoof Intelligence flagged the email but delivered it with a warning rather than blocking it.

### Resolution
**Immediate:**
1. Alert the user — do not respond, do not transfer funds
2. Delete the email from the mailbox
3. Block the sending domain (`attacker-domain.com`) in Exchange anti-spam rules
4. Report to the security team for further investigation
5. Notify the CEO that their name is being used in an active phishing campaign

**Short-term:**
6. Enable DMARC with `p=quarantine` to begin blocking similar attempts
7. Train users on BEC recognition — urgency, secrecy, and financial requests are red flags
8. Implement an external email banner in Exchange to alert users when email comes from outside the organization

**Long-term:**
9. Move to a custom domain and implement DMARC `p=reject` for full enforcement
10. Enable Microsoft Defender's impersonation protection features
11. Implement a call-back verification policy for financial requests over a defined threshold

### Prevention
A published DMARC `p=reject` policy would have blocked this email before it reached the inbox. This is the clearest illustration of why DMARC enforcement matters — Spoof Intelligence helps, but a properly configured DMARC rejection policy is the definitive defense against domain impersonation attacks.

---

## Investigation Summary

| Scenario | Outcome | Key Lesson |
|----------|---------|------------|
| A — Authenticated email | ✅ Legitimate | SPF + DKIM passing confirms identity and integrity |
| B — SPF failure | ⚠️ Delivery issue | All authorized senders must be in the SPF record |
| C — Spoofing attempt | 🚨 Attack detected | DMARC p=reject is the definitive defense against impersonation |
