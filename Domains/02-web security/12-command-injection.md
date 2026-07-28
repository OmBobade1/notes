# OS Command Injection

## Why this comes right after XXE
This closes out the "user input reaches something dangerous" family that started with SQL Injection. SQLi put input into a database query, XSS into HTML/JavaScript, LFI/XXE into a file path or parser — Command Injection puts input directly into a command executed by the underlying operating system itself. It's arguably the most severe of the entire group, since it hands the attacker a shell on the server directly, no further chaining required.

---

## What it is (in plain terms)
Some applications need to run system-level commands — pinging an address, converting a file format, checking a domain's DNS record. Command Injection happens when user input is inserted directly into that system command without being properly separated from it, letting an attacker append their own additional commands that the operating system will execute right alongside the intended one.

## Why it exists — the real-life cause

```python
# VULNERABLE — user input concatenated directly into a shell command
import os

def ping_host(host):
    os.system(f"ping -c 4 {host}")
```
A normal request might supply `host=google.com`, running `ping -c 4 google.com` as intended. But if an attacker supplies `host=google.com; rm -rf /important-data` or `host=google.com && cat /etc/passwd`, the shell interprets `;` and `&&` as command separators — it runs the intended ping command, *and then* runs the attacker's completely separate, arbitrary command right after it, because the shell can't tell where the "intended" command ends and the attacker's addition begins.

```python
# SECURE — uses a subprocess call with arguments passed separately, no shell interpretation
import subprocess

def ping_host(host):
    subprocess.run(["ping", "-c", "4", host], shell=False)
```
Here, `host` is passed as a distinct argument to the `ping` program directly — never interpreted by a shell at all, so special characters like `;` or `&&` are just treated as a literal (and likely invalid) part of the hostname, not as command separators. This is the exact same underlying principle as parameterized queries for SQLi: keep the "instruction" and the "data" structurally separate, so the data can never be misread as a new instruction.

## Payload alternatives (shell metacharacters and what they do)

| Payload addition | What it does |
|---|---|
| `; <command>` | Runs `<command>` as a completely separate command after the intended one (Linux/Unix) |
| `&& <command>` | Runs `<command>` only if the first command succeeds |
| `| <command>` | Pipes the output of the intended command into `<command>` |
| `` `<command>` `` or `$(<command>)` | Command substitution — runs `<command>` and inlines its output directly into the original command |

## How an attacker actually does it, step by step
1. Find any feature that appears to interact with the underlying system — network diagnostic tools (ping, traceroute), file conversion utilities, image processing that shells out to external tools (ImageMagick, ffmpeg).
2. Test with a simple, harmless-looking injection: `host=google.com; whoami` — if the response includes the current user the server process is running as, command injection is confirmed.
3. Escalate to something with real impact: reading sensitive files (`cat /etc/passwd`), establishing a reverse shell back to an attacker-controlled machine for persistent, interactive access, or directly interacting with the database/filesystem.

## Technical Impact
- **Full remote code execution** — arguably the most direct and severe outcome of any vulnerability in this entire series, since there's no intermediate step needed; the attacker's command runs immediately
- Complete server compromise, with whatever privileges the application's process happens to run as (which is itself a strong argument for least-privilege service accounts)

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | Identical open-ended exposure to the file-upload RCE and RFI scenarios already covered — from server compromise, an attacker can pivot to the database, internal network, or cloud infrastructure |
| **Regulatory / compliance** | Same severity tier as RCE via file upload or RFI — this is a critical, emergency-response-level finding in any serious security assessment, not a routine bug |
| **Reputational damage** | Full server compromise via command injection is exactly the kind of incident that ends up in public breach disclosures and security news coverage, given the scale of what's typically accessible from that point |
| **Legal liability** | A directly exploitable command injection reaching the shell is about as clear-cut a negligence case as this entire list gets — there's no ambiguity about whether basic secure coding practice was followed |
| **Operational cost** | Same as any confirmed RCE — assume full compromise, rebuild from known-clean state, rotate every credential the compromised process had access to, extensive forensic investigation |

**One-line interview answer:** *"Technically, command injection lets an attacker append their own operating-system commands to ones the application already runs — resulting in direct remote code execution, no chaining or further exploitation required. For a bank, this is one of the most severe possible findings across the entire assessment, because it's an immediate, complete server compromise rather than a partial or conditional risk."*

## Mitigation — layered, not just one fix

1. **Avoid shell execution entirely where possible (the real fix)** — use language-native libraries instead of shelling out to system commands (e.g. a DNS resolution library instead of shelling out to `nslookup`).
2. **When shell commands are genuinely necessary, pass arguments as a list, never as a concatenated string** — as shown above, this avoids shell interpretation of the input entirely.
3. **Strict input validation/allow-listing** — if the input should only ever be a hostname or IP address, validate it against that exact expected format before use, as a defense-in-depth backup layer.
4. **Least-privilege service accounts** — the application process should run with the minimum OS-level permissions it actually needs, limiting the damage even if command injection does succeed.
5. **Sandboxing/containerization** — running the vulnerable component in an isolated container limits what a successful command injection can actually reach on the broader system.

## Explaining it to a developer
*"This network diagnostic feature builds a shell command by directly joining the user's input into a string — which means the shell interprets special characters like semicolons the same way it would if someone typed a second command by hand. The fix is to stop building a shell command string at all: pass the hostname as a separate argument to the ping program directly, without going through a shell interpreter, so there's no possibility of the input being read as a new instruction rather than just a value."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Avoid shell execution / use native libraries | Removes the vulnerability class entirely where feasible |
| Pass arguments as a list, not a concatenated string | Shell metacharacters treated as literal data, not commands |
| Strict input validation | Backup layer catching obviously malformed input |
| Least-privilege service account | Limits damage even if injection succeeds |
| Sandboxing/containerization | Contains the blast radius of a successful exploit |
