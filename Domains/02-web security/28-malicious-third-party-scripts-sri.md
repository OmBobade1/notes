# Malicious Third-Party Scripts & Missing Subresource Integrity (SRI)

## Why this comes next
Hardcoded secrets (file `27`) was about a company's own code leaking a secret. This is the reverse trust problem: a page loading *someone else's* script (analytics, ad tags, a CDN-hosted library) and trusting it completely — with no way to detect if that external script has been silently modified, whether through a compromised third-party vendor or a compromised CDN.

---

## What it is (in plain terms)
Modern websites commonly load JavaScript from external sources — analytics platforms, ad networks, payment widgets, or libraries hosted on a public CDN instead of the site's own server. Every one of these is a trust relationship: the site is agreeing to run whatever code that third party serves, on every visitor's browser, with the same level of access as the site's own first-party code. If that third party is compromised, or the CDN itself is compromised, the malicious code runs on the target site with full trust — no different from an XSS vulnerability, except the site never had a coding bug of its own.

## The real-world pattern: Magecart-style attacks
This isn't theoretical — **Magecart** is the name given to a long-running series of real attacks where criminal groups compromised third-party JavaScript libraries (often analytics or chat-widget scripts) used by e-commerce and payment checkout pages. Once compromised, the modified script would silently capture credit card details as customers typed them into the checkout form, sending the data to an attacker-controlled server — all while the site's own code remained completely unchanged and, on its own, perfectly secure. This affected major, well-known companies over several years, precisely because the vulnerability lived in a trusted dependency, not the site's own code.

## Why it exists — the real-life cause

```html
<!-- VULNERABLE — loads a third-party script with no verification of its integrity -->
<script src="https://third-party-analytics.com/tracker.js"></script>
```
If `third-party-analytics.com` is compromised (or an attacker gains access to modify what's served from that URL), the browser has no way to know the script it just downloaded and executed is different from what the site's developers originally intended to load — it simply runs whatever bytes arrive from that URL, with full page access.

```html
<!-- SECURE — Subresource Integrity: browser verifies the script matches an expected cryptographic hash -->
<script src="https://third-party-analytics.com/tracker.js"
        integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
        crossorigin="anonymous"></script>
```
With SRI, the browser computes a cryptographic hash of the script it actually receives and compares it against the expected hash specified in the `integrity` attribute. If the third-party script has been modified in any way — whether through a compromise or an unexpected update — the hash won't match, and the browser refuses to execute the script at all, failing safely instead of silently running altered code.

## How this actually plays out, step by step (from the attacker's side)
1. Identify a widely-used third-party script (analytics, chat widget, ad tag) used by the target site — or the CDN infrastructure hosting it.
2. Compromise that third party directly (their servers, their CDN account, or their own supply chain) rather than attacking the target site at all — this is significantly easier than finding a vulnerability in the target's own, likely better-defended, code.
3. Modify the script to include malicious functionality — commonly, capturing form input (card numbers, credentials) and silently exfiltrating it to an attacker-controlled endpoint.
4. Every site that loads the compromised script, without SRI verification, now unknowingly serves the malicious code to their own visitors — often for weeks or months before detection, since the site's own code and logs show nothing unusual.

## Technical Impact
- Full script execution with the same trust level as the site's own first-party code — equivalent in capability to a successful XSS attack, but without any flaw in the site's own code being the cause
- Particularly severe on payment/checkout pages, where a compromised script can directly capture and exfiltrate card data as it's typed

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | If a compromised third-party script runs on a banking checkout or payment page, it can silently capture card numbers or credentials directly — one of the most damaging possible outcomes, since it affects every customer who uses that page during the compromise window, not just one |
| **Regulatory / compliance** | PCI-DSS explicitly requires organizations to monitor and manage third-party scripts on payment pages (this became a more prominent, explicit requirement specifically in response to the Magecart pattern) — running unmonitored third-party scripts on payment flows without SRI is a direct compliance gap |
| **Reputational damage** | Because this pattern has a well-known name (Magecart) with well-documented major-company incidents, discovery of a similar issue is immediately recognizable and heavily covered in security press — it's a well-understood attack category, not an obscure edge case |
| **Legal liability** | Given how well-documented this attack pattern is, failing to apply the known mitigation (SRI, or vetting/monitoring third-party scripts) is difficult to defend as reasonable practice, especially for anything touching payment data |
| **Operational cost** | Detection is often slow specifically because the site's own code never changed — incident response requires auditing every third-party script in use, when each was added, and whether any showed unexpected behavior changes, which is a broader and slower investigation than a bug in first-party code |

**One-line interview answer:** *"Technically, this is about a site trusting third-party JavaScript completely, with no way to detect if that script has been silently modified — and it's not theoretical, it's the exact pattern behind real Magecart attacks that captured card data from major companies' checkout pages for months at a time. For a bank, the real danger is that the site's own code never has to have a bug at all — the compromise happens entirely in a trusted dependency, which makes it both harder to detect and explicitly called out in PCI-DSS as something that must be actively monitored."*

## Mitigation

1. **Subresource Integrity (SRI) on every third-party script (the real fix)** — as shown above; the browser refuses to execute anything that doesn't match the expected hash.
2. **Minimize third-party script usage on sensitive pages** — payment/checkout pages specifically should load as few external scripts as possible; every one is a trust relationship and additional attack surface.
3. **Content-Security-Policy `script-src` directive** (file `01`) — restricts which domains scripts can be loaded from at all, providing a second layer even without SRI on each individual script.
4. **Monitor third-party scripts for unexpected changes** — services exist specifically to alert if a monitored third-party script's content changes unexpectedly, catching a compromise even between manual reviews.
5. **Vet third-party vendors' own security practices** before integrating their scripts, particularly for anything touching payment or authentication flows.

## Explaining it to a developer
*"When we load a script from a third party without SRI, we're trusting that whatever that server sends us right now is the same thing every time, forever — with no way to verify it. If that third party is ever compromised, their script can be silently modified to do anything, including capturing what customers type into our own forms, and we'd have no way to detect it just by looking at our own code, since ours never changed. Adding an `integrity` attribute with the expected hash means the browser itself checks this for us and refuses to run anything that doesn't match — this is especially important on any page handling payment or login information."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Subresource Integrity (SRI) | Browser refuses to execute a modified/compromised script |
| Minimize third-party scripts on sensitive pages | Reduces overall trust surface |
| CSP `script-src` restriction | Limits which domains scripts can load from at all |
| Ongoing third-party script monitoring | Detects compromise between manual reviews |
| Vendor security vetting | Reduces likelihood of integrating an already-weak third party |
