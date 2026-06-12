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

> **Evidence:** This scenario was reproduced in the lab. SPF failure was intentionally triggered by temporarily modifying the SPF record to `v=spf1 -all`, blocking all authorized senders including Microsoft Exchange Online servers.

### Situation
A user reports that emails sent to external contacts are landing in spam folders or being rejected. No bounce messages initially. Recipients confirm emails arrive flagged as suspicious or do not arrive at all.

### Symptoms
- Outbound emails from `jmartinez@northgsolutions.com` not reaching Gmail inbox
- No error on the sending side — Outlook shows email as sent
- Recipient spam filter notes SPF authentication failure
- Issue triggered after SPF record was modified

### Investigation Steps
1. Ask the affected user which email address they sent from and to which recipient
2. Go to **Exchange admin center → Mail flow → Message trace** — confirm the email was sent through Exchange Online
3. Run **MXToolbox SPF Lookup** on `northgsolutions.com` — review the current record
4. Check if the SPF record is correct: `v=spf1 include:spf.protection.outlook.com -all`
5. Ask the recipient to forward the email with headers showing the SPF result
6. Analyze the Authentication-Results header for SPF verdict

### Findings

SPF check failed because the record `v=spf1 -all` blocks all senders — including Microsoft's Exchange Online servers. The sending IP `2a01:111:f403:c107::3` is a legitimate Microsoft Exchange Online server but was not authorized by the misconfigured SPF record. The sending IP was not found in any authorized range.

### Root Cause
**Misconfigured SPF record.** The `-all` mechanism without any preceding `include:` statement rejects all senders. Microsoft Exchange Online's sending servers (`spf.protection.outlook.com`) were not listed as authorized, causing every outbound email to fail SPF validation at the recipient's server.

### Resolution
1. Go to **Cloudflare DNS** → find the SPF TXT record → Edit
2. Restore the correct value: v=spf1 include:spf.protection.outlook.com -all
3. Save → wait 5-10 minutes for DNS propagation
4. Run **MXToolbox SPF Lookup** → confirm single clean record
5. Send test email → verify `spf=pass` in recipient headers

### Prevention
- Never modify the SPF record without first documenting the current value
- Always test with MXToolbox after any SPF change before considering it complete
- Use `-all` (hard fail) for full enforcement — but only after confirming all authorized senders are listed
- If using third-party senders (Mailchimp, Salesforce, HubSpot), add their include mechanisms before deploying `-all`
- Monitor SPF lookup count — RFC limit is 10 DNS lookups per SPF evaluation

## Scenario C — Spoofing Attempt Investigation

> **Evidence:** This scenario was reproduced in the lab. A DMARC rejection was 
triggered by temporarily setting the DMARC policy to `p=reject` while SPF and 
DKIM were broken, causing Gmail to reject the email entirely and return a bounce 
message to the sender.

### Situation
A user receives a bounce notification stating their email was rejected by the 
recipient's server due to the domain's DMARC policy. In a real-world context, 
this same mechanism blocks attackers attempting to spoof a domain that has DMARC 
enforcement enabled.

### Symptoms
- Email sent from `jmartinez@northgsolutions.com` to external Gmail was rejected
- Bounce message received in Outlook: "mx.google.com rejected your message"
- Error states: "Unauthenticated email from northgsolutions.com is not accepted 
due to domain's DMARC policy"
- No email delivered to the recipient

### Investigation Steps
1. Review the bounce message in the sender's Outlook inbox
2. Identify the rejecting server — in this case `mx.google.com`
3. Check Authentication-Results in the bounce headers for SPF, DKIM, and DMARC verdicts
4. Verify DMARC policy is set to `p=reject` in Cloudflare DNS
5. Run **MXToolbox DMARC Lookup** on `northgsolutions.com` — confirm active policy
6. Determine whether SPF and DKIM are passing or failing

### Findings

<img width="1036" height="551" alt="phase-d-dmarc-fail" src="https://github.com/user-attachments/assets/314f1dcc-0253-4435-a354-174fca094139" />

The bounce message above was received in Jordan Martinez's Outlook inbox after 
sending to Gmail with DMARC p=reject active and SPF/DKIM broken. Gmail's 
receiving server evaluated the email against the published DMARC record and 
rejected it with the following error:

"Unauthenticated email from northgsolutions.com is not accepted due to 
domain's DMARC policy."

DMARC policy at time of rejection:
v=DMARC1; p=reject; rua=mailto:dmarc@northgsolutions.com

Authentication failure:
- SPF: fail — sending IP not authorized (v=spf1 -all)
- DKIM: not signing (disabled during test)
- DMARC: fail → p=reject → email blocked entirely

### Root Cause
**DMARC p=reject enforcement blocked unauthenticated email.** With SPF broken 
(`v=spf1 -all`) and DKIM disabled, neither authentication mechanism passed. The 
DMARC policy of `p=reject` instructed receiving servers to block the message 
entirely rather than deliver it. This is the intended behavior — any email that 
cannot be authenticated by SPF or DKIM is rejected outright.

In a real spoofing scenario, an attacker sending email claiming to be from 
`northgsolutions.com` using unauthorized servers would receive this same 
rejection — protecting recipients from impersonation attacks.

### Resolution
**For the lab test:**
1. Restore SPF record: `v=spf1 include:spf.protection.outlook.com -all`
2. Re-enable DKIM in Microsoft Defender portal for `northgsolutions.com`
3. Confirm legitimate email flows normally after restoration

**In a real spoofing incident:**
1. Confirm the rejection was triggered by an attacker, not a misconfiguration
2. Check Exchange admin center → Message trace for any related activity
3. Alert users that the domain is being targeted
4. Verify DMARC, SPF, and DKIM are all correctly configured
5. Consider enabling Microsoft Defender impersonation protection

### Prevention
- A published DMARC `p=reject` policy is the definitive defense against domain spoofing
- Without DMARC enforcement, SPF and DKIM alone cannot prevent spoofed emails 
from reaching inboxes
- Progressive deployment (none → quarantine → reject) ensures legitimate mail is 
not disrupted during enforcement rollout
- Monitor DMARC aggregate reports (`rua`) to identify unauthorized senders 
attempting to use your domain
---

## Investigation Summary

| Scenario | Outcome | Key Lesson |
|----------|---------|------------|
| A — Authenticated email | ✅ Legitimate | SPF + DKIM passing confirms identity and integrity |
| B — SPF failure | ⚠️ Delivery issue | All authorized senders must be in the SPF record |
| C — Spoofing attempt | 🚨 Attack detected | DMARC p=reject is the definitive defense against impersonation |
