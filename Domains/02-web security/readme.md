# Web Application Vulnerabilities — Index

38 write-ups, ordered the way a real assessment actually flows: passive inspection first, then progressively deeper testing. Each file covers what it is, why it's vulnerable (with vulnerable-vs-secure code), how an attacker exploits it, technical impact, business impact (translated into financial/regulatory/reputational/legal terms), and mitigation.

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
Each file is self-contained and interview-ready — includes a one-line "interview answer" summarizing the business impact, and a "explaining it to a developer" section for communicating the fix to someone unfamiliar with the vulnerability.
