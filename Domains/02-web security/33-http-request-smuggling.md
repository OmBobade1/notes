# HTTP Request Smuggling

## Why this comes next
Most modern web applications sit behind at least one intermediary — a load balancer, reverse proxy, or CDN — before the request reaches the actual application server. HTTP Request Smuggling exploits a discrepancy in how the front-end intermediary and the back-end server each interpret where one request ends and the next one begins, letting an attacker "smuggle" a hidden second request that the back-end processes but the front-end never saw as a separate request.

---

## What it is (in plain terms)
When a client sends an HTTP request with a body, the server needs to know exactly where that body ends — this is normally determined by either a `Content-Length` header (an exact byte count) or `Transfer-Encoding: chunked` (the body is sent in labeled chunks, ending with a zero-length chunk). Request smuggling happens when a request includes **both** headers, or an ambiguous/malformed version of one, and the front-end proxy and back-end server disagree about which one to trust — each interpreting the boundary differently, leaving a portion of the request that one system considers part of the current request, and the other considers the *start of a brand new one*.

## Why it exists — the real-life cause

```
POST / HTTP/1.1
Host: bank.com
Content-Length: 13
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
Host: bank.com
...
```
This single request includes **both** `Content-Length` and `Transfer-Encoding: chunked` — which should never happen in a well-formed request. If the front-end proxy processes this using `Content-Length` (treating everything as one request of exactly 13 bytes, ending right after the `0\r\n\r\n` chunked-terminator sequence), but the back-end server processes it using `Transfer-Encoding` instead (correctly parsing the zero-length chunk as the end of *that* request, then treating everything after it — the `GET /admin...` line — as the beginning of a completely separate, new request), the two systems now disagree about where one request ends and the next begins.

The practical consequence: that smuggled `GET /admin` fragment gets prepended onto whatever the *next* real user's request happens to be, on the same persistent connection between the proxy and the back-end (these connections are often reused/pooled across multiple different users' requests for performance). This can result in one user's request getting mixed with another's, response queue poisoning (users receiving responses meant for someone else), or bypassing access controls that only the front-end proxy enforces.

## Why this isn't something to "fix with a code snippet"
Unlike SQLi or XSS, this isn't a single application-level mistake — it's a discrepancy between two *different pieces of infrastructure* (the specific proxy/load balancer software and version, and the specific back-end server software and version) disagreeing on HTTP parsing edge cases. The fix is architectural and configuration-based, not a code change in the application itself.

## How an attacker actually does it, step by step
1. Identify that the target sits behind a reverse proxy/load balancer (common in virtually all production deployments) — visible through response header differences, timing characteristics, or simply assuming it given standard architecture.
2. Send probing requests with ambiguous `Content-Length`/`Transfer-Encoding` combinations and observe response timing or behavior differences that indicate the front-end and back-end disagree on request boundaries (specialized tools like Burp Suite's HTTP Request Smuggler extension automate this detection).
3. Once a discrepancy is confirmed, craft a smuggled request designed to achieve a specific goal — bypassing a front-end access control check (since the smuggled request never passed through the front-end's own inspection), or capturing another user's request/response on a shared connection.

## Technical Impact
- **Access control bypass** — if the front-end proxy enforces certain restrictions (e.g. blocking direct access to `/admin`), a smuggled request can reach the back-end directly, skipping that front-end check entirely
- **Request/response confusion between users** — on connection-reused/pooled architectures, a smuggled request fragment can become prepended to another user's legitimate request, potentially exposing one user's response to a different user, or vice versa
- **Cache poisoning** — if a caching layer is involved, a smuggled request can poison what gets cached and subsequently served to other users (related to, but distinct from, the CRLF-based version in file `22`)

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | If request smuggling enables one user's session/response to be exposed to another user, this could mean one customer momentarily seeing another customer's account information — a serious, hard-to-explain confidentiality breach |
| **Regulatory / compliance** | This vulnerability class specifically demonstrates infrastructure-level gaps (proxy/backend configuration mismatch) rather than application code issues — auditors treat this as an architecture-level finding requiring infrastructure team involvement, not just a development fix |
| **Reputational damage** | User-to-user data/response confusion, if it occurred in the wild, is a distinctly alarming category of incident for customers to hear about — "another customer saw my data" is a uniquely visceral kind of breach report |
| **Legal liability** | Given how well-documented this vulnerability class has become in recent security research, running a known-vulnerable proxy/backend combination without mitigation is increasingly difficult to characterize as reasonable, unavoidable risk |
| **Operational cost** | Remediation often requires infrastructure-level changes (upgrading or reconfiguring the proxy and/or backend, standardizing on HTTP/2 for backend connections where feasible) rather than a simple application code fix, making this more disruptive to remediate than most vulnerabilities in this series |

**One-line interview answer:** *"Technically, request smuggling exploits a disagreement between a front-end proxy and back-end server about where one HTTP request ends and the next begins, usually via conflicting Content-Length and Transfer-Encoding headers — letting an attacker smuggle a hidden request that gets processed separately by the back-end. For a bank, the most concerning outcome is user-to-user request/response confusion on shared connections, which could mean one customer's session data briefly crossing into another's — and because it's an infrastructure-level mismatch rather than an application bug, fixing it usually requires coordination between development and infrastructure teams, not just a code change."*

## Mitigation

1. **Disable connection reuse between the front-end proxy and back-end where feasible**, or ensure both ends strictly and identically reject ambiguous requests (any request containing both `Content-Length` and `Transfer-Encoding` should be rejected outright by both layers, not silently resolved by picking one).
2. **Normalize on HTTP/2 for back-end connections** where supported — HTTP/2's framing mechanism doesn't have this specific ambiguity, since message boundaries are defined structurally rather than via a header that could conflict with another.
3. **Keep both the front-end proxy and back-end server software up to date** — this connects to file `18`; known request-smuggling-enabling parsing bugs in specific software versions get patched over time as they're discovered.
4. **Use a single, consistent technology/vendor for request parsing across the entire request path** where architecturally feasible, reducing the chance of two different parsers interpreting the same bytes differently.

## Explaining it to a developer
*"This isn't something fixable with a code change in the application itself — it's about our proxy and our backend server potentially disagreeing on where one HTTP request ends and the next begins, when a request includes conflicting length information. The practical risk is that an attacker can smuggle a hidden second request that only the backend sees, potentially bypassing checks the proxy normally enforces, or even getting mixed up with a completely different user's request on a shared connection. This needs to go to whoever manages our infrastructure/proxy configuration, since the fix is at that layer, not in application code."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Reject ambiguous Content-Length/Transfer-Encoding combos at both layers | Removes the parsing ambiguity that enables smuggling |
| Disable/limit connection reuse between proxy and backend | Reduces the blast radius even if smuggling occurs |
| Normalize on HTTP/2 for backend connections | Eliminates this specific class of framing ambiguity |
| Keep proxy/backend software updated | Reduces exposure to known parser-specific bugs |
