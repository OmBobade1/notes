# Cookie Security Attributes

## Why this comes right after headers
`Set-Cookie` is itself a response header — so checking how cookies are configured is still part of the initial inspection, before you ever touch the login form or start injecting anything into input fields.

---

## HttpOnly

**What it is:** Tells the browser that JavaScript running on the page is not allowed to read this cookie at all — only the browser itself can send it back to the server automatically.

**Example:**
```
Set-Cookie: session_id=abc123; HttpOnly
```

**Why it matters:** If an attacker manages to inject a script into the page (via XSS), the very first thing they usually try is `document.cookie` to steal the session cookie and hijack the logged-in session. `HttpOnly` blocks that specific theft path entirely — the script simply can't see the cookie, even if the injection itself succeeded.

**Impact if missing:**
- Technical: any successful XSS on the page can now also steal the session cookie directly, turning a script-injection bug into a full account takeover
- Business: for a banking app, this converts a "someone can pop an alert box" bug into "someone can take over a customer's live banking session" — a direct fraud and regulatory-disclosure event, not a cosmetic issue

**Mitigation:** Always set `HttpOnly` on session and authentication cookies. There's essentially never a legitimate reason for client-side JavaScript to need to read a session cookie directly.

---

## Secure

**What it is:** Tells the browser to only ever send this cookie over an HTTPS connection — never over plain HTTP, even if the user somehow ends up on an HTTP version of the page.

**Example:**
```
Set-Cookie: session_id=abc123; Secure
```

**Why it matters:** Without this flag, if a user's connection ever gets downgraded to HTTP (accidentally, or via a MITM attack on public Wi-Fi), the session cookie gets sent in plain text — anyone on the same network can capture it.

**Impact if missing:**
- Technical: session cookie can be intercepted in transit on any unencrypted connection
- Business: same outcome as above — intercepted session cookie means account takeover — but this time the attack surface is any public/untrusted network the customer happens to be on, which is a very realistic scenario for mobile banking users

**Mitigation:** Always set `Secure` on every cookie, and pair it with the `Strict-Transport-Security` header (covered in the headers file) so HTTPS is enforced site-wide, not just for the cookie.

---

## SameSite

**What it is:** Controls whether this cookie gets sent along when a request originates from a *different* site than the one that set it — directly relevant to Cross-Site Request Forgery (CSRF).

**Values:**
- `Strict` — cookie is never sent on cross-site requests, even when following a legitimate link from another site (most secure, occasionally breaks normal cross-site navigation flows)
- `Lax` — cookie is sent on top-level navigation (e.g. clicking a link) but not on background cross-site requests (forms auto-submitted, images, etc.) — a reasonable balance, and the current browser default if unset
- `None` — cookie is sent on all cross-site requests regardless of origin (must be paired with `Secure`) — effectively disables this protection, only use if there's a genuine cross-site use case

**Why it matters:** Without a restrictive `SameSite` setting, if a logged-in user visits a malicious site, that site can silently trigger a request to your bank's server (e.g. a hidden auto-submitting form for a funds transfer), and the browser will still attach the user's real session cookie — the bank's server sees what looks like a legitimate, authenticated request.

**Impact if missing:**
- Technical: enables Cross-Site Request Forgery (CSRF) — actions get performed using the victim's real session, without their knowledge
- Business: for banking, unrestricted `SameSite` combined with a state-changing action (funds transfer, password change) with no other CSRF protection is a direct path to unauthorized transactions — this is the same "unauthorized transaction" business impact discussed for SQL injection, but reached through a completely different technical route

**Mitigation:** Set `SameSite=Strict` or `SameSite=Lax` on session cookies. Combine with CSRF tokens on state-changing requests (covered in its own file later) as defense-in-depth — `SameSite` alone is a strong browser-level protection, but shouldn't be the only one.

---

## Quick-reference table

| Attribute | Blocks | Missing = |
|---|---|---|
| HttpOnly | JavaScript from reading the cookie | XSS → session theft |
| Secure | Cookie sent over plain HTTP | MITM interception on untrusted networks |
| SameSite | Cookie sent on cross-site requests | CSRF — unauthorized actions using victim's session |

## Explaining it to a developer
*"Right now our session cookie doesn't have HttpOnly, Secure, or SameSite set. That means: if any page ever has an XSS bug, an attacker's script can read this cookie directly and hijack the session — HttpOnly stops that. If a user's connection is ever downgraded to HTTP, the cookie can be intercepted — Secure stops that. And a malicious site could trick a logged-in user's browser into sending real requests to us using this cookie — SameSite stops that. All three are one-line additions on the Set-Cookie header, and each one closes a completely different attack path."*
