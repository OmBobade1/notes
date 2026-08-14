# Module 01 - Broken Object Level Authorization (BOLA)

## Why this is #1 on the OWASP API Top 10, and where you've already seen this idea
This is the exact same root problem as IDOR in the web security series (file `07`) — the server checks that you're authenticated, but never checks that the specific object/record you're asking for actually belongs to you. It's ranked #1 for APIs specifically because APIs expose object IDs directly and constantly, by design, making this the single most common real-world API finding.

---

## What it is, applied specifically to APIs
Every API endpoint that accepts an object identifier (a user ID, an order ID, an invoice number) in the URL or request body is a potential BOLA target if the server doesn't independently verify the authenticated caller actually owns or is authorized to access that specific object — it only checks that *some* valid authentication token was presented.

## Why it exists — the real API-specific cause

```javascript
// VULNERABLE — fetches whatever order ID is given, no ownership check
app.get('/api/orders/:orderId', authenticate, (req, res) => {
  const order = db.getOrder(req.params.orderId);  // just uses the ID, trusts the caller
  res.json(order);
});
```
`authenticate` middleware confirms a valid token was sent — but says nothing about whether *this specific token's owner* is allowed to see *this specific order*. A logged-in user viewing their own order at `/api/orders/5001` can simply change the URL to `/api/orders/5002` and receive someone else's order data directly.

```javascript
// SECURE — verifies the requesting user actually owns this specific object
app.get('/api/orders/:orderId', authenticate, (req, res) => {
  const order = db.getOrder(req.params.orderId);
  if (!order || order.userId !== req.user.id) {
    return res.status(404).send();  // 404, not 403 — see the note below
  }
  res.json(order);
});
```

**Why returning 404 instead of 403 here is a deliberate, correct choice, not an oversight:** returning 403 ("Forbidden") confirms to an attacker that order `5002` genuinely exists, just isn't theirs — valuable reconnaissance, as covered in Module 00's status code discussion. Returning 404 for both "doesn't exist" and "exists but isn't yours" denies the attacker that distinction entirely.

---

## How an attacker actually finds and exploits this, step by step

1. Log in normally with a real, legitimate account, and identify any endpoint that references an object by ID — `/api/orders/{id}`, `/api/users/{id}/profile`, `/api/invoices/{id}`.
2. Note the ID belonging to your own legitimate object (e.g. your own order is `5001`).
3. Change the ID to a nearby value (`5000`, `5002`) and resend the exact same authenticated request.
4. If the response returns another user's data instead of an error, BOLA is confirmed.
5. **Automate the discovery at scale** — this is the part that makes BOLA especially dangerous for APIs specifically:
```
for i in $(seq 1 10000); do
  curl -s -H "Authorization: Bearer $TOKEN" https://api.target.com/orders/$i -o "order_$i.json"
done
```
A simple loop like this, run with your own valid token, can potentially pull every single order in the entire system — this is trivially scriptable specifically because API responses are structured JSON, easy to parse and store automatically, unlike scraping data out of rendered HTML pages.

---

## Why BOLA is specifically worse for APIs than IDOR often is for traditional web apps

**Predictable ID schemes are far more common and more exposed in APIs:** REST's own convention (`/resource/{id}`) practically advertises the pattern to test — you don't need to find the vulnerable parameter, the API's own URL structure hands it to you directly, unlike a web app where the vulnerable ID might be buried in a hidden form field or POST body you'd have to discover first.

**Automation is trivial:** as shown above, a simple script can enumerate thousands of objects in minutes, since there's no HTML rendering, no CAPTCHA-protected forms typically in the way, and clean, structured JSON output that requires zero parsing effort to process at scale.

**Mobile apps make this worse in practice:** a mobile app's API calls can be intercepted (using a proxy like Burp Suite configured for mobile traffic) and then replayed/modified directly, and mobile app users are statistically less likely to notice or report unusual behavior compared to a web user watching a browser's address bar.

---

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | For a banking API, BOLA on an accounts/transactions endpoint means one compromised or even legitimate low-privilege account can potentially enumerate every customer's financial data — this is a mass-breach vector, not a single-user issue |
| **Regulatory / compliance** | Automatable, mass exposure of financial records via simple ID iteration is treated as a severe, systemic access-control failure in any banking security review — the automatable-at-scale nature specifically elevates severity beyond a single-record IDOR |
| **Reputational damage** | Because exploitation is trivially scriptable, a single unpatched BOLA endpoint can be exploited against the entire user base within minutes of discovery, not gradually — breach disclosure at that scale is a major public event |
| **Legal liability** | Mass data exposure through a well-known, top-of-the-list vulnerability class (BOLA has been #1 on the OWASP API Top 10 across multiple revisions) is difficult to characterize as anything but a foreseeable, preventable failure |
| **Operational cost** | Investigating exactly which records were actually accessed (versus just theoretically exposed) at API scale often requires extensive log analysis across every affected endpoint |

**One-line interview answer:** *"BOLA is the API-specific version of IDOR — the server checks you're authenticated, but never checks you're authorized for the specific object you're requesting. It's ranked #1 on the OWASP API Top 10 because APIs expose object IDs directly in predictable URL patterns and return clean, structured JSON — both of which make mass automated enumeration trivial, turning a single missing check into a potential full-database exposure within minutes."*

---

## Mitigation

1. **Enforce object-level ownership checks on every single endpoint that accepts an object ID (the real fix)** — verify the authenticated user actually owns or is authorized for that specific object, every time, server-side.
2. **Use indirect, non-sequential object references** — UUIDs instead of sequential integers — so even attempting to guess a nearby valid ID doesn't work; this doesn't replace the ownership check, but removes the trivial "just increment the number" enumeration path.
3. **Centralize authorization logic** in shared middleware, rather than re-implementing the ownership check by hand in every individual route — a single missed endpoint reintroduces the vulnerability.
4. **Rate-limit and monitor for enumeration patterns** — a single account rapidly requesting sequential or widely-varying object IDs is a detectable pattern worth alerting on specifically.
5. **Return identical responses (404) for "doesn't exist" and "exists but not yours"** — denying an attacker the reconnaissance value of distinguishing the two cases.

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Server-side ownership check on every object-ID endpoint | The core vulnerability itself |
| UUIDs instead of sequential IDs | Trivial "just increment the number" enumeration |
| Centralized authorization middleware | A missed check on one endpoint reintroducing the flaw |
| Rate limiting / enumeration monitoring | Mass automated exploitation, even if a flaw briefly exists |
| Identical 404 for not-found vs not-authorized | Removes reconnaissance value from probing IDs |
