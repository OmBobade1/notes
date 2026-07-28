# JWT (JSON Web Token) Vulnerabilities

## Why this comes next
This connects directly back to file `03` (Authentication & Session Management) — JWTs are one specific, very common way modern applications implement sessions, especially in APIs. Instead of the server storing session state and handing the client an opaque ID (a traditional session cookie), the server issues a self-contained token that carries the user's identity and claims directly inside it, cryptographically signed. That signature is exactly where most JWT vulnerabilities live.

---

## What a JWT actually is
A JWT has three parts, separated by dots: `header.payload.signature` — e.g. `eyJhbGc...​.eyJ1c2Vy...​.SflKxwRJ...`. The header and payload are just base64-encoded JSON (readable by anyone, not encrypted — this matters, see below). The signature is what's supposed to prove the token wasn't tampered with, generated using a secret key only the server should know.

```json
// Header (base64-decoded)
{ "alg": "HS256", "typ": "JWT" }

// Payload (base64-decoded) — visible to ANYONE who has the token, not secret
{ "user_id": 1001, "role": "customer", "exp": 1735689600 }
```

---

## Algorithm Confusion (`alg: none`)

**What it is:** JWT supports an `alg: none` option, meaning "this token has no signature at all." Some poorly-implemented verification libraries will accept a token with `alg: none` and skip signature verification entirely, treating whatever the payload claims as trusted.

**Example:**
```python
# VULNERABLE — verification library configured to accept whatever algorithm the token claims
decoded = jwt.decode(token, options={"verify_signature": False})  # or an outdated library defaulting unsafely
```
An attacker takes any valid token, decodes the payload (trivial — it's just base64, not encrypted), changes `"role": "customer"` to `"role": "admin"`, re-encodes it, sets the header to `alg: none`, and removes the signature section entirely. If the server's verification logic honors the `none` algorithm, this forged, unsigned token is accepted as valid.

```python
# SECURE — explicitly specify and enforce the expected algorithm
decoded = jwt.decode(token, secret_key, algorithms=["HS256"])  # explicitly reject anything else, including "none"
```

**Business Impact:** Privilege escalation via a trivially forged token — an attacker can potentially grant themselves admin access simply by editing readable JSON and choosing a "no signature" mode the server should never have accepted. This is one of the most severe possible authentication bypasses precisely because it requires no cryptographic attack at all.

---

## Algorithm Confusion (RS256 → HS256)

**What it is:** A more advanced variant. Some setups use asymmetric signing (`RS256` — a private key signs, a public key verifies). The public key is, by design, actually public — often published, or embedded in the application itself. If the server's verification code doesn't strictly enforce which algorithm is expected, an attacker can craft a token using the symmetric `HS256` algorithm, using the *public* key (which they legitimately have access to) as the "secret" — and some vulnerable libraries will incorrectly verify it as valid.

**Business Impact:** Same outcome as the `alg: none` issue — full authentication bypass and privilege escalation — but requires a specific implementation flaw where the algorithm isn't strictly pinned, making it a subtler variant that's easy to miss in code review.

**Mitigation for both:** Always explicitly specify and enforce the expected algorithm on the verification side (never trust the algorithm named inside the token itself), and never allow `none` as an accepted algorithm under any circumstance.

---

## Weak / Guessable Signing Secret

**What it is:** The secret key used to sign `HS256` tokens is short, a common default, or hardcoded/leaked in source code — allowing an attacker to brute-force or simply discover it, then forge arbitrarily valid tokens.

**Business Impact:** Once the secret is known, an attacker can generate a perfectly valid, correctly-signed token claiming to be any user — including an admin — completely bypassing authentication with a token indistinguishable from a real one, since it *is* cryptographically legitimate once the secret is compromised.

**Mitigation:** Use a long, cryptographically random secret (or better, asymmetric signing with the private key kept genuinely private and never embedded in any client-accessible code), and treat it with the same protection as a database password — never in source code, never in a public repository.

---

## No Expiration / Excessively Long Expiration

**What it is:** A JWT with no `exp` claim, or one set far longer than necessary, remains valid indefinitely (or for a very long time) — meaning a stolen token stays useful to an attacker for that entire window, and unlike a traditional session, a JWT typically can't be individually revoked server-side once issued (this is the core trade-off of the "stateless" JWT design).

**Business Impact:** A stolen token (via XSS, a leaked log, or any other exposure) remains a fully valid credential for as long as it's set to last — for a banking API, a long-lived token compromised once means extended unauthorized access with no easy way to cut it off short of rotating the signing secret entirely (which invalidates *every* issued token, not just the compromised one).

**Mitigation:** Keep JWT expiration short (minutes, not days), paired with a separate, longer-lived refresh token that's stored more securely (e.g. `HttpOnly` cookie) and can be revoked server-side via a maintained revocation list if compromise is suspected.

---

## Sensitive Data in the Payload

**What it is:** Putting sensitive information directly in the JWT payload, forgetting that the payload is only base64-*encoded*, not encrypted — anyone holding the token (including the end user themselves, or anyone who intercepts it) can trivially decode and read it in full.

**Business Impact:** Depending on what's included, this can directly leak account numbers, internal user IDs useful for further IDOR attempts (file `07`), or other data never meant to be client-visible — a silent information disclosure issue distinct from, but related to, the token-forgery risks above.

**Mitigation:** Treat the JWT payload as visible to the end user at all times — only include the minimum claims genuinely needed (user ID, role, expiration), never sensitive personal or financial data directly.

---

## Quick-reference table

| Issue | Root Cause | Impact |
|---|---|---|
| `alg: none` accepted | Verification logic honors an unsigned token | Full authentication bypass |
| RS256/HS256 confusion | Algorithm not strictly enforced server-side | Full authentication bypass using the public key |
| Weak signing secret | Short/default/leaked secret key | Attacker can forge arbitrarily valid tokens |
| No/long expiration | No `exp` claim or excessively long validity | Stolen tokens remain valid for extended periods |
| Sensitive payload data | Payload is encoded, not encrypted, and assumed private incorrectly | Information disclosure to anyone holding the token |

## Explaining it to a developer
*"JWTs are only as trustworthy as the verification step — the payload itself is just readable JSON, not encrypted, so anyone can see or even edit it. What actually prevents a forged token from being accepted is strict signature verification with an explicitly pinned algorithm and a strong secret. If the verification code trusts the algorithm named inside the token itself, or accepts an unsigned token, or uses a weak secret, none of that matters — the token can be forged to claim to be anyone, including an admin, and the server has no way to know."*
