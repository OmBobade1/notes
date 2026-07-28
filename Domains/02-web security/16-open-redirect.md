# Open Redirect

## Why this comes next
Security Misconfiguration was about what a server exposes passively. Open Redirect is the reverse — a legitimate, intentional feature (redirecting a user somewhere after login, or via a shortened link) that gets abused because the destination isn't restricted. It's often underrated because it looks harmless on its own — but it becomes a serious phishing enabler precisely because it's hosted on a domain the victim already trusts.

---

## What it is (in plain terms)
Many applications redirect users based on a URL parameter — most commonly, "log in, then go back to where you were" (`?redirect=/dashboard`). Open Redirect happens when that parameter isn't restricted to internal, expected destinations, letting an attacker supply an external URL instead — so the trusted domain redirects the victim straight to an attacker-controlled site.

## Why it exists — the real-life cause

```python
# VULNERABLE — redirects to whatever URL is provided, no restriction
@app.route('/login')
def login():
    # ... authentication logic ...
    redirect_url = request.args.get('redirect', '/dashboard')
    return redirect(redirect_url)  # trusts the parameter completely
```
A legitimate link looks like `https://bank.com/login?redirect=/dashboard` — harmless, keeps the user on the bank's own site. But nothing stops an attacker from crafting `https://bank.com/login?redirect=https://bank-secure-login.attacker.com` instead. The link itself starts with the real, trusted `bank.com` domain (which is what the victim actually sees and trusts when they hover over or glance at the link), but after login, it silently sends them to the attacker's look-alike phishing page.

```python
# SECURE — only allows relative, internal paths as redirect destinations
from urllib.parse import urlparse

@app.route('/login')
def login():
    # ... authentication logic ...
    redirect_url = request.args.get('redirect', '/dashboard')
    parsed = urlparse(redirect_url)
    if parsed.netloc or parsed.scheme:  # rejects anything with a domain/scheme — only bare paths allowed
        redirect_url = '/dashboard'
    return redirect(redirect_url)
```
Here, the redirect target is checked to ensure it's a relative path only (no domain, no `http://`/`https://` prefix) — an attacker-supplied external URL gets rejected and replaced with a safe default.

## How an attacker actually does it, step by step
1. Find any redirect parameter in the application — `?redirect=`, `?returnUrl=`, `?next=`, `?continue=`.
2. Test whether an external URL is accepted: `?redirect=https://evil.com` — if the application actually sends the browser to `evil.com`, Open Redirect is confirmed.
3. Build a phishing campaign using the real, trusted domain as the visible link (`https://bank.com/login?redirect=https://attacker-phishing-page.com`) — since the link starts with the bank's own genuine domain, it passes casual inspection, bypasses some email security filters that check the visible domain, and feels far more trustworthy to the victim than an obviously unrelated phishing URL would.
4. The victim clicks, briefly sees the real bank's actual login page (since the redirect happens *after* authentication), then gets silently forwarded to a convincing fake page designed to capture credentials or push a fake "security update" download.

## Technical Impact
- Doesn't compromise the application itself directly — the vulnerability is entirely about *abusing user trust in the domain*
- Enables highly convincing phishing, since the initial link is genuinely hosted on the real, trusted domain

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | Phishing campaigns built on a real bank's own Open Redirect are measurably more effective than generic phishing, since the initial link passes basic trust checks — leading to more successful credential theft and subsequent account takeover/fraud |
| **Regulatory / compliance** | While the technical severity is lower than RCE-class findings, banks are often specifically evaluated on anti-phishing controls given how central phishing is to real-world financial fraud — an Open Redirect actively undermines those controls rather than being a neutral, separate issue |
| **Reputational damage** | Victims who fall for a phishing attack launched via the bank's own domain often (understandably) blame the bank directly, even though the credential theft happened on a different final page — "the bank's own website sent me there" is a difficult narrative to walk back publicly |
| **Legal liability** | Weaker than RCE-class findings individually, but contributes to liability if it can be shown the bank's own infrastructure was knowingly used to add legitimacy to a phishing campaign that led to customer losses |
| **Operational cost** | Typically lower remediation cost than other findings in this series — usually a straightforward code fix — but customer-facing incident response (warning customers about active phishing campaigns using the bank's domain) can still be significant |

**One-line interview answer:** *"Technically, Open Redirect lets an attacker craft a link that starts with our real domain but ends up sending the victim to an attacker-controlled site after login. The business impact is specifically about phishing effectiveness — because the initial link is genuinely on our real domain, it passes casual trust checks far better than a normal phishing link would, making successful credential theft and resulting fraud measurably more likely."*

## Mitigation — layered, not just one fix

1. **Restrict redirect targets to relative, internal paths only (the real fix)** — as shown above; never allow a full external URL as a valid redirect destination.
2. **If external redirects are genuinely required** (e.g. a legitimate partner integration), use an explicit allow-list of specific approved external domains — never accept an arbitrary URL.
3. **Show an interstitial warning page for any external redirect** ("You are leaving [bank]'s website") as a defense-in-depth measure, even for allow-listed destinations, so the user has a clear signal they're leaving the trusted domain.
4. **Avoid putting the actual redirect logic in a heavily-trusted, high-traffic domain path** where possible — isolating redirect functionality reduces how convincing an abused link can look.

## Explaining it to a developer
*"This login redirect trusts whatever URL is passed in the `redirect` parameter completely — including a full external URL, not just a path on our own site. That means someone can craft a link that starts with our real domain, which people trust and click without hesitation, but actually sends them to a phishing site right after they log in. The fix is to only accept relative paths as valid redirect targets — reject anything that includes a domain or `http://` prefix, and fall back to a safe default page instead."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Restrict to relative/internal paths only | External redirect targets rejected entirely |
| Explicit allow-list for legitimate external redirects | Only pre-approved destinations ever reachable |
| Interstitial "leaving this site" warning | Gives the user a clear signal even for allowed redirects |
