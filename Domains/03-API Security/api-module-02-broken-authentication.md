# Module 02 - Broken Authentication (API-Specific)

## Why this comes right after BOLA
BOLA assumed authentication was already working correctly, and focused on what happens *after* — authorization to a specific object. This module steps back one level: the authentication mechanism itself, and the specific ways it breaks down differently in APIs compared to traditional session-based web apps.

---

## Why API authentication is a genuinely different problem than web session auth
Traditional web apps mostly rely on server-side sessions plus a cookie (covered fully in the web security series, files `02`-`03`). APIs — especially ones consumed by mobile apps, third-party integrations, and microservices talking to each other — overwhelmingly rely on **tokens** (JWTs, API keys, OAuth tokens) instead, because there's often no browser involved to hold a cookie at all. This shift in mechanism creates its own distinct set of failure modes.

---

## Weak JWT Validation in API Contexts

**Already covered in full technical depth in the web security series (file `17`)** — algorithm confusion, `alg: none`, weak signing secrets. Worth restating specifically why this matters *more* for APIs: a JWT is frequently the **only** thing standing between an anonymous request and full API access, with no secondary session-cookie layer behind it the way a traditional web app might have. A forged JWT against an API often means immediate, complete access — there's no second authentication layer to fall back on.

**A specific API-context testing step, beyond what's in file `17`:**
```
curl -H "Authorization: Bearer <forged-token>" https://api.target.com/users/me
```
Directly testing whether a forged/tampered token is accepted is often the very first authentication test run against any new API target, precisely because JWT misconfigurations are so common and immediately high-impact when found.

---

## API Keys Treated as Sufficient Authentication on Their Own

**What the problem actually is:** An API key is meant to identify *which application or client* is calling an API (for billing, rate limiting, or basic access control) — it was never designed to be a strong, sole proof of *user* identity. Treating a static API key as fully sufficient authentication, with no additional per-user verification behind it, is a common, serious mistake.

**Why this fails in practice:**
```javascript
// VULNERABLE — API key alone determines the response, no per-user check
app.get('/api/data', (req, res) => {
  const apiKey = req.headers['x-api-key'];
  if (!isValidApiKey(apiKey)) return res.status(401).send();
  res.json(getAllSensitiveData());  // same data returned for EVERY valid key holder
});
```
If the API key is meant to represent one specific client/customer but the endpoint doesn't actually check *which* customer that key belongs to before returning data, **any** valid key — even one meant for a completely different, unrelated customer — returns the same broad dataset.

**Why API keys leak so easily, connecting back to the web series:** Static API keys are exactly the kind of secret covered in the hardcoded secrets file (`27`) — frequently committed to public repositories, embedded directly in mobile app binaries (extractable via simple decompilation, covered in the mobile security content), or exposed in client-side JavaScript. A long-lived, static secret used as the *sole* authentication factor is a fragile design from the start, independent of any individual mistake in how it's stored.

**The correct model:** API keys should identify the *application*; a separate, per-user authentication mechanism (a user-specific token, OAuth flow) should determine *what that specific user* is allowed to see — two distinct layers, not one key doing both jobs.

---

## Credential Stuffing Against Login/Auth APIs Specifically

**Already covered conceptually in the network security series (Module 09) and the web security auth file (`03`)** — the API-specific angle worth adding: **login APIs are frequently less protected by rate limiting than the equivalent web login page**, because developers often focus hardening effort on the user-facing login form while forgetting that the underlying API endpoint it calls can often be hit directly, bypassing whatever frontend-level protections (CAPTCHA, JavaScript-based throttling) were added to the visible form.

```
hydra -L users.txt -P passwords.txt api.target.com https-post-form "/api/v1/login:{\"username\":\"^USER^\",\"password\":\"^PASS^\"}:Invalid credentials" -H "Content-Type: application/json"
```
This directly targets the underlying JSON API endpoint the login form actually calls, entirely bypassing any client-side-only protections that only apply to interactions through the visible web form itself.

**Why this specific gap is so common:** a CAPTCHA implemented purely in JavaScript on the frontend does nothing to stop a script calling the API endpoint directly — the protection has to exist at the API layer itself (server-side rate limiting, server-side CAPTCHA verification) to actually matter.

---

## Missing or Weak Token Expiration/Revocation

**The API-specific angle beyond what's in file `17`:** many APIs, particularly older or simpler ones, issue long-lived tokens with no practical revocation mechanism at all — if a token is compromised, there's often no way to invalidate just that one token without invalidating every token system-wide (or without waiting for a very long expiration period to naturally pass).

**Why this matters practically:** a stolen API token from a compromised mobile device or a leaked mobile app binary could remain valid and usable for months if the API was never designed with short expiration and proper refresh-token rotation in mind.

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | A forged JWT or a mishandled API key with no per-user check can mean full, immediate account access — for a banking API, this is direct unauthorized access to financial data or actions, no secondary barrier to fall back on |
| **Regulatory / compliance** | API authentication is explicitly scrutinized in banking security assessments given how central API access has become to mobile banking specifically — a weak token/key scheme is a foundational-level finding |
| **Reputational damage** | Because a single compromised API credential (key or token) can potentially grant broad access with no secondary check, the scope of a real incident originating here tends to be larger than a single-account web login compromise |
| **Legal liability** | Using a client-identification mechanism (an API key) as if it were user authentication is a well-documented anti-pattern — difficult to defend as reasonable design after the fact |
| **Operational cost** | Without proper token revocation, incident response after a suspected compromise is significantly harder — you may not be able to cleanly invalidate just the affected credential |

**One-line interview answer:** *"API authentication breaks differently than web session auth because APIs typically rely entirely on tokens or keys with no secondary session layer behind them. The two most common real mistakes are treating a static API key — which only identifies the calling application — as if it were sufficient proof of user identity, and protecting the visible login form's UI while leaving the actual underlying API endpoint unprotected against direct, scripted credential stuffing."*

## Mitigation

1. **Separate application identity (API key) from user identity (a proper per-user auth token)** — never let a static API key alone determine what user-specific data is returned.
2. **Apply the full JWT hardening from file `17`** — pinned algorithm, strong secret, short expiration — specifically for API-issued tokens, not just traditional web session tokens.
3. **Rate-limit and monitor the actual API endpoint directly**, not just the visible frontend form — server-side protection that a direct API call cannot bypass.
4. **Implement genuine token revocation** — short-lived access tokens paired with a separately revocable refresh token, so a compromised credential can be cut off without a system-wide reset.
5. **Never treat client-side (JavaScript-only) protections as sufficient** — CAPTCHA, throttling, or validation implemented purely in frontend code protects nothing against a script calling the API directly.

## Quick-reference table

| Issue | Root Cause | Fix |
|---|---|---|
| Weak JWT validation | Same as web file `17`, higher stakes with no session fallback | Pinned algorithm, strong secret, short expiry |
| API key as sole auth | Key identifies the app, not the user | Separate app identity from user identity |
| Credential stuffing on API | Frontend-only protections don't cover direct API calls | Server-side rate limiting on the actual endpoint |
| No token revocation | Long-lived tokens, no invalidation mechanism | Short-lived tokens + revocable refresh tokens |
