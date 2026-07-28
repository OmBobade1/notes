# Subdomain Takeover

## Why this comes next
This is a DNS-level vulnerability rather than an application-code one — a completely different layer from anything covered so far, but a genuinely common real-world finding in bug bounty programs specifically, since large organizations accumulate forgotten subdomains over time.

---

## What it is (in plain terms)
Organizations often point a subdomain (e.g. `promo.bank.com`) at a third-party cloud service (a GitHub Pages site, an AWS S3 bucket, a Heroku app) via a CNAME DNS record. If that third-party resource is later deleted or the service is cancelled — but the DNS record pointing to it is never removed — the subdomain still resolves to that provider, just to nothing (a "dangling" record). An attacker can then register that same resource name on the same third-party service themselves, effectively claiming ownership of content served under the organization's own trusted subdomain.

## Why it exists — the real-life cause

**The setup (how it becomes vulnerable):**
```
# DNS record for promo.bank.com
promo.bank.com.   CNAME   bank-promo-campaign.github.io.
```
A marketing team sets up a promotional microsite hosted on GitHub Pages, pointed to by this CNAME. Months later, the campaign ends, the GitHub repository is deleted — but nobody remembers to also remove the DNS CNAME record, since DNS configuration and application/hosting decisions are often managed by completely different teams with no coordinated cleanup process.

**The exploitation:**
The CNAME record `promo.bank.com` still points to `bank-promo-campaign.github.io` — but that GitHub Pages site no longer exists, meaning the name `bank-promo-campaign` is now available for *anyone* to register on GitHub Pages. An attacker creates their own GitHub account, names a new repository/Pages site exactly `bank-promo-campaign`, and GitHub now serves *the attacker's content* whenever anyone visits `promo.bank.com` — because the DNS record still points there, and GitHub has no way to know the original owner's intent, only that the name is now claimed by whoever registered it.

## How an attacker actually does it, step by step
1. Enumerate an organization's subdomains (using tools like `subfinder`, `amass`, or certificate transparency logs, which record every subdomain a certificate was ever issued for).
2. For each discovered subdomain, check its DNS record type — CNAME records pointing to common third-party services (GitHub Pages, AWS S3, Heroku, Azure, Shopify, and dozens of others) are the primary targets.
3. Attempt to resolve the subdomain — services like GitHub Pages, Heroku, and misconfigured S3 buckets often return a distinctive "no such app/page" error message when the CNAME points to a deprovisioned resource, which automated tools (e.g. `Can-I-Take-Over-XYZ`, a maintained public list of vulnerable service fingerprints) specifically check for.
4. If confirmed dangling, register the exact matching resource name on the third-party service — the attacker's content is now served under the organization's own trusted subdomain and, critically, under their real TLS certificate if HTTPS is configured, making it appear completely legitimate.

## Technical Impact
- Full control over content served under a legitimate, trusted subdomain of the organization
- Can be used to host convincing phishing pages (identical concern to Open Redirect in file `16`, but arguably more convincing since the entire URL, not just the domain prefix, is genuinely the organization's own)
- Can potentially be used to steal cookies if the cookie's scope (`Domain` attribute) includes the parent domain broadly enough to be readable from the takeover subdomain

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | A convincingly hosted phishing page on a genuinely trusted subdomain (not even a look-alike domain, but the real one) can be highly effective at harvesting customer credentials, leading to the same account-takeover and fraud outcomes covered elsewhere in this series |
| **Regulatory / compliance** | Demonstrates a gap in organizational asset management/decommissioning process — auditors increasingly ask specifically about subdomain/DNS hygiene as part of external attack surface assessments |
| **Reputational damage** | Discovery (often by an external security researcher submitting a bug bounty report) that an organization's own subdomain was serving attacker content is a distinctly embarrassing, easily-explainable-to-media finding, since the root cause (forgotten DNS record) is simple to describe |
| **Legal liability** | Weaker direct liability than an active code vulnerability, but contributes if it can be shown the takeover was used in a documented phishing campaign against customers |
| **Operational cost** | Remediation itself is simple (remove the dangling DNS record) once found — the actual cost is in the process gap: establishing a reliable decommissioning checklist that ties DNS record removal to resource deprovisioning across every team that manages either |

**One-line interview answer:** *"Technically, subdomain takeover happens when a DNS record still points to a third-party resource that's been deleted, letting an attacker register that same resource name themselves and serve content under the organization's own trusted subdomain. For a bank, the business impact is specifically phishing effectiveness — a phishing page on the actual real subdomain, not a look-alike, is far more convincing than typical phishing, and it usually reflects a gap in the decommissioning process between whichever team manages DNS and whichever team manages the third-party hosting resource."*

## Mitigation

1. **Maintain an accurate, current DNS record inventory** — know what every subdomain points to and why, ideally with an owner assigned to each entry.
2. **Remove DNS records as part of the decommissioning process** — whenever a third-party resource (a hosting service, a cloud bucket) is deprovisioned, removing the corresponding DNS record should be an explicit, checklist-enforced step, not an afterthought left to memory.
3. **Regularly scan your own subdomains for dangling records** — tools that check for known vulnerable-service fingerprints can be run proactively against your own domain inventory, the same way an attacker would, catching the issue before it's found externally.
4. **Use DNS record ownership verification features** where the third-party service offers them (some services require a verification token in the DNS record itself, which naturally prevents an unrelated party from simply claiming an abandoned name).

## Explaining it to a developer
*"This subdomain's DNS record still points to a third-party service, but the actual resource on that service was deleted a while ago — which means the name is now available for anyone else to register there. Since the DNS record is still pointing at it, whoever claims that name now controls what gets served under our own subdomain. The fix here is simple — remove the DNS record since it's no longer needed — but the bigger fix is making sure removing a DNS record is always part of the checklist whenever we decommission anything it points to, so this doesn't keep happening as things get shut down over time."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Accurate DNS record inventory with ownership | Prevents records from being forgotten in the first place |
| DNS removal as part of decommissioning checklist | Closes the gap between resource deletion and DNS cleanup |
| Proactive scanning for dangling records | Catches the issue internally before external discovery |
| DNS ownership verification features | Prevents unrelated parties from claiming an abandoned name |
