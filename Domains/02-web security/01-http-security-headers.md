# HTTP Security Headers

## Why start here
Before testing for any specific vulnerability, the first thing to check when you intercept a request/response (in Burp Suite or your browser's dev tools) is the **response headers**. Missing or misconfigured security headers don't always cause a breach by themselves, but they remove layers of defense that would otherwise limit or block an attack — so checking them first, in order, mirrors how a real assessment actually flows: intercept → check headers → then test deeper (SQLi, XSS, etc.).

---

## X-Frame-Options

**What it is:** Tells the browser whether this page is allowed to be displayed inside a `<iframe>` on another site.

**Values and what they mean:**
- `DENY` — page can never be framed by anyone, including the site itself
- `SAMEORIGIN` — page can only be framed by pages on the same domain
- `ALLOW-FROM https://example.com` — (largely obsolete now) allow one specific external domain to frame it

**Why it matters:** Without this header, an attacker can embed your page inside an invisible `<iframe>` on their own malicious site, and trick a logged-in user into clicking buttons they think belong to a different, harmless-looking page — while actually clicking real buttons on your real site underneath (e.g. "Transfer funds" or "Change password"). This attack is called **clickjacking**.

**Impact if missing:**
- Technical: attacker can overlay invisible UI elements to trick clicks
- Business: for a banking app, clickjacking could trick a logged-in user into unknowingly authorizing a transaction — direct financial loss to the customer, and the bank held responsible for inadequate protection

**Mitigation:** Set `X-Frame-Options: DENY` (or `SAMEORIGIN` if legitimate same-site framing is needed) on every response. Modern equivalent: the `frame-ancestors` directive in Content-Security-Policy (see below) supersedes this header and offers more granular control.

**Explaining it to a developer:** *"Right now any external site can put our page inside an invisible frame and trick users into clicking real buttons on our real site without knowing it. Adding `X-Frame-Options: DENY` tells the browser to simply refuse to display our page inside any frame at all — one line, no functionality lost unless we intentionally need to be framed somewhere."*

---

## Content-Security-Policy (CSP)

**What it is:** A header that tells the browser exactly which sources (scripts, styles, images, fonts, frames) are allowed to load and execute on the page — everything else is blocked by the browser itself, even if it somehow got injected into the page.

**Example value:**
```
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted-cdn.com; frame-ancestors 'none'
```
- `default-src 'self'` — by default, only load resources from our own domain
- `script-src 'self' https://trusted-cdn.com` — scripts allowed only from our own domain and one named trusted CDN
- `frame-ancestors 'none'` — nobody can frame this page (modern replacement for X-Frame-Options)

**Why it matters:** This is one of the strongest defenses against Cross-Site Scripting (XSS). Even if an attacker manages to inject a `<script>` tag into the page (via a vulnerable input field), a properly configured CSP tells the browser to simply refuse to execute any script that isn't from an explicitly allowed source — so the injected script never runs, even though the injection technically succeeded.

**Impact if missing:**
- Technical: any successful script injection (XSS) executes with full run of the page — no browser-level backstop
- Business: an injected script on a banking login page could silently capture and exfiltrate typed credentials to an attacker-controlled server — direct account takeover and fraud risk, and a regulatory incident once discovered

**Mitigation:** Define a CSP starting from `default-src 'self'` and explicitly allow only the specific external sources genuinely needed (fonts, analytics, CDNs) — never use `'unsafe-inline'` or `'unsafe-eval'` unless there is no alternative, since those two directives effectively disable CSP's main XSS protection.

**Explaining it to a developer:** *"Even if a script gets injected somewhere due to a bug, CSP tells the browser 'only run scripts I explicitly listed as trusted' — so an injected script from an attacker just won't execute, because it's not on the allowed list. Think of it as a second independent lock, even if the first lock (input validation) fails."*

---

## X-Content-Type-Options

**What it is:** Tells the browser not to try to guess ("sniff") a file's type based on its content — only trust the `Content-Type` header the server actually sent.

**Value:** `X-Content-Type-Options: nosniff` (this is the only valid value)

**Why it matters:** Without this header, a browser might look at a file the server says is a harmless `.txt` or image, decide on its own that it "looks like" JavaScript or HTML based on content, and execute it as such — letting an attacker upload something disguised as an image that actually runs as a script.

**Impact if missing:**
- Technical: enables MIME-sniffing-based attacks, often combined with a file upload feature, to achieve script execution
- Business: if a banking app allows document/photo uploads (e.g. KYC document upload), this becomes a path to stored XSS via a disguised file — same account-takeover and regulatory risk as above

**Mitigation:** Set `X-Content-Type-Options: nosniff` on every response, and ensure the server always sends the correct, explicit `Content-Type` for every file it serves.

---

## Strict-Transport-Security (HSTS)

**What it is:** Tells the browser "always use HTTPS for this site, never fall back to plain HTTP, for the next N seconds" — even if the user types `http://` or clicks an old `http://` link.

**Example value:**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```
- `max-age=31536000` — enforce HTTPS-only for 1 year from this response
- `includeSubDomains` — applies the same rule to all subdomains
- `preload` — allows the domain to be included in browsers' built-in HSTS preload list, enforced before the user ever visits once

**Why it matters:** Without HSTS, a user's very first connection attempt (or an old bookmark/link) can go out over plain HTTP, giving an attacker on the same network (public Wi-Fi, compromised router) the chance to intercept and downgrade the connection — a **man-in-the-middle (MITM)** attack — before HTTPS ever kicks in.

**Impact if missing:**
- Technical: connection can be silently downgraded to HTTP on an untrusted network, exposing all data in transit
- Business: for banking, this means session tokens, credentials, or transaction data could be intercepted on something as simple as public airport Wi-Fi — direct fraud exposure and a severe regulatory finding, since encryption-in-transit is a baseline compliance requirement

**Mitigation:** Set the HSTS header on every HTTPS response, use a long `max-age`, and submit the domain to the browser HSTS preload list for maximum protection from the very first request.

---

## Referrer-Policy

**What it is:** Controls how much of the current page's URL gets sent to the next site when a user clicks a link away from your page (in the `Referer` header of the following request).

**Example value:** `Referrer-Policy: strict-origin-when-cross-origin` (send full URL to same-origin requests, only the domain to cross-origin ones, nothing over plain HTTP)

**Why it matters:** If your URLs contain sensitive data (e.g. `https://bank.com/reset-password?token=abc123`), and a page with that URL contains any link to an external site (even an image hosted elsewhere), the full URL — including that token — can leak to the external site via the Referer header.

**Impact if missing:**
- Technical: sensitive URL parameters (tokens, session identifiers) leak to third-party domains
- Business: a leaked password-reset token could let an attacker take over an account without ever breaching the bank's own systems directly — reputational and regulatory exposure from a seemingly small oversight

**Mitigation:** Set `Referrer-Policy: strict-origin-when-cross-origin` or stricter, and avoid putting sensitive tokens directly in URLs at all (use POST body or headers instead) as a deeper fix.

---

## Quick-reference table (for fast recall in an interview)

| Header | One-line purpose | Missing = |
|---|---|---|
| X-Frame-Options | Blocks the page from being framed | Clickjacking |
| Content-Security-Policy | Whitelists what scripts/resources can run | XSS runs unrestricted |
| X-Content-Type-Options | Stops MIME-type guessing | Disguised file execution |
| Strict-Transport-Security | Forces HTTPS always | MITM downgrade attacks |
| Referrer-Policy | Controls URL leakage on outbound links | Token/URL leakage to third parties |
