# Module 03 - Broken Object Property Level Authorization

## Why this is different from BOLA (Module 01)
BOLA asks "can I access this *object* at all?" This module asks a narrower question: even for an object you're legitimately allowed to access, "can I see or change *fields* on it I shouldn't?" Same family of problem, one level more granular — it's specifically about individual properties/fields within an object, not the object as a whole.

This OWASP category actually combines two related but distinct issues: **Excessive Data Exposure** (seeing fields you shouldn't) and **Mass Assignment** (changing fields you shouldn't).

---

## Excessive Data Exposure

### What it is
An API endpoint returns more fields in the response than the client actually needs or should see — often because the backend simply serializes and returns an entire database record, relying on the frontend to just not *display* the extra fields, rather than the server withholding them in the first place.

### Why it exists — the real cause
```javascript
// VULNERABLE — returns the entire user object as-is
app.get('/api/users/:id', authenticate, (req, res) => {
  const user = db.getUser(req.params.id);
  res.json(user);  // includes EVERYTHING in the database record
});
```
If the `user` database record includes fields like `passwordHash`, `internalNotes`, `ssnLastFour`, or another customer's linked account ID, all of it gets sent in the response — even if the actual app's frontend only ever displays the name and email. **The extra fields are still fully visible to anyone inspecting the raw network response**, trivial with browser dev tools or a proxy like Burp Suite, entirely independent of what the visible UI happens to render.

```javascript
// SECURE — explicitly whitelist only the fields that should ever be returned
app.get('/api/users/:id', authenticate, (req, res) => {
  const user = db.getUser(req.params.id);
  res.json({
    id: user.id,
    name: user.name,
    email: user.email
    // passwordHash, internalNotes, etc. are simply never included
  });
});
```

**Why relying on "the frontend just doesn't show it" is never sufficient:** this is the exact same lesson as the sensitive data exposure content in the web security series, applied specifically to API response bodies — the security boundary has to be enforced by the server deciding what to *send*, never by trusting the client to politely only *display* part of what it received.

**How an attacker actually finds this — it's often just reading the response, nothing clever required:**
```
curl -H "Authorization: Bearer $TOKEN" https://api.target.com/users/101 | jq
```
`jq` pretty-prints the full raw JSON response — simply reading the complete response (rather than only looking at what the app's UI shows) is frequently all it takes to spot fields that were never meant to be exposed.

---

## Mass Assignment

### What it is
The reverse direction of the same underlying problem: an API endpoint that accepts a JSON object for creating/updating a record automatically maps *every* field in the submitted JSON directly onto the database record, without restricting which fields the client is actually allowed to set.

### Why it exists — the real cause
```javascript
// VULNERABLE — blindly assigns every field the client sends
app.put('/api/users/:id', authenticate, (req, res) => {
  db.updateUser(req.params.id, req.body);  // whatever the client sent, gets written
  res.json({ success: true });
});
```
A legitimate request might update just `{"name": "New Name"}`. But since the endpoint applies *whatever* JSON is submitted directly onto the record, nothing stops a request like:
```json
{
  "name": "New Name",
  "role": "admin",
  "accountBalance": 999999,
  "isVerified": true
}
```
If `role`, `accountBalance`, or `isVerified` are real fields on the underlying user record — and the server never restricted which fields a normal user-facing "update profile" endpoint is allowed to touch — the attacker just granted themselves admin privileges, or directly edited their own account balance, simply by including extra fields the frontend's own form never intended to expose, but the backend never explicitly blocked either.

```javascript
// SECURE — explicitly whitelist which fields this endpoint is allowed to update
app.put('/api/users/:id', authenticate, (req, res) => {
  const allowedFields = ['name', 'email', 'phone'];
  const updates = {};
  for (const field of allowedFields) {
    if (req.body[field] !== undefined) updates[field] = req.body[field];
  }
  db.updateUser(req.params.id, updates);  // only whitelisted fields can ever be written
  res.json({ success: true });
});
```

### How an attacker actually finds and exploits this, step by step
1. Identify a create/update endpoint (`POST`/`PUT`/`PATCH`) and capture a normal, legitimate request via an intercepting proxy.
2. Look at Module 03's earlier Excessive Data Exposure finding on a related `GET` endpoint for the *same* object type — a response that leaked field names like `role` or `isAdmin` directly hands you the exact field names worth trying to inject on the write side. **This is exactly why these two vulnerabilities are grouped in the same OWASP category — one frequently reveals the field names needed to exploit the other.**
3. Add the discovered field name(s) to a normal, legitimate-looking update request with an attacker-favorable value.
4. Resend the modified request and check whether the added field was actually applied — confirm by re-fetching the object afterward.

---

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | Mass assignment on a field like account balance, credit limit, or role directly enables financial manipulation or privilege escalation with a single crafted request — no separate exploitation technique needed beyond adding an extra JSON field |
| **Regulatory / compliance** | Excessive data exposure of fields like partial SSNs, internal risk scores, or other customers' linked data is a direct confidentiality breach, and mass assignment enabling privilege escalation is a severe access-control failure — both are foundational-level findings in any banking API review |
| **Reputational damage** | Both vulnerabilities are typically silent — a customer using the app normally has no way to know extra fields were exposed in the raw response, or that another user somewhere maliciously mass-assigned their way to elevated privileges |
| **Legal liability** | "We returned the whole database record and trusted the frontend not to display it" and "we accepted every field the client sent without restriction" are both well-documented anti-patterns with clear, known correct alternatives (whitelisting) — difficult to defend as reasonable practice |
| **Operational cost** | Mass assignment incidents specifically require auditing exactly which accounts had fields maliciously modified, not just which data was viewed — a more invasive, harder cleanup than a pure data-exposure incident |

**One-line interview answer:** *"These are two sides of the same coin — excessive data exposure is the API returning more fields than it should on read, and mass assignment is the API accepting more fields than it should on write. They're grouped together because a leaky read endpoint frequently reveals the exact internal field names — like 'role' or 'isAdmin' — that make the write-side mass assignment attack possible in the first place."*

## Mitigation

1. **Explicitly whitelist response fields on every endpoint (the real fix for data exposure)** — never serialize and return an entire database record directly; construct the response object deliberately, field by field.
2. **Explicitly whitelist which fields a given endpoint is allowed to write (the real fix for mass assignment)** — never pass a raw request body directly into a database update call.
3. **Use dedicated DTOs (Data Transfer Objects) or schema validation libraries** that enforce field whitelisting structurally, at the framework level, rather than relying on developers remembering to whitelist manually on every single endpoint.
4. **Treat sensitive fields (role, permissions, balance, verification status) as never client-settable through standard update endpoints at all** — changes to these should go through a separate, more tightly controlled, audited process/endpoint entirely, not the same general-purpose "update my profile" route.

## Quick-reference table

| Vulnerability | Direction | Root Cause | Fix |
|---|---|---|---|
| Excessive Data Exposure | Read (response) | Returning entire records, relying on frontend to hide fields | Explicit response field whitelisting |
| Mass Assignment | Write (request) | Accepting entire request bodies, applying all fields blindly | Explicit writable field whitelisting |
