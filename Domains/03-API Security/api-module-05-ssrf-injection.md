# Module 05 - SSRF & Injection (API-Specific Angles)

## Why this module doesn't repeat the web security content
SSRF (file `09`) and injection (files `04`, `11`, `20`) are already covered in full technical depth in the web series. This module covers something different: the specific *shapes* these same vulnerability classes take when the target is an API rather than a traditional web form — webhooks, GraphQL, and JSON-native injection patterns that don't map directly onto the earlier content.

---

## SSRF via Webhooks — the API-specific delivery mechanism

**Why webhooks are practically the API version of the "upload from URL" SSRF pattern:** A webhook is a URL an API owner provides so that *your* application gets notified automatically when something happens (a payment completes, an order updates) — the API server itself makes an outbound HTTP request to whatever URL you registered. This is functionally identical to the SSRF pattern in web file `09`, just reframed as a first-class, expected API feature rather than an edge-case "fetch a URL" form field.

**Why this makes it a more commonly *available* SSRF vector, not a rarer one:** Because webhook registration is a standard, documented feature of most modern APIs (payment processors, CRM platforms, SaaS integrations), it's a near-guaranteed thing to test on any API that offers it — unlike web file `09`'s SSRF, which required specifically finding a "fetch from URL" feature, webhook SSRF testing is close to a default checklist item for any API with this feature at all.

```
POST /api/webhooks
{ "url": "http://169.254.169.254/latest/meta-data/", "events": ["payment.completed"] }
```
If the API server itself makes a request to validate/ping the registered webhook URL upon registration (a common pattern, to confirm the endpoint is reachable before saving it), this alone can trigger the SSRF — the same cloud metadata endpoint exposure covered in web file `09`, reached through a completely different, API-native feature.

**Mitigation, specific to webhooks:** validate the registered URL against the same allow-list/private-IP-blocking rules covered in web file `09`, and critically, perform that validation **every time** the webhook actually fires, not just once at registration time — a URL that resolves safely at registration could be changed via DNS to point somewhere internal by the time the webhook actually triggers later (a **DNS rebinding** attack specifically relevant here).

---

## SSRF via API-Driven File/Image Processing

**A second common API-specific pattern:** APIs that accept a URL to fetch and process an image, PDF, or document (thumbnail generation, PDF-from-URL conversion, avatar-from-URL features) — architecturally identical to web file `09`'s core example, but worth naming specifically because it's an extremely common API feature in practice, not a rare one.
```
POST /api/documents/generate-pdf
{ "sourceUrl": "http://internal-admin-panel.local/sensitive-report" }
```
If the API server fetches and renders whatever URL is provided into a PDF, and that PDF is then returned to the requester, this becomes a way to exfiltrate the *content* of an internal-only resource by having the server fetch it and hand you back a rendering of it — even if the raw internal resource itself was never directly reachable to you.

---

## GraphQL Injection

**Why this needs separate treatment from standard SQLi/NoSQLi:** GraphQL's structured query language sits *between* the client and the actual database query — but if the GraphQL resolver (the backend code that translates a GraphQL field into an actual database call) builds its underlying query unsafely, the exact same injection principle from file `04`/`20` applies, just one layer removed.

```javascript
// VULNERABLE — GraphQL resolver builds a raw query using unescaped input
const resolvers = {
  Query: {
    user: (parent, args) => {
      return db.query(`SELECT * FROM users WHERE username = '${args.username}'`);
    }
  }
};
```
The GraphQL layer itself doesn't introduce this vulnerability — it's the exact same unsafe string concatenation from file `04`, just happening inside a resolver function instead of a traditional route handler. **Why it's worth calling out specifically:** GraphQL's flexible query structure means an attacker can often reach deeply nested resolvers in ways that aren't obvious from a simple API surface scan, potentially finding an injectable resolver buried several levels deep in a complex query that wouldn't be discovered by testing only the "obvious" top-level fields.

```javascript
// SECURE — same fix as file 04, applied inside the resolver
const resolvers = {
  Query: {
    user: (parent, args) => {
      return db.query('SELECT * FROM users WHERE username = ?', [args.username]);
    }
  }
};
```

