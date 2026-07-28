# Weak TLS/SSL Configuration

## Why this comes next
Everything so far has been about application-layer flaws. This one sits a level below — it's about whether the encrypted connection itself (HTTPS) is actually configured strongly, independent of whether the application code is perfect. This connects directly back to the `Strict-Transport-Security` header from file `01` — HSTS forces HTTPS to be used, but this file is about whether the HTTPS being used is actually trustworthy.

---

## What it is (in plain terms)
HTTPS isn't just "on or off" — it involves a negotiation between the browser and server over which TLS *version* and which *cipher suite* (the actual encryption algorithm) to use for the connection. If a server is configured to still accept old, broken protocol versions or weak ciphers (often left enabled for legacy compatibility reasons that no longer apply), an attacker positioned on the network can potentially force the connection to downgrade to the weakest option both sides technically support, then break that weaker encryption.

## The specific weaknesses

**Outdated protocol versions:**
- **SSLv2, SSLv3** — fundamentally broken, should never be enabled under any circumstance
- **TLS 1.0, TLS 1.1** — deprecated by major browsers and PCI-DSS as of 2020/2021, contain known weaknesses (e.g. BEAST attack against TLS 1.0)
- **TLS 1.2** — currently acceptable, but should be considered a floor, not a target
- **TLS 1.3** — current best practice, removes support for weak ciphers entirely by design and is faster due to a simplified handshake

**Weak cipher suites:**
- Ciphers using **RC4** — has known statistical biases that can be exploited to recover plaintext
- Ciphers using **DES/3DES** — small key size, vulnerable to practical attacks (e.g. the "Sweet32" attack against 3DES)
- **Export-grade ciphers** (deliberately weakened, historically for export regulation compliance) — trivially breakable by modern computing standards
- Ciphers without **forward secrecy** (not using ephemeral Diffie-Hellman key exchange) — if the server's private key is ever compromised in the future, *all* previously recorded encrypted traffic can be decrypted retroactively

**Certificate issues:**
- Self-signed certificates in production (browsers will show explicit warnings, training customers to click through security warnings — a dangerous habit)
- Expired certificates
- Certificates using outdated signature algorithms (e.g. SHA-1, which has known collision vulnerabilities)

## How an attacker actually does it, step by step
1. Scan the target's TLS configuration using a tool like `testssl.sh`, `sslyze`, or Nmap's `ssl-enum-ciphers` script — these report exactly which protocol versions and cipher suites the server accepts.
2. If weak protocols/ciphers are accepted alongside strong ones, an attacker positioned on the network path (public Wi-Fi, compromised router) can attempt a **downgrade attack** — interfering with the negotiation to force the connection down to the weakest mutually-supported option.
3. Once downgraded to a genuinely weak cipher, apply the specific known attack against that cipher (e.g. Sweet32 against 3DES) to recover plaintext from the supposedly encrypted traffic.

## Technical Impact
- Interception and decryption of traffic that should have been protected by encryption — session tokens, credentials, transaction details, personal data all potentially exposed in transit
- This is the actual mechanism behind a real Man-in-the-Middle attack against HTTPS traffic, rather than the simpler "no HTTPS at all" scenario already covered under HSTS in file `01`

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | Successfully downgrading and breaking encryption on a banking session exposes session tokens or credentials directly, leading to the same account-takeover and fraud outcomes as other interception-based attacks already covered |
| **Regulatory / compliance** | PCI-DSS (directly applicable to any organization processing card payments) explicitly mandates disabling TLS 1.0/1.1 and requiring TLS 1.2 minimum — running outdated protocols is a direct, unambiguous compliance violation, not a judgment call |
| **Reputational damage** | Automated scanning tools (including ones used by security researchers and journalists) regularly scan the internet for weak TLS configurations on financial institutions specifically — public "name and shame" reports about a bank's weak encryption configuration are a real, recurring category of security news coverage |
| **Legal liability** | A known, standards-mandated configuration (TLS 1.2 minimum, strong ciphers only) that was simply never applied is very difficult to argue as reasonable practice after the fact |
| **Operational cost** | Generally low remediation cost (a server configuration change) once identified — but if it enabled an actual interception incident, the cost matches any other credential-theft/account-takeover incident already discussed |

**One-line interview answer:** *"Technically, weak TLS configuration means the server still accepts outdated protocol versions or weak ciphers that can be downgraded to and then broken by an attacker on the network path. For a bank, this isn't just a technical hygiene issue — TLS 1.2 minimum is an explicit PCI-DSS requirement, so running anything weaker is both a direct fraud-enabling risk and an unambiguous compliance violation, not a debatable finding."*

## Mitigation

1. **Disable SSLv2, SSLv3, TLS 1.0, and TLS 1.1 entirely** — TLS 1.2 as the minimum accepted version, TLS 1.3 preferred wherever supported by the server/infrastructure.
2. **Disable weak cipher suites explicitly** — remove RC4, DES/3DES, export-grade, and non-forward-secret ciphers from the accepted list; prefer AEAD ciphers (like AES-GCM) with ephemeral key exchange.
3. **Use a properly signed, currently valid certificate** from a trusted Certificate Authority — never self-signed in production, and monitor expiration dates with automated renewal (e.g. Let's Encrypt with auto-renewal) to prevent lapses.
4. **Regularly scan your own configuration** with tools like `testssl.sh` or the Qualys SSL Labs online scanner, since browser/standards requirements evolve over time (what was acceptable a few years ago may no longer be).
5. **Set the HSTS header** (file `01`) alongside strong TLS configuration — HSTS ensures HTTPS is always used; strong TLS configuration ensures that HTTPS is actually trustworthy once it is.

## Explaining it to a developer
*"HTTPS being enabled doesn't automatically mean the encryption is strong — the server might still be configured to accept old, broken protocol versions or weak ciphers for legacy compatibility reasons that usually don't apply anymore. An attacker on the same network can potentially force the connection down to the weakest option we still accept, then break that specifically. The fix is a server configuration change — disable anything older than TLS 1.2, and remove known-weak cipher suites from what the server offers — not a code change, but just as important as one."*

## Quick-reference table

| Mitigation | What it addresses |
|---|---|
| Disable SSLv2/SSLv3/TLS 1.0/1.1 | Removes fundamentally broken/deprecated protocol versions |
| Disable weak ciphers (RC4, DES/3DES, export-grade) | Removes practically-breakable encryption options |
| Valid CA-signed certificate, monitored expiration | Avoids browser warnings and lapses |
| Regular scanning (testssl.sh, SSL Labs) | Keeps configuration current as standards evolve |
| HSTS header (file 01) | Ensures HTTPS is always used, complementing strong TLS config |
