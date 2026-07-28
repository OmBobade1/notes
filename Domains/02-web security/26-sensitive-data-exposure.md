# Sensitive Data Exposure

## Why this comes next
Weak TLS (file `25`) was about data being exposed *in transit*. This file covers the broader category: sensitive data left inadequately protected either in transit *or* at rest — a category that spans several of the specific mechanisms already covered (weak TLS, verbose errors, exposed config files) plus genuinely new ground: what happens to data once it's actually stored.

---

## What it is (in plain terms)
Sensitive data — passwords, card numbers, personal identifiable information (PII), session tokens — needs protection both while moving across the network (in transit) and while sitting in a database or file (at rest). This category covers the ways that protection commonly fails, beyond the specific TLS/transit issue already detailed separately.

## Unencrypted or Weakly-Hashed Data at Rest

**What it is:** Storing sensitive data in plain text in the database, or using a weak/fast hashing algorithm for passwords instead of a purpose-built slow one.

**Example:**
```python
# VULNERABLE — plain MD5 hash, no salt, extremely fast to brute-force
import hashlib
password_hash = hashlib.md5(password.encode()).hexdigest()
```
MD5 (and SHA-1/SHA-256 used alone) are *fast* hash functions — designed for speed, which is exactly the wrong property for password storage. An attacker who obtains the database can compute billions of MD5 hashes per second on modern hardware, making brute-forcing even complex passwords from their hash entirely feasible, especially using precomputed rainbow tables for common passwords.

```python
# SECURE — bcrypt: a slow, purpose-built password hashing algorithm with built-in salting
import bcrypt
password_hash = bcrypt.hashpw(password.encode(), bcrypt.gensalt())
```
`bcrypt` (or `argon2`, `scrypt`) is deliberately slow and includes a unique salt automatically, making large-scale brute-forcing computationally impractical even with significant attacker resources — this is the actual, purpose-built solution, not a general-purpose hash function repurposed for something it was never designed for.

**Business Impact:** If a database breach occurs (through any of the vulnerabilities covered earlier — SQLi, misconfigured cloud storage, etc.), passwords hashed with a weak/fast algorithm are realistically crackable at scale, converting "we had a breach" into "every customer's actual password is now known" — a dramatically worse outcome than the same breach against properly bcrypt-hashed passwords.

## Excessive Data Retention/Collection

**What it is:** Storing more sensitive data than is actually needed for the business purpose, or keeping it far longer than necessary (e.g. full card numbers stored indefinitely instead of only the last 4 digits, or old KYC documents never purged after an account closes).

**Business Impact:** Every piece of sensitive data retained is additional exposure in the event of any breach — data that was never actually needed operationally, but still counts as customer harm if exposed. Data minimization (collecting and retaining only what's genuinely necessary) directly reduces the scope of any future incident.

## Sensitive Data in Logs

**What it is:** Application logs accidentally capturing sensitive values — full request bodies logged for debugging that happen to include passwords or card numbers, or a stack trace (file `15`) that includes sensitive variable values.

**Example:**
```python
# VULNERABLE — logs the entire request body for debugging, including sensitive fields
logger.info(f"Login attempt: {request.form}")  # includes the plaintext password if present
```
**Business Impact:** Logs are often more broadly accessible (to developers, support staff, third-party log aggregation services) than the primary database itself — sensitive data leaking into logs creates an additional, often overlooked exposure path that bypasses whatever access controls protect the main database.

## Sensitive Data Sent to the Client Unnecessarily

**What it is:** An API response including more fields than the frontend actually displays or needs — e.g. a `/user/profile` endpoint returning the user's full password hash, internal notes, or another customer's data embedded in a nested object, simply because the backend query fetched the whole record and the response was never trimmed down.

**Business Impact:** Even if the frontend UI never displays these extra fields, they're still fully visible to anyone inspecting the raw network response (trivial with browser dev tools) — an information disclosure risk that exists independent of anything the visible interface shows.

## Business Impact Summary

| Angle | What it actually means |
|---|---|
| **Financial loss** | Weakly-hashed passwords turn any future breach into full credential compromise at scale; excessive data retention increases the scope of harm from any single incident |
| **Regulatory / compliance** | PCI-DSS mandates specific handling for card data (never storing full card numbers/CVV where avoidable); most data protection regulations require data minimization — collecting/retaining only what's necessary — as an explicit principle, not just good practice |
| **Reputational damage** | "Weakly hashed passwords" and "excessive data retention" are both specifically called out in post-breach public analysis as aggravating factors — they don't cause the breach, but they determine how bad the breach turns out to be |
| **Legal liability** | Storing more data than necessary, or protecting it inadequately once collected, strengthens claims of inadequate data protection practice |
| **Operational cost** | A breach involving weakly-hashed passwords typically requires forcing a password reset for the entire user base, not just notifying affected users — significantly higher operational and customer-trust cost than a breach where passwords remained genuinely uncrackable |

**One-line interview answer:** *"Technically, sensitive data exposure covers how data is protected at rest and in transit beyond just the TLS layer — weak password hashing, excessive retention, or accidental logging of sensitive fields. For a bank, the real business impact is that these factors don't cause a breach themselves, but they determine how catastrophic a breach turns out to be — weakly-hashed passwords in particular can turn a contained incident into full credential compromise across the entire customer base."*

## Mitigation

1. **Use purpose-built password hashing** (bcrypt, argon2, scrypt) — never a general-purpose fast hash function, even with a salt added manually.
2. **Practice data minimization** — only collect and retain what's genuinely necessary for the business purpose; define and enforce explicit data retention/deletion policies rather than keeping everything indefinitely.
3. **Encrypt sensitive data at rest** (database-level encryption, or field-level encryption for especially sensitive fields like card numbers), in addition to transit encryption.
4. **Audit and sanitize logging** — explicitly exclude known-sensitive fields (passwords, tokens, full card numbers) from anything written to logs, and review logging code as part of standard code review.
5. **Trim API responses to only the fields the client actually needs** — never rely on the frontend simply not displaying extra fields as the security boundary.

## Explaining it to a developer
*"A few different things fall under this umbrella, but they share a theme: sensitive data being protected less than it should be, at some stage of its lifecycle. Passwords need a slow, purpose-built hash like bcrypt, not a fast general-purpose one. We shouldn't retain data longer than we actually need it. Logs shouldn't capture sensitive fields even for debugging convenience. And an API response should only include what the frontend genuinely needs — not the entire database record just because that's what the query happened to return."*

## Quick-reference table

| Issue | Fix |
|---|---|
| Weak/fast password hashing | Use bcrypt/argon2/scrypt instead |
| Excessive data retention | Data minimization, explicit retention/deletion policy |
| Sensitive data in logs | Exclude sensitive fields from logging explicitly |
| Over-inclusive API responses | Return only fields the client actually needs |
