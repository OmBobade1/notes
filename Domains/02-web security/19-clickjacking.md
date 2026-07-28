# Clickjacking

## Why this gets its own file now
X-Frame-Options was mentioned briefly in file `01` as one header among several. This is the actual attack that header exists to prevent — worth its own deep-dive since it's a distinct technique, not just a missing-header checkbox.

---

## What it is (in plain terms)
Clickjacking tricks a logged-in user into clicking something they can't actually see. An attacker loads your real site inside an invisible `<iframe>` on their own malicious page, positions it precisely under something that looks harmless (a "Claim your prize" button), and when the victim clicks what they think is the attacker's button, they're actually clicking a real button on your real site underneath — like "Confirm Transfer" or "Enable 2FA-bypass."

## Why it exists — the real-life cause

```html
<!-- Attacker's malicious page -->
<style>
  iframe {
    opacity: 0.0001;   /* invisible to the victim */
    position: absolute;
    top: 300px; left: 200px;
    width: 200px; height: 50px;
    z-index: 2;
  }
  .fake-button {
    position: absolute;
    top: 300px; left: 200px;
    z-index: 1;
  }
</style>
<button class="fake-button">Click here to win a prize!</button>
<iframe src="https://bank.com/transfer?amount=5000&to=attacker_account"></iframe>
```
The invisible iframe is positioned exactly over the fake button. The victim sees only "Click here to win a prize!" and clicks — but the actual click lands on the real bank.com transfer confirmation button underneath, which the browser dutifully processes because the victim genuinely is logged into bank.com in that hidden frame.

```
# SECURE — response header on the real site
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'
```
With either header set, the browser simply refuses to load bank.com inside any iframe at all — the attacker's page gets a blank space instead, and the attack doesn't work no matter how it's disguised.

## How an attacker actually does it, step by step
1. Confirm the target page can be framed — load it inside a test `<iframe>` on a separate page; if it renders instead of refusing, it's frameable.
2. Identify a valuable one-click action on the target (confirm transfer, enable a setting, delete something, like/follow for social engineering purposes).
3. Build a decoy page with an enticing visible element, and precisely position an invisible iframe of the real target directly on top, aligned so the real button sits exactly where the decoy button appears.
4. Distribute the malicious page link; any logged-in victim who clicks the visible decoy unknowingly triggers the real action underneath.

## Technical Impact
- Tricking a user into performing a real action on a trusted site without their knowledge or consent — the specific impact depends entirely on what action was framed (funds transfer, settings change, permission grant)

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | If a funds-transfer confirmation or similar financial action can be framed, this becomes a direct path to unauthorized transactions, similar in outcome to CSRF but achieved through visual deception rather than a forged request |
| **Regulatory / compliance** | Missing frame protection on any page containing a sensitive action is a standard, expected finding in banking security assessments — straightforward to test for and straightforward to fix, so its presence reflects poorly in an audit |
| **Reputational damage** | Victims of a successful clickjacking attack often can't explain what happened at all — they clicked something that looked completely unrelated — making it a confusing, hard-to-diagnose customer complaint |
| **Legal liability** | A missing, one-line header protecting against a well-known attack technique is a hard omission to justify after the fact |
| **Operational cost** | Low remediation cost (a header), but investigation cost for any actual exploited incident is similar to other unauthorized-transaction scenarios |

**One-line interview answer:** *"Technically, clickjacking tricks a user into clicking a real button on our site that's hidden inside an invisible frame under a decoy. For a bank, if any sensitive one-click action — like a transfer confirmation — can be framed, this becomes a path to unauthorized actions the victim never knowingly agreed to, which is both a fraud risk and a straightforward audit finding since the fix is just one missing header."*

## Mitigation

1. **`X-Frame-Options: DENY`** (or `SAMEORIGIN` if legitimate same-site framing is needed) — the direct, simple fix.
2. **`Content-Security-Policy: frame-ancestors 'none'`** — the modern equivalent, more granular, and takes precedence over X-Frame-Options in browsers that support both; safe to set both together for broader compatibility.
3. **JavaScript frame-busting as a legacy backup only** (e.g. checking `if (top !== self) top.location = self.location`) — historically used before the headers above existed, but easily bypassed and should never be the *only* protection in place.

## Explaining it to a developer
*"Any page with a sensitive one-click action needs to explicitly tell the browser it can't be loaded inside another site's iframe — otherwise an attacker can overlay our real button underneath their own decoy content and trick a logged-in user into clicking it without knowing. It's a one-line fix: set `X-Frame-Options: DENY` or the CSP `frame-ancestors 'none'` directive on the response."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| X-Frame-Options: DENY | Page cannot be framed by any other site at all |
| CSP frame-ancestors 'none' | Modern, more granular equivalent |
| JS frame-busting | Legacy fallback only — not a primary defense |
