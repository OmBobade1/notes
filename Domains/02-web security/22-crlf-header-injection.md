# CRLF / HTTP Header Injection

## Why this comes next
Same family again — user input reaching somewhere it can be misread as structure rather than data. This time the "structure" is the HTTP response itself: headers and body are separated by a specific character sequence (CRLF — Carriage Return + Line Feed, `\r\n`), and if user input can include that sequence, an attacker can inject entirely new headers or even a fake response body.

---

## What it is (in plain terms)
HTTP responses are structured as: status line, then headers, then a blank line, then the body — each separated by `\r\n`. If user input is placed into a header value without checking for these characters, an attacker can include their own `\r\n` inside their input, effectively telling the server "end this header here, and here's a brand new header/line I'm defining myself."

## Why it exists — the real-life cause

```python
# VULNERABLE — user input placed directly into a response header
@app.route('/set-language')
def set_language():
    lang = request.args.get('lang')
    response = make_response(redirect('/home'))
    response.headers['Set-Cookie'] = f'lang={lang}'  # lang is attacker-controlled
    return response
```
A normal request `?lang=en` sets a harmless cookie. But if an attacker submits `?lang=en%0d%0aSet-Cookie: session_id=attacker_controlled_value` (URL-encoded `\r\n`), the resulting raw HTTP response header section becomes:
```
Set-Cookie: lang=en
Set-Cookie: session_id=attacker_controlled_value
```
The attacker has injected an entirely new header the application never intended to send. Taken further, enough injected CRLF sequences can even inject a fake response body, enabling **HTTP Response Splitting** — tricking a caching proxy or the browser into treating the attacker's injected content as a completely separate, second response.

```python
# SECURE — reject or strip CRLF characters from any input placed into a header
@app.route('/set-language')
def set_language():
    lang = request.args.get('lang')
    if '\r' in lang or '\n' in lang or '%0d' in lang.lower() or '%0a' in lang.lower():
        abort(400)
    response = make_response(redirect('/home'))
    response.headers['Set-Cookie'] = f'lang={lang}'
    return response
```
Most modern web frameworks now do this validation automatically for built-in header-setting functions — but custom/manual header construction, or older framework versions, can still be vulnerable.

## How an attacker actually does it, step by step
1. Find any user input that ends up reflected into a response header — a redirect URL, a language/locale parameter, a custom header value.
2. Try injecting URL-encoded CRLF sequences (`%0d%0a`) followed by a new header name — e.g. a second `Set-Cookie` line, or a `Location` header for an open-redirect-style attack.
3. If the injected header appears in the raw response (visible via Burp Suite's raw response view, not the rendered browser output), CRLF injection is confirmed.
4. Escalate toward **HTTP Response Splitting** — injecting enough CRLF sequences to define a complete second, fake HTTP response, which can be used to poison a caching proxy so it serves the attacker's fake response to *other* users who request the same URL afterward.

## Technical Impact
- Injecting arbitrary headers (e.g. setting cookies for other users, or manipulating security headers)
- **Cache poisoning** — if a caching proxy sits in front of the application, a successful response-splitting attack can cause the poisoned/attacker-controlled response to be served to every subsequent visitor, not just the original attacker
- Session fixation via injected `Set-Cookie` headers (connects to file `03`)

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | Cache poisoning affecting a shared caching layer means a single successful injection can serve malicious content (or a phishing page, via injected headers) to every customer requesting that page afterward — turning a single-request vulnerability into a mass-impact one |
| **Regulatory / compliance** | Response splitting affecting multiple customers simultaneously is treated as a broad-impact incident rather than an isolated bug, escalating its severity in any formal assessment |
| **Reputational damage** | If the poisoned cache serves a phishing page or defaced content to many customers at once, this is a highly visible, public-facing incident |
| **Legal liability** | Similar reasoning to other well-documented injection classes — a known, well-understood vulnerability left unmitigated |
| **Operational cost** | Requires purging poisoned cache entries across potentially many edge/CDN nodes, in addition to fixing the underlying injection point |

**One-line interview answer:** *"Technically, CRLF injection lets an attacker insert new HTTP headers or even a fake response by including line-break characters in input that ends up in a response header. The business impact scales with caching — if a shared cache sits in front of the app, one successful injection can poison what gets served to every subsequent visitor, turning a single-request bug into a mass-impact incident."*

## Mitigation

1. **Reject or strip CRLF characters (`\r`, `\n`, and their URL-encoded forms) from any input destined for a response header** — the direct fix, shown above.
2. **Use framework-native header-setting APIs** rather than manually constructing raw header strings — most modern frameworks handle this validation automatically when using their built-in methods (e.g. `response.set_cookie()` rather than manually building the `Set-Cookie` string).
3. **Keep web servers, frameworks, and any front-end caching/proxy layers up to date** — this connects to file `18`, since older software versions are more likely to lack this protection built-in.

## Explaining it to a developer
*"This value is being placed directly into a response header without checking for line-break characters — which means someone could include their own line breaks in the input to inject an entirely new header the application never intended to send. Using the framework's built-in cookie/header-setting function instead of building the header string manually usually handles this automatically, but it's worth double-checking anywhere headers are constructed by hand."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Reject/strip CRLF characters in header input | Header injection at the source |
| Use framework-native header APIs | Automatic protection in most modern frameworks |
| Keep server/proxy software updated | Reduces exposure to known, older vulnerable implementations |
