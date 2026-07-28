# WebSocket Security

## Why this comes next
Everything so far assumed the standard request-response HTTP model. WebSockets are different — a persistent, two-way connection that stays open, commonly used for real-time features (live chat, live transaction/price feeds, notifications). Several of the protections already covered (CSRF tokens, SameSite cookies) don't automatically carry over to this different protocol, which is exactly why it needs its own dedicated coverage.

---

## What it is (in plain terms)
A WebSocket connection starts as a normal HTTP request (`Upgrade: websocket`) but then becomes a persistent, full-duplex channel — both sides can send messages at any time, not just in response to a request. Because the initial handshake is still an HTTP request, it still carries cookies automatically (the same way any HTTP request does) — but crucially, **the Same-Origin Policy protections that apply to normal HTTP requests don't automatically apply the same way here**, unless the server explicitly checks.

## Cross-Site WebSocket Hijacking (the core vulnerability)

**What it is:** The WebSocket equivalent of CSRF (file `06`) — if the server doesn't validate the `Origin` header during the WebSocket handshake, a malicious page on a completely different domain can open a WebSocket connection to the target server, and the victim's browser will automatically attach their session cookie to that handshake request, exactly as it would for CSRF.

**Example:**
```python
# VULNERABLE — accepts a WebSocket connection with no origin check at all
@sock.route('/live-transactions')
def live_transactions(ws):
    user = get_user_from_session(request.cookies)  # trusts the session cookie blindly
    while True:
        data = get_latest_transactions(user)
        ws.send(data)
```
A malicious page hosted on `attacker.com` can open a WebSocket connection directly to `wss://bank.com/live-transactions` using standard JavaScript (`new WebSocket(...)`) — the browser attaches the victim's real session cookie to the handshake automatically, the server accepts it with no origin check, and now the attacker's page receives the victim's live transaction feed directly, for as long as the connection stays open.

```python
# SECURE — explicitly validates the Origin header during the handshake
ALLOWED_ORIGINS = ['https://app.bank.com']

@sock.route('/live-transactions')
def live_transactions(ws):
    origin = request.headers.get('Origin')
    if origin not in ALLOWED_ORIGINS:
        ws.close()
        return
    user = get_user_from_session(request.cookies)
    while True:
        data = get_latest_transactions(user)
        ws.send(data)
```

**Business Impact:** Unlike a single CSRF-triggered action, an open WebSocket connection can leak an ongoing stream of live, sensitive data for as long as the victim's malicious tab stays open — for a banking application, this could mean continuously streaming a customer's live transaction activity directly to an attacker, a far larger exposure window than a single forged request.

## Missing Input Validation on WebSocket Messages

**What it is:** Once a WebSocket connection is established, messages flow in both directions freely — but it's easy to assume the connection itself being authenticated means every message on it is automatically safe, forgetting that each individual message still needs the same input validation as any other user-controlled input (the same lesson as every injection vulnerability since file `04`, just delivered over a different transport).

**Business Impact:** If message content is used to build a database query, displayed in the UI without encoding, or used to determine an action taken, all the same injection/XSS risks already covered apply here too — the persistent connection doesn't grant any special trust to what flows through it.

## Lack of Rate Limiting on Message Frequency

**What it is:** Unlike a normal HTTP request (which naturally has some overhead per request, providing a mild natural brake), a WebSocket connection can send an enormous number of messages in rapid succession with very little overhead — making this a particularly effective vector for the resource-exhaustion issues covered in file `29` if the server doesn't specifically rate-limit message frequency per connection.

**Business Impact:** A single malicious connection sending messages as fast as possible can potentially overwhelm server-side processing dedicated to handling that connection, contributing to the same availability concerns already discussed.

## Business Impact Summary

| Angle | What it actually means |
|---|---|
| **Financial loss** | A hijacked WebSocket streaming live transaction/account data is a continuous, ongoing data exposure — more severe in scope than a single-request vulnerability, since it persists for the connection's entire lifetime |
| **Regulatory / compliance** | Real-time data streams (live balances, live transaction feeds) are exactly the kind of sensitive data flow that regulatory data-protection expectations apply to — a missing origin check here is a direct, serious finding |
| **Reputational damage** | Same severity tier as other silent data-theft vulnerabilities, but with a longer typical exposure window per incident given the persistent nature of the connection |
| **Legal liability** | Missing origin validation on a WebSocket handshake is a known, documented anti-pattern with a clear, simple fix — hard to justify after the fact |
| **Operational cost** | Investigation requires understanding exactly what data any hijacked connection could have accessed and for how long it remained open, which can be harder to bound precisely than a single discrete HTTP request |

**One-line interview answer:** *"Technically, WebSocket hijacking is the same underlying problem as CSRF — the handshake carries the victim's session cookie automatically, and without an origin check, a malicious site can open a connection using it. For a bank, the business impact is actually worse than typical CSRF, because instead of one forged action, the attacker gets a live, ongoing stream of the victim's data for as long as the connection stays open, not just a single moment of exposure."*

## Mitigation

1. **Validate the `Origin` header during the WebSocket handshake (the real fix)** — exactly the same allow-list principle as CORS (file `31`), applied to this different connection type.
2. **Apply the same input validation to WebSocket messages as any other user input** — don't assume the persistent connection grants any special trust to what flows through it.
3. **Rate-limit message frequency per connection** — protects against the resource-exhaustion risk unique to how cheaply WebSocket messages can be sent compared to full HTTP requests.
4. **Use a proper authentication token exchanged after the handshake** (rather than relying solely on the cookie automatically attached during the initial upgrade request) for particularly sensitive WebSocket use cases, adding a layer independent of cookie-based session trust.

## Explaining it to a developer
*"WebSocket connections still carry the session cookie automatically during the initial handshake, the same way any HTTP request does — but the browser's usual same-origin protections don't apply here the way they do elsewhere, unless we explicitly check the Origin header ourselves. Without that check, any website can open a connection to this endpoint using a logged-in victim's real session, and because it's a persistent connection, they'd get a continuous stream of data, not just one response. The fix is the same idea as CORS — check the Origin against a specific list of our own trusted domains during the handshake, and reject anything else."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Origin validation during handshake | Cross-site WebSocket hijacking |
| Input validation on messages | Injection/XSS risks flowing through the connection |
| Rate limiting per connection | Message-flood-based resource exhaustion |
| Post-handshake token authentication | Adds trust independent of cookie-based session alone |
