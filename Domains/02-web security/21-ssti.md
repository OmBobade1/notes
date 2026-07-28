# Server-Side Template Injection (SSTI)

## Why this comes next
Same injection family as everything since file `04` — user input reaching something that interprets it as code instead of pure data. This time it's a templating engine (Jinja2, Twig, FreeMarker, etc.) — commonly used to generate dynamic HTML — that gets tricked into evaluating attacker-supplied template syntax instead of just displaying it as text.

---

## What it is (in plain terms)
Templating engines let developers embed dynamic values into HTML using special syntax, e.g. `{{ username }}` in Jinja2, which gets replaced with the actual username when the page renders. SSTI happens when user input is inserted directly into the *template string itself* (not just as a value being substituted into an existing template) — so if a user's input happens to contain template syntax like `{{ 7*7 }}`, the engine doesn't just display that text, it actually evaluates it as a template expression.

## Why it exists — the real-life cause

```python
# VULNERABLE — user input is inserted directly into the template STRING before rendering
from jinja2 import Template

@app.route('/greet')
def greet():
    name = request.args.get('name')
    template = Template(f"Hello, {name}!")  # name becomes part of the template itself
    return template.render()
```
A normal request `?name=Om` renders "Hello, Om!" as expected. But if an attacker submits `?name={{7*7}}`, the template becomes `Hello, {{7*7}}!` — and since `{{7*7}}` is now genuinely part of the template *syntax*, not just a value, Jinja2 evaluates it and returns "Hello, 49!" — proving the engine is executing the input as code, not displaying it as text. From there, Jinja2's full expression language can be abused to reach Python's underlying object model and, in many cases, achieve full remote code execution.

```python
# SECURE — the template structure is fixed; user input is only ever passed as a VALUE to substitute in
from jinja2 import Template

@app.route('/greet')
def greet():
    name = request.args.get('name')
    template = Template("Hello, {{ name }}!")  # template string itself is fixed, never built from user input
    return template.render(name=name)  # name is just a value substituted safely into the existing template
```
This is the exact same underlying principle as parameterized SQL queries and safe output encoding for XSS: keep the fixed structure (the template) and the untrusted data (the user's input) strictly separate. Here, `name` is only ever treated as a value to display — even if it contains `{{7*7}}`, it's rendered as the literal text `{{7*7}}`, not evaluated.

## The proof-of-concept escalation path
1. **Detect:** submit `{{7*7}}` — if the response shows `49` instead of the literal text, SSTI is confirmed.
2. **Identify the engine:** different templating engines have different syntax (`{{ }}` for Jinja2, `${ }` for FreeMarker, `{{ }}` for Twig) — the exact payload that triggers evaluation helps fingerprint which one is in use.
3. **Escalate:** using the confirmed engine's expression language to walk Python's object hierarchy (a well-documented technique for Jinja2/Python specifically) to reach the `os` module or equivalent, ultimately executing arbitrary system commands — turning a "template evaluated my math" proof-of-concept into full remote code execution.

## Technical Impact
- **Remote Code Execution** — SSTI frequently escalates all the way to full RCE, placing it in the same severity tier as Command Injection (file `12`) and unrestricted file upload (file `08`)

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | Same open-ended exposure as any other RCE-class finding in this series — full server compromise, potential pivot into database or internal infrastructure |
| **Regulatory / compliance** | Critical-severity finding, same tier as command injection or insecure deserialization |
| **Reputational damage** | Full server compromise scenarios tend to produce the most damaging public disclosures, same as other RCE-class findings |
| **Legal liability** | Same reasoning as other RCE-class vulnerabilities — a clear, well-documented vulnerability class with no ambiguity about whether secure coding practice was followed |
| **Operational cost** | Full-compromise incident response — same tier as command injection or file-upload RCE |

**One-line interview answer:** *"Technically, SSTI happens when user input becomes part of the template's own syntax rather than just a value substituted into it — meaning a template engine can end up evaluating attacker-supplied code, which frequently escalates to full remote code execution. For a bank, this sits in the same severity tier as command injection — it's not a partial risk, it's typically a complete server compromise."*

## Mitigation

1. **Never build a template string dynamically from user input (the real fix)** — the template structure must always be a fixed string defined by the developer; user input should only ever be passed as a *value* for substitution, exactly as shown in the secure example.
2. **Use a sandboxed/logic-less templating mode where available** — some engines offer a restricted execution environment (e.g. Jinja2's sandboxed environment) that blocks access to dangerous underlying object methods, as defense-in-depth even if the primary mistake is somehow still made.
3. **Input validation as a backup layer** — reject input containing template-syntax characters (`{{`, `${`, etc.) in fields never expected to contain them.

## Explaining it to a developer
*"Right now, user input is being inserted directly into the template string itself before it's rendered — which means if someone's input happens to look like template syntax, the engine treats it as real template code instead of plain text. The fix is to keep the template fixed and only ever pass user input in as a value to be substituted, the same way we'd use a parameterized query instead of building SQL by hand."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Fixed template, input as substitution value only | Removes the vulnerability at the design level |
| Sandboxed template execution mode | Limits damage even if the primary mistake occurs |
| Input validation for template-syntax characters | Backup layer catching obvious attempts |
