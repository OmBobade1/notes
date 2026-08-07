# Web Application Vulnerabilities — Index

38 write-ups, ordered the way a real assessment actually flows: passive inspection first, then progressively deeper testing. Each file covers what it is, why it's vulnerable (with vulnerable-vs-secure code), how an attacker exploits it, technical impact, business impact, and mitigation.

---

## 🧭 What to Test, Based on What You See

Not every check applies to every page. Before testing, look at what's actually *on* the page/feature, then only run the checks that apply. Use this table like a lookup — find the element, run those file numbers.

| If the page/feature has... | Test these files |
|---|---|
| **A login form** | `03` (auth/session), `04` (SQLi), `29` (rate limiting/brute force), `17` (if JWT-based), `36` (Host Header — password reset link) |
| **A search bar or any text input that echoes back results** | `04` (SQLi), `05` (XSS), `20` (LDAP/NoSQL if backend uses those) |
| **A file upload feature** | `08` (file upload/RCE), `11` (XXE — if uploading .docx/.xlsx/.svg/.xml), `38` (CSV injection — if it's later exported/reopened) |
| **A "forgot password" flow** | `03` (token predictability, expiry), `36` (Host Header manipulation) |
| **A URL parameter that redirects you somewhere (`?redirect=`, `?next=`)** | `16` (Open Redirect) |
| **A URL/API using a visible ID (`?id=101`, `/user/101`)** | `07` (IDOR / Broken Access Control) |
| **An admin panel or any role-restricted feature** | `23` (Broken Function-Level Authorization) |
| **Any form that changes data (transfer, settings, profile update)** | `06` (CSRF), `02` (check SameSite cookie flag) |
| **A "fetch from URL" or "import from URL" feature** | `09` (SSRF) |
| **A page-loading parameter (`?page=`, `?template=`, `?lang=`)** | `10` (LFI/RFI) |
| **A field that ends up in an XML/SOAP request** | `11` (XXE) |
| **A network-diagnostic or "ping this host" type feature** | `12` (Command Injection) |
| **A feature that saves/loads user session state as a serialized object** | `13` (Insecure Deserialization) |
| **A checkout, payment, or multi-step workflow** | `14` (Business Logic Flaws — price manipulation, race conditions, step-skipping) |
| **A dynamically-generated page using a template engine** | `21` (SSTI) |
| **Anything reflecting a header value back (redirects, cookies)** | `22` (CRLF Injection) |
| **An iframe embed, or `postMessage` used anywhere in page JS** | `19` (Clickjacking), `24` (postMessage) |
| **A JWT visible in requests (check dev tools → Application/Storage or the Authorization header)** | `17` (JWT vulnerabilities), `37` (check where it's stored — cookie vs localStorage) |
| **An API that responds with `Access-Control-Allow-Origin`** | `31` (CORS Misconfiguration) |
| **A WebSocket connection (`wss://` in Network tab)** | `34` (WebSocket Security) |
| **Any export-to-CSV/Excel feature** | `38` (CSV/Formula Injection) |
| **Third-party scripts loaded (analytics, chat widgets, ad tags)** | `28` (Malicious Third-Party Scripts / SRI) |
| **Any subdomain that looks abandoned or campaign-specific** | `32` (Subdomain Takeover) |

## 🧭 Check These On Every Single Page, Regardless of Content
Some checks aren't tied to a specific element — they're site-wide and worth running once per assessment, not per page:

- `01` — HTTP Security Headers
- `02` — Cookie flags (HttpOnly/Secure/SameSite)
- `15` — Security Misconfiguration (verbose errors, exposed files, directory listing)
- `18` — Known Vulnerable Components (check versions in page source/headers)
- `25` — Weak TLS/SSL Configuration
- `26` — Sensitive Data Exposure (check what's returned in API responses vs. what's displayed)
- `27` — Hardcoded Secrets (check page source/JS bundles)
- `29` — Application-Layer DoS (rate limiting generally, not just login)
- `30` — Insufficient Logging & Monitoring (can't test directly from outside, but worth noting in scope discussions)
- `33` — HTTP Request Smuggling (if the app sits behind a proxy/load balancer — usually true)

## 🧭 A Practical Testing Order (Putting It Together)
1. Passive pass first — headers, cookies, TLS, exposed files (`01`, `02`, `15`, `18`, `25`, `27`) — no interaction needed, just observation.
2. Map the site — list every distinct feature (login, search, upload, checkout, admin panel, API endpoints, third-party scripts).
3. For each feature, pull the matching row from the table above and test only those.
4. Close with the things that need the whole picture, not one page — business logic (`14`), CORS (`31`), request smuggling (`33`), subdomain enumeration (`32`).

---

## 📖 Full File Index

| # | File | Covers |
|---|---|---|
| 01 | `01-http-security-headers.md` | X-Frame-Options, CSP, X-Content-Type-Options, HSTS, Referrer-Policy |
| 02 | `02-cookie-security-attributes.md` | HttpOnly, Secure, SameSite |
| 03 | `03-authentication-session-mgmt.md` | Session fixation, timeout, weak tokens, password reset flaws, rate limiting |
| 04 | `04-sql-injection.md` | SQL Injection — payloads, parameterized queries |
| 05 | `05-xss.md` | XSS — reflected, stored, DOM-based |
| 06 | `06-csrf.md` | Cross-Site Request Forgery |
| 07 | `07-idor-broken-access-control.md` | IDOR / Broken Object-Level Access Control |
| 08 | `08-file-upload-vulnerabilities.md` | Unrestricted file upload → RCE |
| 09 | `09-ssrf.md` | Server-Side Request Forgery |
| 10 | `10-lfi-rfi.md` | Local & Remote File Inclusion |
| 11 | `11-xxe.md` | XML External Entity injection |
| 12 | `12-command-injection.md` | OS Command Injection |
| 13 | `13-insecure-deserialization.md` | Insecure Deserialization |
| 14 | `14-business-logic-flaws.md` | Price manipulation, race conditions, workflow skipping |
| 15 | `15-security-misconfiguration.md` | Verbose errors, directory listing, exposed `.git`/backups |
| 16 | `16-open-redirect.md` | Open Redirect |
| 17 | `17-jwt-token-vulnerabilities.md` | JWT algorithm confusion, weak secrets, expiration |
| 18 | `18-known-vulnerable-components.md` | Outdated dependencies (e.g. Log4Shell) |
| 19 | `19-clickjacking.md` | Clickjacking |
| 20 | `20-ldap-nosql-injection.md` | LDAP Injection, NoSQL Injection |
| 21 | `21-ssti.md` | Server-Side Template Injection |
| 22 | `22-crlf-header-injection.md` | CRLF Injection / HTTP Response Splitting |
| 23 | `23-broken-function-level-authorization.md` | Broken Function-Level Authorization |
| 24 | `24-insecure-postmessage.md` | Insecure `postMessage` handling |
| 25 | `25-weak-tls-ssl-configuration.md` | Weak TLS/SSL protocols and ciphers |
| 26 | `26-sensitive-data-exposure.md` | Weak password hashing, data retention, logging |
| 27 | `27-hardcoded-secrets.md` | Hardcoded credentials/API keys |
| 28 | `28-malicious-third-party-scripts-sri.md` | Supply-chain scripts, Magecart, Subresource Integrity |
| 29 | `29-application-layer-dos.md` | ReDoS, resource exhaustion, rate limiting |
| 30 | `30-insufficient-logging-monitoring.md` | Detection — logging, alerting, incident response |
| 31 | `31-cors-misconfiguration.md` | CORS Misconfiguration |
| 32 | `32-subdomain-takeover.md` | Subdomain Takeover |
| 33 | `33-http-request-smuggling.md` | HTTP Request Smuggling |
| 34 | `34-websocket-security.md` | WebSocket hijacking and security |
| 35 | `35-prototype-pollution.md` | Prototype Pollution (JavaScript) |
| 36 | `36-host-header-injection.md` | Host Header Injection |
| 37 | `37-insecure-client-side-token-storage.md` | localStorage vs. HttpOnly cookie token storage |
| 38 | `38-csv-formula-injection.md` | CSV/Formula Injection in exports |

## How to use this
Each file is self-contained and interview-ready — includes a one-line "interview answer" summarizing the business impact, and an "explaining it to a developer" section for communicating the fix to someone unfamiliar with the vulnerability.
