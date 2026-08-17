# Module 09 - Unsafe Consumption of APIs

## Why this closes the sequence
Every module so far assumed your organization's own API is the thing being attacked. This final module flips the direction entirely: the risk that comes from **your own application trusting a third-party API it calls**, blindly. This is a genuinely different threat model — the vulnerability isn't in code you wrote, but in how much trust you extended to someone else's.

---

## What it actually is
Modern applications rarely stand alone — they call third-party APIs constantly (payment processors, shipping calculators, weather data, identity verification services). Unsafe Consumption is the risk that arises when a developer trusts data coming *back* from these third-party APIs with less scrutiny than they'd apply to data coming from an untrusted end user — treating "it came from our trusted payment provider" as a reason to skip validation that should apply regardless of the data's source.

---

## Why Developers Under-Scrutinize Third-Party API Responses (the real psychology behind the gap)

**The core mistaken assumption:** input validation is almost universally applied to data coming from *end users* (form fields, URL parameters) — that instinct is well-trained by now. But data coming back from a trusted third-party partner API often skips that same scrutiny, based on the reasoning "this is our own trusted vendor, not a random user submitting a form." **Why this reasoning is flawed:** a third-party API can be compromised, can have its own bugs returning malformed data, or — most realistically — the *connection* to it could be intercepted (connecting directly back to the MITM content in the network security series), meaning the data your application receives isn't guaranteed to be exactly what the legitimate third party actually sent, even if the API itself is entirely trustworthy.

---

## Real Example 1: Injection via Trusted Third-Party Data

```javascript
// VULNERABLE — trusts data from a third-party shipping API without validation
const shippingData = await fetch('https://third-party-shipping.com/api/calculate');
const data = await shippingData.json();
db.query(`INSERT INTO shipments (tracking_number) VALUES ('${data.trackingNumber}')`);
```
If the third-party API is ever compromised, has a bug, or the connection between your server and it is intercepted and tampered with, `data.trackingNumber` is no longer guaranteed to be a simple tracking number — it could contain a SQL injection payload (file `04`), and this code applies precisely zero of the input validation it would have applied to the exact same value if it had come from an end-user form field instead.

```javascript
// SECURE — the exact same validation discipline applied regardless of source
const shippingData = await fetch('https://third-party-shipping.com/api/calculate');
const data = await shippingData.json();
db.query('INSERT INTO shipments (tracking_number) VALUES (?)', [data.trackingNumber]);
```
**The actual lesson, stated plainly:** parameterized queries, output encoding, and every other validation principle covered throughout this entire repo apply based on **trust boundary**, not based on *who* or *what* is on the other side of that boundary. A third-party API response crosses exactly the same trust boundary a user's form submission does — both are "external input" from your application's own code's point of view, and should be treated identically.

---

## Real Example 2: Following Unvalidated Redirects from a Third-Party API

