# Insecure Deserialization

## Why this comes right after Command Injection
Command Injection is direct and obvious — user input goes straight into a shell command. Insecure Deserialization is a subtler variant of the same underlying idea: user-controlled data gets converted back into a live object or executed logic, just through a less obvious mechanism — the process of "deserializing" data (turning a stored/transmitted format back into a usable in-memory object) rather than an explicit command string.

---

## What it is (in plain terms)
Applications often need to convert an object into a storable/transmittable format (serialization — e.g. turning a Python object into a byte stream) and then convert it back later (deserialization). Some serialization formats (Python's `pickle`, Java's native serialization, PHP's `unserialize`) don't just restore plain data — they can reconstruct arbitrary objects and, in some cases, trigger code execution as a side effect of reconstructing a maliciously crafted object. If an application deserializes data that an attacker controlled, it can be tricked into executing attacker-defined code during that reconstruction process, without any code injection in the traditional sense.

## Why it exists — the real-life cause

```python
# VULNERABLE — deserializes a session/cookie value using pickle
import pickle
import base64

def load_session(cookie_value):
    data = base64.b64decode(cookie_value)
    return pickle.loads(data)  # reconstructs whatever object the byte stream describes
```
Python's `pickle` format doesn't just store plain data — it can store instructions for *how to reconstruct an object*, including calling arbitrary functions during that reconstruction. If an attacker can control the cookie value, they can craft a malicious pickle payload that, when deserialized, executes a command of their choosing — the vulnerability isn't in what the code does with the resulting object, it's in the act of calling `pickle.loads()` on untrusted data at all.

```python
# SECURE — uses a safe, data-only format instead
import json

def load_session(cookie_value):
    return json.loads(cookie_value)  # JSON can only ever represent plain data — no executable object logic
```
JSON, unlike `pickle`, has no concept of "reconstruct this arbitrary object" — it can only ever represent plain data structures (strings, numbers, lists, dictionaries). There's no equivalent malicious payload possible, because the format itself doesn't support the feature being abused.

## How an attacker actually does it, step by step
1. Identify a place where the application deserializes data it received from the client — a cookie, a hidden form field, an API request body — especially if it's in a format like `pickle`, Java serialization, or PHP's `unserialize`.
2. Confirm the format — often a giveaway is a base64-encoded blob in a cookie, or a `application/x-java-serialized-object` content type.
3. Craft a malicious serialized payload using known "gadget chains" (pre-existing classes in common libraries whose normal behavior can be chained together to achieve code execution when reconstructed) — tools like `ysoserial` (for Java) automate building these payloads.
4. Submit the malicious serialized data in place of the legitimate value — if the application deserializes it, the crafted object's reconstruction triggers the attacker's code, resulting in remote code execution.

## Technical Impact
- **Remote code execution** — the most common and severe outcome, achieved without any traditional "injection" of visible malicious code — it's hidden inside what looks like an ordinary serialized data blob
- Denial of service — crafted objects designed to consume excessive resources during reconstruction

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | Same open-ended exposure as any other RCE-class vulnerability already covered — full server compromise, potential pivot to database or internal infrastructure |
| **Regulatory / compliance** | Critical-severity finding, same tier as command injection or file-upload RCE — but often flagged specifically as a design-level architecture issue (choice of serialization format), not just a coding mistake, which can prompt broader architectural review requirements from auditors |
| **Reputational damage** | Same as other RCE-class findings — full compromise scenarios tend to result in the most damaging public disclosures |
| **Legal liability** | Using an inherently unsafe serialization format (`pickle`, native Java serialization) for untrusted data is a well-documented, well-known risk in the security community — using it anyway is difficult to defend as reasonable practice |
| **Operational cost** | Same as other RCE-class incidents — full compromise response — plus, since the vulnerability is often architectural (the choice of format itself), remediation may require broader refactoring across every place that format is used, not just one endpoint |

**One-line interview answer:** *"Technically, insecure deserialization happens when untrusted data is converted back into a live object using a format that supports executing code as part of that reconstruction — meaning the attacker doesn't need to inject a traditional payload, just craft data that becomes malicious when the format processes it. For a bank, the business impact is the same as any other remote-code-execution finding — full server compromise — but it's often harder to catch in code review because it doesn't look like an obvious injection point."*

## Mitigation — layered, not just one fix

1. **Never deserialize untrusted data using formats capable of arbitrary object reconstruction** (the real fix) — use safe, data-only formats like JSON for anything that crosses a trust boundary (cookies, API requests, user-supplied data).
2. **If a native serialization format is genuinely required internally, never expose it to untrusted input** — keep it strictly for server-to-server communication within a trusted boundary, never accepting it directly from a client.
3. **Use signing/integrity checks** — if serialized data must be sent to the client and back (e.g. a signed session token), cryptographically sign it and verify the signature before deserializing, so tampered data is rejected before it's ever processed.
4. **Keep deserialization libraries updated** and monitor for known gadget-chain vulnerabilities in dependencies, since new exploitation techniques for existing libraries are discovered regularly.

## Explaining it to a developer
*"Right now, this session cookie is being deserialized using a format that can reconstruct arbitrary objects, not just plain data — which means someone could craft a malicious cookie value that runs code the moment it's loaded, without needing to find a traditional injection point anywhere else. The fix is to switch to a format like JSON that can only ever represent plain data, with no ability to trigger code execution during reconstruction — and if we ever do need the original format for internal use, it should never touch anything coming directly from a client."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Use safe, data-only formats (JSON) | Removes the vulnerability class entirely for untrusted input |
| Restrict unsafe formats to trusted, internal-only use | Prevents client-supplied data from reaching a vulnerable deserializer |
| Sign and verify serialized data | Tampered payloads rejected before deserialization occurs |
| Keep libraries updated | Reduces exposure to newly discovered gadget chains |
