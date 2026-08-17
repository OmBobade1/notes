# Module 07 - Security Misconfiguration (API-Specific)

## Why this needs separate treatment from web file `15`
Web file `15` covered misconfiguration generally — verbose errors, directory listing, exposed files. This module covers the specific misconfigurations that show up *because* something is an API rather than a traditional web app — CORS applied incorrectly at API scale, missing security headers on JSON responses, and verbose API error responses that leak more than a typical web error page does.

---

## Overly Permissive CORS on APIs — a bigger deal here than for a typical website

**Already covered in full mechanical depth in web file `31`.** The API-specific point worth adding: CORS misconfiguration on an API is often *more* consequential than on a traditional web app, because an API frequently returns the **entire raw data payload** directly — where a misconfigured CORS policy on a normal webpage might let an attacker's script read some rendered HTML, a misconfigured CORS policy on an API can hand over complete, structured JSON records in one read, with none of the parsing effort a scraped HTML page would require.

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```
Worth restating directly: this exact combination should never appear together (and most browsers actually block it when `*` is used literally) — but the *dynamically-reflected-origin* version of this mistake (covered in full in file `31`) remains a common, real API finding specifically because APIs are so often built to serve multiple client applications and origins, making a lazy "just allow whatever origin asks" implementation tempting during development and easy to forget to lock down before shipping.

---

## Missing Security Headers on JSON API Responses

**Why this is an API-specific gap, not just "forgot the headers":** Many of the headers covered in web file `01` (CSP, X-Frame-Options) are specifically about protecting an HTML page being *rendered* in a browser. Since an API returns raw JSON rather than a rendered page, it's common — and mostly reasonable — for `Content-Security-Policy` to simply not apply the same way. **The genuine gap** is when this reasonable exception gets over-applied, and headers that still absolutely matter for a JSON API get skipped too, specifically:

**`X-Content-Type-Options: nosniff`** — still fully relevant for an API. Without it, if a browser is ever tricked into loading an API response directly (rather than via a proper `fetch`/`XMLHttpRequest` call), MIME-sniffing could interpret unexpected JSON content as something else entirely, connecting back to the same risk covered in file `01`.

**`Content-Type: application/json` set explicitly and correctly, always** — if an API response is ever served with an incorrect or missing content type, a browser might attempt to interpret and render the raw JSON as HTML, and if any part of that JSON contains attacker-influenced text (a username, a comment field echoed back), this becomes a genuine, if unusual, XSS vector — the JSON itself becomes the payload delivery mechanism.

---

## Verbose API Error Responses — leaking more than a typical web error page

**Why APIs are specifically prone to over-sharing in error responses:** developers frequently build APIs with debugging convenience in mind first, especially for internal or early-stage APIs, and error-handling middleware that dumps a full stack trace as structured JSON is an easy, common default to leave enabled.

```json
// VULNERABLE — a real, common example
{
  "error": "Cannot read property 'id' of undefined",
  "stack": "at UserController.getProfile (/app/src/controllers/user.js:47:19)\n at ...",
  "query": "SELECT * FROM users WHERE api_key = 'sk_live_51H8x...'"
}
```
**Why this is often worse than the equivalent web-app verbose error (file `15`):** a JSON error response is structured, easy to programmatically parse and log by an attacker's own tooling, and — as shown above — API error handlers sometimes carelessly echo back the exact query or parameters that caused the error, including, in genuinely observed real-world cases, embedded credentials or API keys that were part of that failing request/query.

```json
// SECURE — generic, safe error message; full detail logged server-side only
{
  "error": "An unexpected error occurred. Reference: a1b2c3",
  "status": 500
}
```

---

## Default/Debug Configurations Left Enabled in Production

**API-specific version of this problem:** Many API frameworks ship with a "debug mode" or "development mode" that provides interactive API exploration tools (auto-generated documentation UIs, live query consoles) — genuinely useful during development, genuinely dangerous if accidentally left enabled in production.
```
https://api.target.com/graphql   →  (with GraphiQL/GraphQL Playground left enabled)
https://api.target.com/admin/docs  →  (interactive API console left accessible)
```
An interactive API exploration tool left enabled in production can let an attacker directly explore and execute API calls through a friendly UI, without needing to construct raw requests manually at all — effectively handing over a built-in testing tool for free.

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | CORS misconfiguration on an API returning full account/transaction JSON directly enables mass, structured data theft in a single read — worse in practice than the equivalent web-page CORS finding, since there's no HTML parsing overhead for the attacker |
| **Regulatory / compliance** | Verbose JSON error responses that echo back credentials or raw queries are a direct, severe finding — this pattern has been responsible for real-world credential leaks in production incidents |
| **Reputational damage** | An exposed interactive API console/playground left in production is an embarrassing, easily-discoverable finding (often turned up by automated scanners and bug bounty researchers specifically because it's a well-known checklist item) |
| **Legal liability** | Leaving debug/development tooling enabled in a production environment handling financial data is difficult to characterize as reasonable practice |
| **Operational cost** | Low remediation cost for each individual issue, but the pattern across several of these findings together often points to a broader gap in the deployment/release process rather than one-off mistakes |

**One-line interview answer:** *"API-specific misconfiguration covers the same broad category as general web misconfiguration, but the consequences are often amplified — CORS mistakes hand over structured JSON directly instead of requiring HTML scraping, and verbose JSON error handlers have a real, observed history of echoing back credentials embedded in the failing request itself, which is a more severe and more common pattern for APIs than for typical web error pages."*

## Mitigation

1. **Apply the exact same CORS discipline from file `31`**, with extra attention given the higher-value data APIs typically return directly.
2. **Never assume "it's just JSON" means headers don't matter** — `X-Content-Type-Options` and a correctly, explicitly set `Content-Type` still apply and still matter.
3. **Generic error messages to the client always; full detail logged server-side only**, and specifically audit error-handling middleware for any pattern that echoes back request parameters, queries, or configuration values.
4. **Explicitly disable debug/interactive tooling (GraphQL Playground, auto-docs consoles) in any production deployment**, verified as part of the release process, not left to a developer remembering to flip a flag.

## Quick-reference table

| Issue | API-Specific Amplification | Fix |
|---|---|---|
| CORS misconfiguration | Hands over full structured JSON directly, not scraped HTML | Same fix as file `31`, treat as higher severity |
| Missing headers | Wrongly assumed unnecessary for "just JSON" | Apply `X-Content-Type-Options` + correct `Content-Type` regardless |
| Verbose errors | JSON error handlers have echoed real credentials in practice | Generic client-facing errors, full detail server-side only |
| Debug tooling left enabled | Hands over an interactive attack console for free | Explicitly verified disabled as part of release process |