**The scenario:** your application calls a third-party API that, as part of its normal response, includes a URL your application is meant to redirect the user to (e.g. completing a payment flow, being sent to a third-party's hosted checkout page).
```javascript
// VULNERABLE — blindly redirects to whatever URL the third party's response contains
const paymentResponse = await fetch('https://payment-provider.com/api/create-session');
const data = await paymentResponse.json();
res.redirect(data.redirectUrl);  // trusts this URL completely
```
**Why this matters:** if the third-party API is compromised, or — again — the connection to it is intercepted and the response tampered with, `data.redirectUrl` could be swapped for an attacker-controlled URL, and your own application becomes the mechanism sending your own users to a phishing page — the exact Open Redirect risk from web file `16`, just triggered by trusting a third-party response instead of a client-supplied parameter.

---

## Real Example 3: Excessive Data Exposure — Passing Third-Party Responses Straight Through

**The scenario:** your API calls a third-party service and forwards that response directly back to your own end users, without filtering it first.
```javascript
// VULNERABLE — whatever the third party returns, the end user receives directly
app.get('/api/user-verification', async (req, res) => {
  const result = await fetch('https://identity-verification-service.com/api/check');
  const data = await result.json();
  res.json(data);  // may include internal fields the third party's API returns but your app never needed
});
```
**Why this connects directly back to Module 03's Excessive Data Exposure:** the exact same principle applies here in reverse — a third-party API's response might include internal debugging fields, rate-limit details, or other information genuinely never meant for your end users to see, and blindly passing it straight through exposes all of it, exactly the same as blindly returning your own database record.

---

## Why This Matters Specifically for Financial/Banking Integrations
Banking applications integrate with an especially large number of third parties by nature of the industry — payment processors, credit bureaus, fraud-detection services, identity verification providers. Each one of these integrations is a place where "trusted vendor" reasoning could quietly bypass validation discipline that would otherwise be applied rigorously everywhere else in the same application — making this category a realistic, systemic risk specifically for this domain, not a rare edge case.

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | An injection or open-redirect vulnerability reached through a trusted third-party integration produces the exact same downstream impact as the same vulnerability reached through user input — full account compromise or financial fraud, just via an unexpected path |
| **Regulatory / compliance** | Third-party integration security is explicitly a growing regulatory focus area (supply-chain/vendor risk) — an incident traced back to unvalidated trust in a partner API reflects on the organization's own due diligence, not just the third party's |
| **Reputational damage** | "A vulnerability in our trusted payment partner's response led to a redirect attack against our own users" is a genuinely confusing, hard-to-explain incident narrative for both the organization and affected customers |
| **Legal liability** | Contractual/vendor relationships don't transfer legal responsibility for how your own application handles data — your organization remains responsible for validating what it consumes, regardless of the source's own trustworthiness |
| **Operational cost** | Incident investigation is more complex when the root cause traces through a third party — coordinating with an external vendor's own security team adds real time and complexity beyond a purely internal incident |

**One-line interview answer:** *"Unsafe consumption of APIs is the reverse of every other vulnerability in this series — instead of your own API being attacked directly, it's about your application extending less scrutiny to data from a trusted third-party API than it would to a random user's input. The fix is recognizing that a trust boundary is a trust boundary regardless of who's on the other side — a compromised or intercepted third-party response is exactly as dangerous as malicious user input, and needs the exact same validation discipline applied to it."*

## Mitigation

1. **Apply identical input validation to third-party API responses as to any user-supplied input** — parameterized queries, output encoding, and allow-list validation, based on trust boundary, never based on assumed source trustworthiness.
2. **Validate redirect URLs from third-party responses against an explicit allow-list** — the exact same principle as web file `16`, applied to this specific source.
3. **Explicitly whitelist which fields from a third-party response actually get passed through to your own end users** — never a blind pass-through, the same discipline as Module 03's data exposure fix.
4. **Use TLS with proper certificate validation on every third-party API connection**, and don't disable certificate checking "temporarily" during development in a way that accidentally survives into production — this is exactly what makes response tampering via interception (Module 10 in the network series) practically possible in the first place.
5. **Maintain an inventory of third-party API dependencies** (connects to Module 08 — the same inventory discipline applies to what you consume, not just what you expose) so that a vulnerability disclosed in a specific third-party service can be quickly checked against your own actual usage.

## Quick-reference table

| Risk | Root Cause | Fix |
|---|---|---|
| Injection via third-party data | Trusted source assumed to mean "safe," validation skipped | Identical validation regardless of data source |
| Open redirect via third-party response | Blindly following a URL from a trusted API | Allow-list validation on redirect URLs, same as file `16` |
| Excessive exposure via pass-through | Forwarding entire third-party responses unfiltered | Explicit field whitelisting before returning to end users |
| Tampered responses via interception | No/weak TLS validation on the outbound connection | Proper certificate validation, always, in every environment |

---

## This closes the 10-module API security sequence (00-09)
Together: API fundamentals and the real differences between REST/SOAP/GraphQL/gRPC (`00`) → object-level authorization (`01`) → authentication specifically as it breaks differently for tokens/keys (`02`) → field-level exposure and mass assignment (`03`) → function-level authorization and resource consumption (`04`) → SSRF and injection through API-native delivery mechanisms (`05`) → business-logic abuse at automation scale (`06`) → API-specific misconfiguration (`07`) → the risk of not knowing your own API surface (`08`) → and finally, the risk of trusting someone else's (`09`). Every module built on named concepts from the web and network series rather than repeating them, exactly as intended from the start of this project.
