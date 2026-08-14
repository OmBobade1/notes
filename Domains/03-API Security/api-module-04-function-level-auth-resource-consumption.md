# Module 04 - Broken Function Level Authorization & Unrestricted Resource Consumption

## Why these two share a module
Both are about a missing *limit* — one on which functions a caller can reach, the other on how much a caller can demand in a single request or session. Different mechanisms, same underlying theme: the API trusted the caller to behave reasonably instead of enforcing a hard boundary.

---

## Broken Function Level Authorization (API Context)

### Why this needs its own API-specific treatment, beyond web file `23`
The web security series covered this as "a hidden admin button doesn't stop someone calling the endpoint directly." For APIs, there's an even more direct version of the same problem: **there often isn't a UI at all to hide anything behind** — API endpoints are called directly by mobile apps, other services, or scripts, meaning the entire "hide the button" concept doesn't even apply. The only thing standing between a regular user and an admin-only API function is whatever the server actually checks — full stop.

### Why it exists — the real API-specific cause
```javascript
// VULNERABLE — only checks that a token is valid, not what role it belongs to
app.delete('/api/users/:id', authenticate, (req, res) => {
  db.deleteUser(req.params.id);
  res.json({ success: true });
});
```
Nothing here distinguishes a regular user's valid token from an admin's valid token — `authenticate` only confirms *a* legitimate session exists, not *which kind*. A regular customer, simply by knowing (or discovering, via API documentation, mobile app decompilation, or guessing based on REST conventions) that a `DELETE /api/users/:id` endpoint exists, can call it directly with their own valid token.

```javascript
// SECURE — explicitly checks the caller's role before performing an admin-level action
app.delete('/api/users/:id', authenticate, requireRole('admin'), (req, res) => {
  db.deleteUser(req.params.id);
  res.json({ success: true });
});
```

### How this gets discovered in practice, specifically for APIs
**API documentation itself is often the leak:** many organizations publish OpenAPI/Swagger documentation describing every available endpoint, including admin-only ones — intended for legitimate developers integrating with the API, but equally readable by anyone testing it.
```
https://api.target.com/swagger.json
https://api.target.com/openapi.yaml
```
Finding and reading this documentation can hand an attacker a complete list of every function-level endpoint that exists, admin ones included, without any guessing required at all.

**Mobile app decompilation** — since a mobile app has to know the API's endpoint structure to function, decompiling the app (covered in the mobile security series) frequently reveals the exact URL patterns for every function the app calls, including admin/internal-only routes the app's own regular-user interface never exposes a button for.

