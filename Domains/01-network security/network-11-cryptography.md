# 11 - Cryptography Fundamentals (Explained Simply)

## Why this closes out the network sequence
Almost every defense mentioned across files `00` through `10` — HTTPS, WPA2/WPA3, VPNs, password hashing — ultimately relies on cryptography underneath it. This file is the "how does the lock actually work" behind all of those mentions, tying the whole series together.

---

## The core idea, in one sentence
Cryptography is **turning readable information into scrambled nonsense that only someone with the right key can turn back into something readable** — a secret code, but mathematically rigorous instead of a childhood cipher.

## Encryption vs. Hashing — the mix-up that trips people up most
These get confused constantly, but they do fundamentally different jobs:

**Encryption** — scrambles data in a way that's meant to be **reversible** — if you have the right key, you can turn the scrambled data back into the original. Used when you need to get the original information back later (sending a private message, storing a file securely).

**Hashing** — scrambles data in a way that's meant to be **one-way, never reversible**. You can't "un-hash" something to get the original back. Used when you only need to *verify* something matches, not recover it — this is exactly why the sensitive data exposure file in the web security series insisted passwords be hashed (with bcrypt), not encrypted: the system only ever needs to check "does this match what I stored," never "what was the original password."

## Symmetric vs Asymmetric Encryption — the two families

**Symmetric Encryption** — the **same key** both locks and unlocks the data. Fast and efficient, but has one obvious logistical problem: both sides need to already have the same secret key, which means that key has to be shared somehow first — and if that sharing step is intercepted, the whole system is broken.

```
Same key encrypts AND decrypts:  Plaintext --[Key]--> Ciphertext --[Same Key]--> Plaintext
```

**Asymmetric Encryption (Public-Key Cryptography)** — uses **two mathematically related keys**: a **public key** (safe to share with literally anyone) and a **private key** (kept secret, never shared). Anything encrypted with the public key can only be decrypted with the matching private key — solving symmetric encryption's key-sharing problem, since the public key never needs to be kept secret in the first place.

```
Public key encrypts, only the matching private key can decrypt:
Plaintext --[Public Key]--> Ciphertext --[Private Key]--> Plaintext
```

**Why both are actually used together in practice (this is what HTTPS really does):** asymmetric encryption is much slower than symmetric encryption, so real systems use a hybrid approach — asymmetric encryption is used briefly at the start of a connection just to safely exchange a temporary symmetric key, then the fast symmetric encryption handles the actual bulk of the conversation from that point on. This is exactly what happens during an HTTPS connection's handshake.

## Where hashing shows up, beyond just passwords
**Integrity checking** — hashing a file lets you verify later that it hasn't been altered even slightly. If you hash a file today and hash it again next month, an identical hash proves nothing changed — even a single-character change produces a completely different hash. This is conceptually related to the log file validation concept from the cloud security lab report — cryptographic hashing is what makes tamper-evidence possible in the first place.

**Digital Signatures** — combine hashing and asymmetric encryption to prove both that a message came from a specific sender *and* that it wasn't altered in transit — the sender hashes the message, then encrypts that hash with their own private key; anyone with the sender's public key can verify the signature matches, proving authenticity.

## Certificates and PKI — how "who do I actually trust" gets solved
Asymmetric encryption solves *how* to encrypt safely, but not *who to trust* in the first place — how does your browser know a website's public key genuinely belongs to that website, and not to an attacker pretending to be them? This is what **digital certificates** and **PKI (Public Key Infrastructure)** solve: a trusted third party (a **Certificate Authority**) verifies a website's identity and issues a certificate vouching for it, which your browser checks automatically — this is the padlock icon and "connection is secure" message in a browser, and it's exactly what a weak/self-signed certificate (covered in the weak TLS configuration file in the web security series) undermines.

## Why "weak encryption" and "no encryption" are treated as equally serious in real assessments
An outdated or broken encryption algorithm (like the old, deprecated ciphers mentioned in the weak TLS file) can create a false sense of security — everything *looks* encrypted (the padlock is there), but the underlying math is weak enough that a sufficiently motivated attacker can break it anyway. This is why cryptography assessments care not just about "is it encrypted" but specifically "encrypted with what, and is that specific method still considered strong today" — the field moves, and yesterday's strong standard can become tomorrow's known-broken one.

## Quick-reference table

| Term | Plain meaning |
|---|---|
| Encryption | Reversible scrambling — can get the original back with the right key |
| Hashing | One-way scrambling — used to verify, never to recover the original |
| Symmetric Encryption | Same key locks and unlocks — fast, but requires safely sharing that key first |
| Asymmetric Encryption | Public key locks, private key unlocks — solves the key-sharing problem |
| Digital Signature | Proves a message's sender and that it wasn't altered |
| Certificate Authority | A trusted third party that verifies "this public key really belongs to this website" |

---

## This closes the 12-file network fundamentals sequence (00-11)
Together, these cover: the basics of how networks talk (`00`), finding what's out there (`01`), learning what it's willing to tell you (`02`), listening in on traffic (`03`), figuring out what's actually weak (`04`), what happens once someone's in (`05`), the malicious software often used to get and stay there (`06`), the non-technical human-layer path in (`07`), taking a service down entirely (`08`), the defensive tools that catch and block all of the above (`09`), the wireless-specific version of many of these same ideas (`10`), and finally the cryptographic foundation that so many of the defenses mentioned throughout actually depend on (`11`).
