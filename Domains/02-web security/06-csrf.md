# Cross-Site Request Forgery (CSRF)

## Why this comes right after XSS
XSS and CSRF are often confused, but they're opposites in one key way: XSS is about an attacker running *their* code inside *your* trusted page. CSRF is about an attacker tricking the *victim's browser* into sending a real, authenticated request to your site — without ever needing to inject anything into your page at all. This one connects directly back to the `SameSite` cookie attribute from file `02`.

---

## What it is (in plain terms)
CSRF happens when a website blindly trusts that any request arriving with a valid session cookie must have been intentionally made by that logged-in user. It wasn't checking anything else — so if an attacker can get the victim's browser to send a request (even from a completely different website), and the victim happens to be logged in, the browser automatically attaches their real session cookie, and the server processes it as a legitimate action.

## Why it exists — the real-life cause

```html
<!-- VULNERABLE — no CSRF token, relies only on the session cookie -->
<form action="https://bank.com/transfer" method="POST">
  <input type="hidden" name="amount" value="5000">
  <input type="hidden" name="to_account" value="attacker_account">
</form>
<script>document.forms[0].submit()</script>
```
If this HTML sits on a malicious site, and a victim who's currently logged into `bank.com` visits that malicious page, their browser auto-submits this form to the real bank.com — attaching their real session cookie automatically, because that's just how cookies work by default. The bank's server has no way to know this request came from an attacker's page instead of the bank's own transfer form, because the session cookie alone looked completely valid.

```html
<!-- SECURE — includes a CSRF token unique to this user's session -->
<form action="https://bank.com/transfer" method="POST">
  <input type="hidden" name="amount" value="5000">
  <input type="hidden" name="to_account" value="recipient_account">
  <input type="hidden" name="csrf_token" value="a1b2c3d4-unique-per-session">
</form>
```
```python
# SECURE — server-side check
if request.form['csrf_token'] != session['csrf_token']:
    abort(403)  # reject the request
```
The attacker's malicious page has no way to know or guess this token, since it's unique per session and never exposed anywhere the attacker can read it. Without the correct token, the server rejects the request — even though the session cookie itself was valid.

## How an attacker actually does it, step by step
1. Find a state-changing action with no CSRF protection — a funds transfer, password change, or email-change form that relies only on the session cookie.
2. Build a page (or hidden auto-submitting form/image tag) that sends that exact same request.
3. Get the victim to visit that page while they're logged into the target site — via a phishing link, malicious ad, or compromised third-party site.
4. The victim's browser automatically attaches their real session cookie to the forged request — the action executes as if the victim did it themselves.

## Technical Impact
- Any state-changing action reachable via the vulnerable endpoint — funds transfer, password change, email change, account deletion — performed without the victim's knowledge or consent
- Particularly dangerous when combined with a password/email change: the attacker changes the victim's recovery email first, then resets the password, achieving full account takeover in two forged requests

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | If a funds-transfer endpoint lacks CSRF protection, this is a direct, automatable path to unauthorized transactions — the exact scenario a banking interviewer is testing for when they ask "how does this harm us" |
| **Regulatory / compliance** | CSRF protection on state-changing financial actions is a standard, explicitly expected control in banking security assessments and compliance frameworks — its absence is a clear-cut audit finding |
| **Reputational damage** | Victims of a CSRF-driven unauthorized transfer often can't easily explain what happened (they never clicked anything suspicious on the bank's own site) — this generates confused, high-friction customer complaints and support escalations even before root cause is identified |
| **Legal liability** | Unauthorized transactions caused by a bank's own missing server-side validation strengthen a customer's case that the bank, not the customer, was negligent |
| **Operational cost** | Every affected transaction needs individual investigation, reversal (where possible), and customer compensation — plus the engineering cost of retrofitting CSRF tokens across every state-changing endpoint after the fact, which is more expensive than building it in from the start |

**One-line interview answer:** *"Technically, CSRF lets an attacker forge a request that the victim's browser sends with their real session cookie attached — but the business impact is that any unprotected state-changing action, like a funds transfer, could be triggered without the customer's knowledge, leading to direct financial loss and a difficult-to-explain fraud claim, since the customer genuinely never clicked anything on our site."*

## Mitigation — layered, not just one fix

1. **CSRF tokens (the real fix)** — a unique, unpredictable token tied to the user's session, included in every state-changing form and verified server-side on submission, as shown above. This is the primary, direct defense.
2. **SameSite cookie attribute** (see file `02`) — set to `Strict` or `Lax`, this stops the browser from attaching the session cookie to cross-site requests in the first place, closing the attack at the browser level as a second, independent layer.
3. **Re-authentication for sensitive actions** — requiring the current password (or a fresh MFA prompt) before high-risk actions like funds transfers or email changes, so even a successful CSRF can't complete the action without that extra step.
4. **Checking the `Origin`/`Referer` header server-side** — a supplementary check confirming the request actually originated from the bank's own domain, not a foolproof primary defense on its own, but useful as an additional signal.

## Explaining it to a developer
*"Right now, this transfer endpoint only checks whether a valid session cookie was sent — it doesn't check whether the request was actually intended by the user or forged by another site. Since browsers attach cookies automatically to any request to a domain, a malicious site can trigger this same request using the victim's real session without them knowing. The fix is a CSRF token: a random value tied to their session that only our own form knows, so a forged request from another site can't include it and gets rejected."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| CSRF token | Forged requests missing the correct per-session token |
| SameSite cookie | Cookie never even sent on cross-site requests |
| Re-authentication | Even a successful forgery can't complete high-risk actions |
| Origin/Referer check | Supplementary signal the request came from an unexpected source |
