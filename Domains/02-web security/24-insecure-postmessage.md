# Insecure `postMessage` Handling

## Why this comes next
Everything since file `05` (XSS) has largely been about the server. This is a purely client-side, JavaScript-level vulnerability — `postMessage` is the browser's built-in mechanism for two different windows/iframes (even across different domains) to communicate safely. "Safely" only holds if the code on both ends actually checks who it's talking to — which is exactly where this goes wrong.

---

## What it is (in plain terms)
`postMessage` lets one browser window/tab/iframe send a message to another, even if they're on completely different domains — this is intentional and useful (e.g. a payment widget embedded via iframe talking to the parent page). The vulnerability shows up in two independent places: the **receiving** end not checking who sent the message, and the **sending** end not checking who it's sending to.

## Why it exists — the real-life cause

### Problem 1: Receiver doesn't check the sender's origin

```javascript
// VULNERABLE — accepts and acts on a message from ANY origin
window.addEventListener('message', function(event) {
  document.getElementById('balance').innerHTML = event.data.balance;  // trusts ANY sender completely
});
```
This listens for messages and blindly trusts whatever arrives — including from a completely different, malicious website that the victim happens to have open in another tab, or an attacker-controlled iframe embedded elsewhere on a page that includes this script. If `event.data` contains HTML/script content instead of a plain balance number, this is also a direct path to DOM-based XSS (connecting back to file `05`), since it's inserted via `innerHTML` with no sanitization.

```javascript
// SECURE — explicitly verifies the sender's origin before processing the message
window.addEventListener('message', function(event) {
  if (event.origin !== 'https://trusted-payment-widget.com') {
    return;  // reject messages from any other origin
  }
  document.getElementById('balance').textContent = event.data.balance;  // textContent, not innerHTML — avoids XSS too
});
```
Here, the code explicitly checks `event.origin` against an exact, known trusted domain before doing anything with the message content — and uses `textContent` instead of `innerHTML` as a second layer, so even a message that somehow got through wouldn't execute as HTML/script.

### Problem 2: Sender doesn't restrict the target origin

```javascript
// VULNERABLE — sends sensitive data to ANY window, using the wildcard target
parentWindow.postMessage({sessionToken: token}, '*');  // '*' means "send to whoever is listening, regardless of origin"
```
Using `'*'` as the target origin means the message goes out regardless of what's actually loaded in that window at the time — if an attacker can get their own page loaded into that window reference (e.g. through a race condition, a redirect, or a compromised iframe elsewhere), the sensitive data goes straight to them instead of the intended recipient.

```javascript
// SECURE — explicitly targets the exact expected origin
parentWindow.postMessage({sessionToken: token}, 'https://bank.com');  // only delivered if the actual origin matches exactly
```

## How an attacker actually does it, step by step
1. Identify a page using `postMessage` — visible in the page's JavaScript source (not hidden from a determined attacker, since all client-side code is inspectable).
2. Check whether the receiving listener validates `event.origin` — if it doesn't, craft a malicious page that opens the target as a popup or embeds it, then sends a forged message containing malicious data (an XSS payload, or data designed to manipulate what the victim sees, like a fake balance).
3. Separately, check whether any `postMessage` calls use `'*'` as the target — if so, look for a way to get an attacker-controlled page into that window reference, so sensitive data being sent gets intercepted instead of reaching the intended destination.

## Technical Impact
- **DOM-based XSS** if received message content is inserted into the page unsafely (connects directly to file `05`)
- **Sensitive data leakage** if a sender uses the wildcard `'*'` target and an attacker can position their page to receive it instead
- **Data/UI manipulation** — e.g. a forged message showing a fake balance or fake confirmation on a banking dashboard, exploiting the victim's trust in the page's own displayed content

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | If a payment widget or similar embedded iframe uses insecure postMessage, sensitive data (tokens, transaction details) could be intercepted, or a forged message could manipulate what a customer sees during a transaction flow |
| **Regulatory / compliance** | Client-side vulnerabilities like this are frequently under-tested compared to server-side flaws, representing a genuine gap in typical security assessment coverage — its presence often signals broader client-side security testing hasn't been thorough |
| **Reputational damage** | A forged message showing manipulated account information (e.g. a fake "transfer successful" confirmation) could be used as part of a broader social-engineering/fraud scheme against a customer, damaging trust in the platform's displayed information generally |
| **Legal liability** | Similar to XSS — a documented, well-known JavaScript security pattern (origin validation) that was skipped |
| **Operational cost** | Requires auditing every use of `postMessage` across the entire client-side codebase, since this is easy to miss in a purely server-side-focused security review |

**One-line interview answer:** *"Technically, insecure postMessage handling means a page either accepts messages from any origin without checking, or sends sensitive data to any origin using a wildcard target — either way breaking the trust boundary the mechanism is supposed to enforce. For a bank, if this touches an embedded payment widget or similar sensitive component, it can lead to intercepted data or a forged message manipulating what a customer sees — and it's a class of bug that's easy to miss since it lives entirely in client-side JavaScript, not server logs."*

## Mitigation

1. **Always validate `event.origin` on the receiving end** against an exact, known trusted domain — never process a message without this check.
2. **Always specify an exact target origin when sending**, never `'*'`, unless the message genuinely contains no sensitive data and the destination is guaranteed not to matter.
3. **Use `textContent` instead of `innerHTML`** when inserting any received message data into the page, as a backup layer against DOM-based XSS even if origin validation is somehow bypassed.
4. **Validate the structure/type of `event.data`** before using it — don't assume it matches the expected shape just because the origin check passed.

## Explaining it to a developer
*"This postMessage listener processes whatever data arrives without checking where it came from — which means any other page the user has open, or any iframe that can reach this window, can send fake data that gets treated as if it came from our trusted widget. The fix is a one-line check: compare `event.origin` against the exact domain we expect, and ignore anything that doesn't match. The same applies in reverse — when we send a message, we should target the exact domain we intend, not use the wildcard, so the data can't be intercepted by an unexpected recipient."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Validate `event.origin` on receive | Accepting forged messages from untrusted senders |
| Specify exact target origin on send (never `'*'`) | Sensitive data being intercepted by an unintended recipient |
| Use `textContent`, not `innerHTML` | DOM-based XSS via message content |
| Validate message data structure/type | Unexpected data shapes causing logic errors or further exploitation |