---

## NoSQL Injection via JSON-Native APIs

**Why this is specifically more relevant for APIs than for traditional web forms:** Web file `20` covered NoSQL injection via a form sending an unexpected JSON object where a string was expected. APIs are **JSON-native by default** — every single request body is already JSON, meaning this exact attack requires zero extra effort to attempt; you're not converting a form submission into JSON, you're just modifying JSON that was already the expected format.

```json
POST /api/login
{ "username": "admin", "password": { "$ne": null } }
```
This is the identical technique from file `20`, but worth emphasizing here: because every API request is already structured JSON, testing for this specific injection is simply a matter of trying an object where a string is expected, on literally every JSON field across the entire API surface — a fast, broad, low-effort test to run systematically.

---

## Command Injection via API Parameters

**The API-specific angle beyond file `12`:** Any API endpoint that triggers a backend process — a report generation job, a file conversion, a network diagnostic feature exposed via API — is a potential command injection target, following the exact same principles from file `12`. The only difference worth noting is that API parameters are often *less scrutinized* than user-facing form fields, since developers may reasonably-but-incorrectly assume "this is an internal/programmatic API, not directly user-facing, so it doesn't need the same input validation rigor" — a dangerous assumption, since any API reachable from outside is exactly as attacker-facing as a web form, regardless of who the "intended" caller is.

## Business Impact Summary

| Angle | What it actually means |
|---|---|
| **Financial loss** | Webhook-based SSRF reaching cloud metadata endpoints carries the identical severe risk as file `09` — credential theft leading into the broader cloud account — just reached through a more commonly-available, standard API feature |
| **Regulatory / compliance** | Injection via GraphQL resolvers or NoSQL-native JSON bodies represents the same severity as traditional injection, but auditors specifically check whether API-specific surfaces (resolvers, nested queries) were included in testing scope, since they're easy to overlook if an assessment only tests "the obvious" endpoints |
| **Reputational damage** | Same as the underlying vulnerability class in each case — SSRF and injection outcomes don't change based on delivery mechanism, only the path to reach them does |
| **Legal liability** | Assuming "internal-only" API parameters don't need the same validation rigor as public-facing forms is a documented, well-known bad assumption — difficult to defend once an API is reachable from outside at all |
| **Operational cost** | Same as underlying vulnerability class — full incident response scope depends on what was actually reached, not on which specific API feature was the entry point |

**One-line interview answer:** *"SSRF and injection aren't new vulnerability classes for APIs — they're the same underlying flaws from the web security content, but APIs offer new, often more commonly available delivery mechanisms: webhooks and URL-based file processing for SSRF, and GraphQL resolvers or JSON-native request bodies for injection. The risk is identical once triggered; what changes is how easy and how likely it is to be discovered, since these API-specific patterns are often standard features rather than rare edge cases."*

## Mitigation
All mitigations from web files `04`, `09`, `11`, `12`, and `20` apply directly and fully — nothing about the API context changes the actual fix. The additional API-specific points:
1. **Validate webhook URLs on every trigger, not just at registration** — defeating DNS rebinding attacks specifically.
2. **Apply the same input validation rigor to every API parameter as to any public-facing web form field** — never assume an API consumer is trusted just because it's "not a browser."
3. **Test GraphQL resolvers individually, at every nesting depth** — a broad top-level scan can miss an injectable field buried several levels into a complex query.
4. **Treat every JSON request body field as a potential NoSQL injection point by default** — systematically test object-substitution on every field, since the JSON-native format makes this attempt essentially free to try broadly.

## Quick-reference table

| API-Specific Vector | Underlying Vulnerability Class | Web Series Reference |
|---|---|---|
| Webhook registration/trigger | SSRF | File `09` |
| URL-based file/PDF/image processing | SSRF | File `09` |
| GraphQL resolver building unsafe queries | SQL/NoSQL Injection | Files `04`, `20` |
| JSON body object-substitution | NoSQL Injection | File `20` |
| API-triggered backend processes | Command Injection | File `12` |
