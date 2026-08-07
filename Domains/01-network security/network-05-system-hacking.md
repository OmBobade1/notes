# 05 - System Hacking (Explained Simply)

## Why this comes right after vulnerability assessment
Vulnerability assessment (`04`) told you *what's* weak. System hacking is what happens once someone actually gets a foothold — the sequence of steps an attacker takes from "I'm in, barely" to "I fully control this machine and can stay here."

---

## The core idea, in one sentence
System hacking is **the ladder an attacker climbs after getting through the front door** — starting with limited access, and working step by step toward full control, then making sure they can get back in later without repeating the whole process.

## The general ladder (this is the mental model to keep)
1. **Gaining Access** — getting some kind of foothold at all, even a low-privilege one (a regular user account, not admin)
2. **Privilege Escalation** — climbing from that low-privilege foothold to full administrative/root control
3. **Maintaining Access** — making sure you can get back in later, even if the original way in gets fixed
4. **Covering Tracks** — hiding evidence that any of this happened

## Password Attacks — the most common way in
**Brute Force** — trying every possible password combination, one by one, until something works. Slow, but guaranteed to eventually succeed against a weak/short password given enough time.

**Dictionary Attack** — instead of trying *every* combination, trying a curated list of likely passwords first (common passwords, leaked password lists from previous breaches). Much faster than pure brute force, because real people tend to reuse a relatively small pool of common passwords.

**Credential Stuffing** — using username/password pairs leaked from a *completely different* breach, betting that people reuse the same password across multiple sites. This connects directly to the authentication file in the web security series — it's exactly why rate limiting on login pages matters so much.

**Password Spraying** — instead of trying many passwords against *one* account (which triggers lockouts quickly), trying one or two common passwords against *many* accounts. This deliberately avoids triggering account lockout policies, since no single account gets enough failed attempts to lock.

## Privilege Escalation — climbing from "some access" to "full access"
Once inside with limited privileges, the goal is finding a way to become admin/root. Two broad categories:

**Vertical Privilege Escalation** — going from a low-privilege account to a higher-privilege one on the *same* machine (regular user → administrator). Often achieved by exploiting a misconfigured service running with high privileges, or exploiting a known OS vulnerability that specifically grants elevated access.

**Horizontal Privilege Escalation** — not gaining *more* power, but gaining access to a *different* account at the same privilege level (e.g. breaking into another regular user's account to access their specific files) — this is conceptually the same idea as IDOR in the web security series, just happening at the operating system level instead of a web application.

## Maintaining Access — making sure the door stays open
Once an attacker has the access they want, the last thing they want is to lose it if the original vulnerability gets patched or the system reboots. Common approaches:

**Backdoors** — planting a hidden way back in, independent of the original vulnerability — a secret account, a scheduled task that reconnects to the attacker periodically, or a modified system file that grants silent access.

**Persistence mechanisms** — configuring something to automatically re-establish access every time the system restarts (a startup script, a scheduled task, a modified service) — without this, a simple reboot could cut the attacker off entirely.

## Covering Tracks — hiding that any of this happened
**Log clearing/manipulation** — deleting or altering system logs that would otherwise show the login times, commands run, or files accessed. This directly connects to the logging & monitoring file in the web security series — it's exactly why tamper-resistant, centralized logging matters: if logs only exist on the compromised machine itself, an attacker with admin access can simply delete the evidence.

**Hiding files/processes** — using techniques to make malicious files or running processes not show up in normal directory listings or task managers, so a casual look at the system doesn't reveal anything unusual.

## Why this whole ladder matters for defense, not just offense
Every stage of this ladder has a corresponding defensive control, which is really the point of learning the attacker's process:
- Strong, unique passwords + rate limiting + MFA → blocks the password attack stage
- Least-privilege configuration + timely patching → blocks privilege escalation
- Monitoring for new scheduled tasks/services/accounts → catches persistence attempts
- Centralized, tamper-resistant logging → defeats track-covering, since the attacker can't reach the real log copy

## Quick-reference table

| Stage | Goal | Example technique |
|---|---|---|
| Gaining Access | Any initial foothold | Exploiting a known vulnerability, phishing |
| Privilege Escalation | Low privilege → full control | Exploiting a misconfigured service |
| Maintaining Access | Survive reboots/patches | Backdoor accounts, scheduled tasks |
| Covering Tracks | Hide evidence | Log deletion, hiding files/processes |
