# CORS (Cross-Origin Resource Sharing) Misconfiguration

## Why this comes next
Browsers enforce the Same-Origin Policy by default — JavaScript on one domain can't read responses from a different domain. CORS is the mechanism a server uses to deliberately *relax* that restriction for specific, trusted origins. Misconfiguration happens when that relaxation is far too broad — effectively telling the browser "any website can read this response," including malicious ones.

---

## What it is (in plain terms)
When a browser makes a cross-origin request (JavaScript on `attacker.com` requesting data from `bank.com`), the browser only lets that JavaScript read the response if `bank.com`'s server explicitly says it's allowed, via the `Access-Control-Allow-Origin` header. CORS misconfiguration happens when that header is set too permissively — especially combined with allowing credentials (cookies) to be sent along with the cross-origin request.

## Why it exists — the real-life cause

```python
# VULNERABLE — reflects whatever Origin the request claims, and allows credentials
@app.after_request
def add_cors_headers(response):
    response.headers['Access-Control-Allow-Origin'] = request.headers.get('Origin')  # reflects ANY origin
    response.headers['Access-Control-Allow-Credentials'] = 'true'  # allows cookies to be sent too
    return response
```
This pattern — dynamically reflecting back whatever `Origin` header the request sent, rather than checking it against an allow-list — combined with `Access-Control-Allow-Credentials: true`, means literally any website can make a request to this API *using the victim's own logged-in session cookie*, and actually read the response back in JavaScript. This is functionally similar to CSRF (file `06`), but worse: CSRF lets an attacker trigger an action blindly, while this misconfiguration lets the attacker's JavaScript directly read the response data (account details, transaction history) back.

```python
# SECURE — explicit allow-list, only trusted origins get credentialed access
ALLOWED_ORIGINS = ['https://app.bank.com', 'https://mobile.bank.com']

@app.after_request
def add_cors_headers(response):
    origin = request.headers.get('Origin')
    if origin in ALLOWED_ORIGINS:
        response.headers['Access-Control-Allow-Origin'] = origin
        response.headers['Access-Control-Allow-Credentials'] = 'true'
    # if origin isn't in the allow-list, no CORS headers are set at all — browser blocks the read
    return response
```

## How an attacker actually does it, step by step
1. Send a request to the target API with an arbitrary `Origin` header (e.g. `Origin: https://attacker.com`) and check whether the response reflects that exact value back in `Access-Control-Allow-Origin`.
2. Check whether `Access-Control-Allow-Credentials: true` is also present — this is the critical combination that makes the misconfiguration exploitable using the victim's real session.
3. Host a malicious page on `attacker.com` that makes a `fetch()` request to the vulnerable API `with credentials: 'include'` — if the victim is logged into the target site and visits the attacker's page, their browser automatically attaches the real session cookie, and because of the misconfiguration, the attacker's JavaScript can read the actual response data back.

## Technical Impact
- Cross-origin theft of any data the API returns for a logged-in user — account details, transaction history, personal information — read directly by attacker-controlled JavaScript using the victim's real session

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | Unlike CSRF (which can trigger actions blindly), a CORS misconfiguration lets the attacker's script directly *read* the response — meaning full account/transaction data theft is possible just by getting a logged-in victim to visit a malicious page, no phishing of credentials required at all |
| **Regulatory / compliance** | Direct, wholesale exposure of customer account data via a browser-based attack is treated as a significant confidentiality breach in any assessment |
| **Reputational damage** | Same tier as other silent data-theft vulnerabilities — customers have no way to know this occurred, since nothing about their own session appeared unusual to them |
| **Legal liability** | A CORS configuration that reflects any origin combined with allowing credentials is a well-documented anti-pattern with clear, known-correct alternatives — difficult to justify after the fact |
| **Operational cost** | Requires determining exactly what data was exposed and to which origins, similar investigative depth to other data-exposure incidents |

**One-line interview answer:** *"Technically, CORS misconfiguration happens when a server reflects back any requesting origin instead of checking it against a trusted list, especially when combined with allowing credentials — that combination lets any malicious website read a logged-in user's data directly through their own browser. For a bank, this is more severe than CSRF in one way: it's not just triggering an action blindly, it's actually reading the private response data back, like account details or transaction history."*

## Mitigation

1. **Use an explicit allow-list of trusted origins (the real fix)** — never dynamically reflect the `Origin` header value back without checking it first.
2. **Never combine a wildcard/reflected origin with `Access-Control-Allow-Credentials: true`** — browsers actually block this exact combination when using a literal wildcard `*`, which is why misconfigured servers resort to dynamically reflecting the origin instead — a clear anti-pattern warning sign to watch for specifically.
3. **Only enable CORS on endpoints that genuinely need cross-origin access** — many APIs don't need CORS enabled at all if they're only ever called from the same origin.

## Explaining it to a developer
*"Right now, this endpoint reflects back whatever Origin header a request sends, and also allows credentials — which together mean any website can make a request using a logged-in user's real session cookie and actually read the response back in their own JavaScript. The fix is to check the Origin against a specific, known list of our own trusted domains, and only send the CORS headers back if it matches — never dynamically reflect an arbitrary value."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Explicit origin allow-list | Prevents arbitrary origins from being trusted |
| Never combine reflected/wildcard origin with credentials | Removes the specific combination that enables data theft |
| Only enable CORS where genuinely needed | Reduces overall attack surface |