### Business Impact
For a banking API specifically, an unrestricted admin-level function (issuing refunds, adjusting account status, viewing any customer's full profile) reachable by a regular authenticated customer is a direct, repeatable path to fraud — and because there's no UI gate at all in the API-only case, discovery via documentation or app decompilation is often *easier* than for a traditional web app, not harder.

---

## Unrestricted Resource Consumption

### Why this deserves separate, deeper treatment than web file `29`
The Application-Layer DoS content in the web series covered ReDoS and general resource exhaustion. APIs have a few genuinely distinct resource-consumption risks tied specifically to how API requests are typically structured — pagination, batch operations, and file/payload size.

### Pagination Abuse
**What it is:** many list-returning endpoints accept a `limit` or `pageSize` parameter controlling how many records to return per request — if the server doesn't cap this value, a client can request an enormous number of records in one call.
```javascript
// VULNERABLE — no upper bound enforced
app.get('/api/transactions', (req, res) => {
  const limit = parseInt(req.query.limit) || 20;
  res.json(db.getTransactions(limit));  // client can request limit=9999999
});
```
```
GET /api/transactions?limit=9999999
```
A single request like this can force the server to fetch and serialize an enormous dataset, consuming excessive memory and processing time — the same core resource-exhaustion concept from web file `29`, but specifically enabled by a pagination parameter, a pattern that's nearly universal across REST APIs specifically.

```javascript
// SECURE — hard maximum, regardless of what's requested
app.get('/api/transactions', (req, res) => {
  const limit = Math.min(parseInt(req.query.limit) || 20, 100);  // hard ceiling
  res.json(db.getTransactions(limit));
});
```

### Batch Operation Abuse
**What it is:** APIs often offer batch endpoints to efficiently process multiple items in one request (e.g. bulk-creating records). If the batch size itself isn't capped, a single request can trigger an enormous amount of backend processing.
```json
POST /api/batch/transfer
{ "transfers": [ /* 100,000 individual transfer objects in one array */ ] }
```
One HTTP request, potentially thousands of expensive individual database operations triggered from it — a highly efficient resource-exhaustion vector specifically because it's disguised as one single, seemingly ordinary API call.

### File Upload / Payload Size Abuse
**Connects to file `08` in the web series (file upload) from a different angle:** even ignoring malicious file *content* entirely, an API accepting file uploads or large JSON payloads with no explicit size limit can be sent an extremely large payload purely to exhaust server memory/bandwidth/disk, independent of whether the content itself is otherwise "valid."

### GraphQL-Specific Resource Consumption (connects to Module 00's GraphQL introduction)
**Why GraphQL has its own distinct version of this problem:** because the client controls query shape and nesting depth directly (Module 00), a deeply nested or overly broad query can force the server to perform an exponentially expanding amount of work to resolve it.
```graphql
query {
  user(id: 1) {
    friends {
      friends {
        friends {
          friends { name }
        }
      }
    }
  }
}
```
Each nesting level potentially multiplies the number of database lookups required — a query like this, sent against an API with no depth or complexity limiting, can trigger a computational cost wildly disproportionate to how simple the request looks on the surface.

**Mitigation specific to GraphQL:** implement explicit query depth limiting and query complexity scoring (assigning a "cost" to each field/nesting level and rejecting queries exceeding a defined budget) — libraries exist specifically for this in most GraphQL server frameworks, since it's a well-known, common gap.

## Business Impact Summary

| Angle | What it actually means |
|---|---|
| **Financial loss** | Every minute of API downtime directly blocks mobile banking transactions in real time — API outages are felt immediately by every connected client simultaneously |
| **Regulatory / compliance** | API availability is explicitly relevant to operational-resilience expectations for banking systems, same as the general DoS content, but the specific vectors (pagination, batch, GraphQL depth) are API-specific findings auditors look for distinctly |
| **Reputational damage** | A resource-exhaustion incident affects every client hitting the API simultaneously — mobile app users experience this as "the app is down," a highly visible, immediate failure |
| **Legal liability** | Missing pagination/batch limits are well-documented, easily-testable gaps — straightforward to characterize as a foreseeable oversight |
| **Operational cost** | Emergency capacity scaling or incident response for an API outage, plus the underlying fix |

**One-line interview answer:** *"Function-level authorization for APIs is often even more exposed than for web apps, because there's frequently no UI hiding the admin function at all — API documentation or a decompiled mobile app can hand an attacker the exact endpoint directly. Resource consumption for APIs specifically shows up through pagination limits, batch operation size, and — for GraphQL — query depth, since the client controls the query shape directly and can request exponentially expanding amounts of work in a single, innocent-looking call."*

## Mitigation

1. **Explicit role checks on every function-level endpoint**, especially ones only discoverable via documentation or app decompilation rather than a visible UI — never assume obscurity is protection.
2. **Treat API documentation itself as something requiring access control** where it describes internal/admin-only functionality — don't publish a complete map of privileged endpoints to anyone who can reach the docs URL.
3. **Hard-cap pagination limits, batch operation sizes, and payload/file sizes server-side**, regardless of what the client requests.
4. **Implement GraphQL-specific query depth and complexity limiting** as a standard part of any GraphQL API's configuration, not an afterthought.
5. **Rate-limit at the API gateway level**, providing a consistent backstop across every endpoint rather than relying on each individual route to implement its own protection correctly.

## Quick-reference table

| Issue | Root Cause | Fix |
|---|---|---|
| Broken function-level auth (API) | Role never checked, no UI to even hide the button behind | Explicit role checks server-side, every endpoint |
| Pagination abuse | No cap on `limit`/`pageSize` | Hard server-side maximum |
| Batch operation abuse | No cap on batch array size | Hard server-side maximum |
| GraphQL depth/complexity | Client controls query nesting freely | Depth limiting + complexity scoring |
