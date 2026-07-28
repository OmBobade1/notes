# Authentication & Session Management

## Why this comes third
Headers and cookies are checked passively, just by looking at responses. The login flow is the first thing you actually *interact* with — so it's the next natural stop before testing individual input fields for injection.

---

## Session Fixation

**What it is:** A vulnerability where the application does not issue a **new** session ID after a user logs in — it just keeps using whatever session ID existed before login. An attacker can set a known session ID on a victim's browser, wait for them to log in, and then use that same known ID to access the now-authenticated session themselves.

**How an attacker does it:**
1. Attacker visits the login page, gets a session ID (e.g. `session_id=attacker123`) before logging in.
2. Attacker sends the victim a link that plants this same session ID in the victim's browser (e.g. via a URL parameter, if the app accepts session IDs that way).
3. Victim logs in normally — but the app keeps the same `session_id=attacker123` instead of issuing a fresh one.
4. Attacker uses `session_id=attacker123` themselves and is now logged in as the victim, with no password needed.

**Technical Impact:** Full account takeover using a session ID the attacker already controlled before the victim ever logged in.

**Business Impact:** For a banking application, this is a direct account-takeover path that bypasses authentication entirely — a customer's account is compromised without their password ever being stolen, making it harder to trace and harder to explain in a post-incident report. Regulators specifically look for session-regeneration-on-login as a baseline control; its absence is a straightforward compliance finding.

**Mitigation:** Always issue a brand-new session ID immediately after successful login (`session.regenerate()` in most frameworks), and invalidate the old pre-login session ID completely.

---

## Session Timeout & Expiration

**What it is:** How long a session stays valid without activity, and whether it ever expires at all.

**Why it matters:** A session with no timeout (or an excessively long one, e.g. 30 days) stays valid on a shared or stolen device indefinitely. If a customer logs into their banking app on a public/shared computer and forgets to log out, an unlimited session means the next person on that device is still logged in as them.

**Technical Impact:** Indefinite window for session hijacking or misuse on shared/stolen/compromised devices.

**Business Impact:** For banking specifically, idle session timeout (commonly 5-15 minutes for financial apps) is often an explicit regulatory expectation, not just good practice — missing it is flagged directly in security audits, and a resulting unauthorized transaction from a "forgotten to log out" scenario is a real, recorded category of financial fraud loss.

**Mitigation:** Implement both an idle timeout (session expires after N minutes of inactivity) and an absolute timeout (session expires after N hours regardless of activity, forcing re-login periodically).

---

## Weak / Predictable Session Tokens

**What it is:** Session IDs that are short, sequential, or generated in a predictable way (e.g. incrementing numbers, or based on timestamp/username) instead of long, random values.

**Example of the flaw:**
```
# VULNERABLE — predictable, sequential session IDs
session_id = current_user_count + 1
```
```
# SECURE — cryptographically random, long token
session_id = secrets.token_urlsafe(32)
```

**Why it matters:** If session IDs are predictable, an attacker doesn't need to steal anyone's cookie — they can simply guess or enumerate valid session IDs directly (e.g. trying `session_id=1001`, `1002`, `1003`...) and land on an active, logged-in session.

**Technical Impact:** Session hijacking without ever needing XSS, MITM, or any other attack — just guessing.

**Business Impact:** This is one of the most severe possible findings for a bank, since it means *no other vulnerability is even required* to take over accounts at scale — an attacker could potentially script through thousands of sessions. This typically escalates directly to a critical-severity finding and immediate remediation demand.

**Mitigation:** Use the framework's built-in, cryptographically secure session ID generator (never write custom session ID logic) — tokens should be long (128+ bits of entropy) and generated from a secure random source.

---

## Password Reset Flow Weaknesses

**What it is:** Flaws in the "forgot password" flow — most commonly, a reset token that's predictable, doesn't expire, or isn't tied to the correct account.

**Common flaws:**
- Reset link never expires, or stays valid for days
- Reset token is guessable (e.g. sequential, or based on timestamp)
- Reset token isn't invalidated after first use — can be reused later
- Reset confirmation reveals whether an email/username exists in the system at all (username enumeration)

**Technical Impact:** Account takeover via a manipulated or reused password-reset token, or systematic discovery of which usernames/emails are valid, feeding into further targeted attacks.

**Business Impact:** For banking, a leaked or predictable reset token is functionally equivalent to a stolen password — it's a direct account-takeover path that specifically targets the *recovery* mechanism, which is often less scrutinized than the login page itself during development.

**Mitigation:** Reset tokens must be long, random, single-use, and expire quickly (typically 15-30 minutes). Never reveal whether a given email/username exists in the system — always show the same generic message ("if that account exists, a reset link has been sent") regardless of whether it does.

---

## Lack of Rate Limiting / Account Lockout

**What it is:** No limit on how many login attempts (or password-reset attempts) a single account or IP address can make.

**Why it matters:** Without rate limiting, an attacker can brute-force passwords directly, or run **credential stuffing** — testing a list of leaked username/password pairs from other breaches against this login page — completely automated, at high speed.

**Technical Impact:** Automated password guessing at scale, with no friction.

**Business Impact:** Credential stuffing specifically is one of the most common real-world attack vectors against banking logins today, precisely because so many users reuse passwords across sites — a bank with no rate limiting is directly exposed to this even if every individual password is otherwise "strong," since the passwords are valid, just stolen from elsewhere.

**Mitigation:** Implement rate limiting per account and per IP, account lockout (or exponential backoff) after repeated failures, and CAPTCHA on the login form after a few failed attempts. Multi-Factor Authentication (MFA) is the strongest mitigation here — even a correctly guessed/stuffed password is useless without the second factor.

---

## Quick-reference table

| Issue | Root Cause | Missing = |
|---|---|---|
| Session Fixation | Session ID not regenerated after login | Account takeover via pre-set session ID |
| Session Timeout | No idle/absolute expiration | Indefinite access on shared/stolen devices |
| Weak Session Tokens | Predictable/sequential ID generation | Session hijacking by guessing |
| Password Reset Flaws | Predictable/reusable/non-expiring reset tokens | Account takeover via recovery flow |
| No Rate Limiting | No cap on login attempts | Brute-force / credential stuffing at scale |

## Explaining it to a developer
*"Each of these is a different door into the same room — a customer's account. Regenerating the session ID at login closes the fixation door. A short idle timeout closes the shared-device door. Random, long session tokens close the guessing door. A single-use, expiring reset token closes the password-recovery door. And rate limiting closes the brute-force door. None of these require complex fixes — they're mostly framework defaults or a few lines of configuration — but skipping any one of them leaves that specific door open regardless of how strong the others are."*
