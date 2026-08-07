# 02 - Enumeration (Explained Simply)

## Why this comes right after scanning
Scanning (file `01`) tells you a door is open — "port 445 is open." Enumeration is walking through that open door and asking specific questions: "who lives here, what's your name, what can I see from the hallway?" It's the difference between knowing a service exists and actually extracting useful information from it.

---

## The core idea, in one sentence
Enumeration is **politely asking a service to tell you about itself, and writing down everything it's willing to share** — usernames, shared folders, software versions, internal names — all without needing a password yet.

## Banner Grabbing — the simplest form of enumeration
Many services announce themselves the moment you connect, the same way a receptionist might say "Thank you for calling Acme Corp" before you've even asked a question. This greeting is called a **banner**.

```
nc 192.168.1.10 21
```
Connecting to an FTP port (21) with a basic tool like Netcat often gets you a response like:
```
220 ProFTPD 1.3.5 Server ready
```
That single line just told you the exact software (ProFTPD) and version (1.3.5) — enough to go look up known vulnerabilities for that specific version (connects to file `18` in the web series).

## SMB Enumeration — Windows file-sharing, and why it's such a common target
**SMB (Server Message Block)** is the protocol Windows uses for sharing files and printers over a network, typically on port 445. It's one of the most commonly enumerated services in real assessments because it tends to reveal a lot with very little effort.

**What you can pull from SMB without a password (depending on configuration):**
- A list of shared folders on the machine
- A list of valid usernames on the system
- The OS version and hostname
- Password policy details (minimum length, lockout threshold) — useful later for password-guessing attempts

```
enum4linux 192.168.1.10
```
This single tool automates pulling all of the above from an SMB service in one go — a classic first step against any Windows target.

## DNS Enumeration — mapping an organization's internal naming
DNS (Domain Name System, port 53) translates names like `mail.company.com` into IP addresses. Enumerating DNS means trying to discover *every* name a domain uses, not just the ones you already know about — subdomains, internal servers, mail servers.

**Zone Transfer** — the most severe DNS misconfiguration. Normally, DNS servers only share individual records when asked directly by name. A zone transfer is meant to let a *backup* DNS server copy the *entire* list of records at once — but if the primary server is misconfigured to allow this request from anyone, not just its authorized backup server, an attacker can request the whole list in one shot.

```
dig axfr @ns1.company.com company.com
```
If this succeeds against a misconfigured server, you instantly get a complete map of every subdomain, internal hostname, and mail server the organization has — reconnaissance that would otherwise take hours of guessing, handed over in one request.

## SNMP Enumeration — the overlooked protocol
**SNMP (Simple Network Management Protocol)**, port 161, is used to monitor and manage network devices (routers, switches, printers). It's frequently overlooked because it's not a "human-facing" service, but it's often left with weak default settings.

**The classic weakness:** SNMP uses a "community string" as a shared password — and it's extremely common to find it left at the default value `public` (read-only) or `private` (read-write), often years after deployment.

```
snmpwalk -c public -v1 192.168.1.10
```
If the default community string works, this can dump an enormous amount of information about the device — network configuration, running processes, sometimes even usernames — again, no real password-cracking required, just checking if the default was ever changed.

## Why enumeration is treated as its own step, not just "more scanning"
Scanning answers "what's open." Enumeration answers "what can I actually learn from what's open, before I try to break anything." A tester who skips straight from scanning to attacking is working blind — enumeration is what tells you *which* attack is even worth attempting (e.g. "this SMB share allows anonymous login" tells you exactly where to focus, instead of guessing).

## Quick-reference table

| Service | Port | What enumeration reveals |
|---|---|---|
| FTP | 21 | Software/version via banner |
| SMB | 445 | Shares, usernames, OS version, password policy |
| DNS | 53 | Subdomains, internal hostnames (via zone transfer if misconfigured) |
| SNMP | 161 | Device config, processes, sometimes usernames (via weak community strings) |

## Ethical note
Same rule as scanning (file `01`) — only enumerate systems you own or have explicit written authorization to test. Enumeration can feel "passive" since no passwords are being guessed, but it's still unauthorized access if done without permission.
