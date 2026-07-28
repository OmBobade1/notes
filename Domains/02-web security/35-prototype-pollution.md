# Prototype Pollution

## Why this comes next
A JavaScript-specific vulnerability (relevant to both Node.js backends and client-side JavaScript) that doesn't fit the "user input reaches a query/command/parser" pattern of earlier files — instead, it exploits a quirk of how JavaScript objects work internally, letting an attacker inject properties that affect *every* object in the application, not just one.

---

## What it is (in plain terms)
In JavaScript, every object inherits properties from a shared "prototype" object (`Object.prototype`) by default — this is how built-in behavior gets shared across all objects without needing to redefine it each time. Prototype Pollution happens when an attacker can inject a property onto this shared prototype (typically via a special key like `__proto__`, `constructor`, or `prototype`) through user input that gets merged into an object without proper filtering — and because *every* object in the application inherits from this same shared prototype, the injected property now silently appears on objects throughout the entire application, not just the one the input was originally processed into.

## Why it exists — the real-life cause

```javascript
// VULNERABLE — a naive "deep merge" utility function, commonly found in many libraries
function merge(target, source) {
  for (let key in source) {
    if (typeof source[key] === 'object') {
      target[key] = merge(target[key] || {}, source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}

// Used to merge user-supplied settings into a defaults object
const userSettings = JSON.parse(request.body.settings);  // attacker controls this JSON
const config = merge({}, userSettings);
```
If an attacker submits JSON like `{"__proto__": {"isAdmin": true}}` as `settings`, the merge function — which naively iterates over every key including `__proto__` — walks into `Object.prototype` itself and sets `isAdmin: true` directly on it. From this point forward, **every plain JavaScript object in the entire running application** — not just this one config object — will report `isAdmin === true` when that property is checked, because every object implicitly inherits from the now-polluted `Object.prototype`, unless it explicitly has its own `isAdmin` property overriding it.

```javascript
// SECURE — explicitly reject dangerous keys before merging
function merge(target, source) {
  const DANGEROUS_KEYS = ['__proto__', 'constructor', 'prototype'];
  for (let key in source) {
    if (DANGEROUS_KEYS.includes(key)) continue;  // skip these keys entirely
    if (typeof source[key] === 'object') {
      target[key] = merge(target[key] || {}, source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}
```
This explicitly skips the dangerous key names that would otherwise let the merge reach into the shared prototype — a targeted fix for this exact pattern. A more robust alternative is to use `Object.create(null)` for objects that don't need to inherit from `Object.prototype` at all, or to use the built-in `Map` type instead of a plain object for anything holding user-controlled keys, since `Map` has no equivalent prototype-pollution risk.

## How an attacker actually does it, step by step
1. Find an endpoint that merges user-supplied JSON into an existing object — common in configuration endpoints, "update profile" features, or any deep-merge/extend utility applied to request data.
2. Submit a payload containing `__proto__` (or `constructor.prototype`) as a key, with a property designed to have some effect elsewhere in the application — e.g. `isAdmin: true`, or a property that some other part of the code checks to decide behavior.
3. Trigger whatever downstream code checks that property — if a completely unrelated part of the application (say, an access-control check) reads `user.isAdmin` and JavaScript's prototype chain now resolves that to `true` for every object that doesn't explicitly override it, this can result in privilege escalation, achieved with zero direct connection between the injection point and the ultimately-affected code path.
4. In more severe cases, depending on how a polluted property is later used (e.g. if it ends up controlling a file path, or gets passed to something that executes code), this can escalate to Remote Code Execution.

## Technical Impact
- **Denial of Service** — polluting a property that breaks assumptions elsewhere in the application, causing crashes or errors application-wide
- **Privilege escalation / logic bypass** — polluting a property like `isAdmin` that gets checked in a completely unrelated part of the codebase, since every plain object inherits it
- **Remote Code Execution** — in specific, more severe cases, depending on exactly how a polluted property is subsequently used elsewhere in the application

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | Depends on what the polluted property ultimately affects — ranging from a service disruption to, in the worst case, privilege escalation enabling unauthorized financial actions, similar in outcome to other privilege-escalation-class vulnerabilities already covered |
| **Regulatory / compliance** | A vulnerability that can affect the *entire application's behavior* from a single injection point (rather than being contained to one feature) is treated seriously precisely because of that unusually broad blast radius |
| **Reputational damage** | Depends heavily on what's ultimately affected — could range from an unnoticed internal glitch to a serious, publicly-disclosed privilege escalation |
| **Legal liability** | This is a well-documented JavaScript-specific vulnerability class with well-known mitigations (key filtering, `Object.create(null)`, using `Map`) — a codebase without any of these protections in a merge/extend utility handling user input reflects a gap in secure coding awareness specific to the JavaScript ecosystem |
| **Operational cost** | Investigation is often unusually difficult precisely *because* the injection point and the ultimately-affected behavior can be in completely unrelated parts of the codebase — tracing "why is this unrelated feature suddenly behaving differently" back to a prototype pollution root cause requires specific awareness of this vulnerability class |

**One-line interview answer:** *"Technically, prototype pollution exploits how JavaScript objects share a common prototype — if user input reaches an unguarded merge function using a key like `__proto__`, the attacker can inject a property that then silently appears on every object across the entire application, not just the one being processed. The business impact varies with what gets polluted, but the distinguishing danger is the blast radius — a single injection point can affect completely unrelated parts of the codebase, which also makes it unusually hard to trace back to its root cause during investigation."*

## Mitigation

1. **Explicitly reject dangerous keys (`__proto__`, `constructor`, `prototype`) in any function that merges user-controlled data into an object (the real fix for existing merge logic)**, as shown above.
2. **Use `Object.create(null)` for objects intended to hold arbitrary user-controlled keys** — this creates an object with no prototype at all, so there's nothing to pollute even if a dangerous key gets through.
3. **Prefer `Map` over plain objects for storing user-controlled key-value data** — `Map` doesn't have this prototype-inheritance behavior at all, sidestepping the vulnerability class entirely by using a different, purpose-built data structure.
4. **Keep dependencies updated** (file `18`) — many popular utility libraries (including some very widely-used ones) have had disclosed prototype pollution vulnerabilities in their own merge/clone/extend functions over the years; using an outdated version of such a library can introduce this risk even without writing a vulnerable merge function yourself.
5. **Use `Object.freeze(Object.prototype)`** as a defense-in-depth measure in Node.js applications — this makes the shared prototype itself immutable, causing any pollution attempt to silently fail (or throw, in strict mode) rather than succeed.

## Explaining it to a developer
*"This merge function loops over every key in the user's input without checking what those keys actually are — including special keys like `__proto__` that don't just set a normal property, they reach into the shared prototype that every JavaScript object inherits from. That means an attacker can inject a property that silently appears on objects everywhere in our application, not just the one this function is processing — which is what makes this different from a typical injection bug: the effect can show up somewhere completely unrelated to where the input was actually accepted. The fix is to explicitly skip those dangerous key names in this function, or better, use a data structure like `Map` that doesn't have this shared-prototype behavior at all."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Reject `__proto__`/`constructor`/`prototype` keys in merge logic | Direct fix for the specific injection pattern |
| `Object.create(null)` for user-controlled-key objects | Nothing to pollute even if a dangerous key gets through |
| Use `Map` instead of plain objects | Sidesteps the vulnerability class entirely |
| Keep dependencies updated | Reduces exposure to known-vulnerable utility libraries |
| `Object.freeze(Object.prototype)` | Defense-in-depth — pollution attempts fail even if the above are missed |
