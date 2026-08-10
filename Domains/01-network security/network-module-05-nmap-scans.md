# Module 5 - Nmap: Scan Types & Mechanics

## Why this comes right after Netcat
Netcat showed the raw building blocks (open a connection, send bytes). Nmap is purpose-built specifically for scanning at scale, and every scan type it offers is really a different manipulation of the same TCP/UDP handshake mechanics you just saw manually with Netcat — this module explains exactly what's happening at the packet level for each one, not just "which flag to type."

---

## TCP Connect Scan [-sT] — the "full, honest" scan

**What it actually does at the packet level:** Completes the entire three-way TCP handshake, exactly like a real application would.
```
SYN ------------------->
<------------------- SYN/ACK
ACK ------------------->
```
```
nmap -sT <target-ip>
```

**What it's for, attacker's perspective:** This is the only scan type that doesn't require root/administrator privileges, since it uses the operating system's normal, standard connection functions rather than crafting raw packets by hand — useful when running from a limited-privilege account.

**The real downside:** because it's a genuine, complete connection, it gets logged by the target the same way any real application connection would — this is the noisiest, most easily detected scan type of everything covered in this module.

---

## SYN Stealth / Half-Open Scan [-sS] — the actual default, and why

**What it actually does at the packet level:**
```
SYN ------------------->
<------------------- SYN/ACK   (port is open)
RST ------------------->        (tear down BEFORE completing the handshake)
```
```
nmap -sS <target-ip>
```

**Why this is Nmap's default when run with root/admin privileges:** By sending an RST instead of the final ACK, the connection is never actually completed — many older logging systems specifically only recorded *completed* connections, so this scan flew under the radar of that class of detection (modern logging/IDS has largely caught up, but the technique's name and reasoning stuck).

**The direct trade-off vs `-sT`:** requires raw packet crafting privileges (root/admin), but is faster and historically stealthier — this is why it's the default whenever you have the privileges to use it.

---

## FIN, Null, and Xmas Scans — exploiting how closed ports are supposed to behave

**The shared underlying logic for all three:** Per the TCP specification, a **closed** port receiving an unexpected packet is supposed to respond with an RST. An **open** port, receiving a packet that isn't a proper SYN, is supposed to simply **drop it silently** — no response at all. These three scans deliberately send "improper" packets specifically to exploit that behavioral difference.

**FIN Scan [-sF]:** sends a packet with only the FIN flag set (normally used to gracefully *close* an existing connection, never to open one).
```
nmap -sF <target-ip>
```

**Null Scan [-sN]:** sends a packet with **no flags set at all** — as invalid and unusual as it sounds.
```
nmap -sN <target-ip>
```

**Xmas Scan [-sX]:** sends a packet with the FIN, PSH, and URG flags all set simultaneously — the name comes from the packet "lighting up like a Christmas tree" with unusual flags.
```
nmap -sX <target-ip>
```

**Interpreting the results (this is the same logic across all three):**
- **RST received** → port is closed
- **No response at all** → port is either open, or filtered by a firewall (same ambiguity discussed in the earlier firewall-visibility explanation)

**What these are actually for, attacker's perspective:** Specifically useful against **older or more permissive firewalls/IDS systems** that were only configured to watch for "normal-looking" scan patterns like `-sS` or `-sT` — an unusual, malformed-looking packet can sometimes slip past detection rules that were only written with standard scans in mind. **Important limitation:** modern Windows systems don't follow this exact RFC behavior reliably, making these three scan types significantly less reliable against Windows targets specifically — worth knowing before relying on them.

---

## Ping Scan [-sP / -sn] — host discovery only, no port scanning at all

**What it actually does:** Sends ICMP Echo Requests (or, depending on configuration, TCP/ARP probes) purely to determine **which hosts are alive**, without checking any ports whatsoever.
```
nmap -sn 192.168.1.0/24
```

**What it's for, attacker's perspective:** The fast, low-noise first step before committing to a full port scan against every address in a range — no point spending time port-scanning 254 addresses if only 12 of them are actually live.

---

## UDP Scan [-sU] — slower, less reliable, but necessary

**Why UDP scanning is fundamentally different and harder:** UDP (covered in the earlier basics content) has no handshake at all — there's no SYN/ACK to rely on. Nmap sends a UDP packet and waits:
- **ICMP "port unreachable" response** → port is closed
- **Actual UDP response data** → port is open
- **No response at all** → open OR filtered (genuinely ambiguous, and this is the *normal* case for UDP, not an edge case)

```
nmap -sU <target-ip>
```

**Why it's so much slower in practice:** many operating systems deliberately rate-limit how many ICMP "unreachable" responses they'll send per second (as a basic anti-flood protection), which directly throttles how fast a UDP scan can realistically run — this isn't a flaw in Nmap, it's an inherent property of the protocol and how systems defensively handle it.

**Why you still have to run it despite the slowness:** critical services run on UDP — DNS (53), SNMP (161), DHCP — a TCP-only scan (`-sT`/`-sS`) would completely miss all of them, leaving a real gap in your assessment.

---

## ACK Scan [-sA] — mapping firewalls, not finding open ports

Already covered in depth in the earlier firewall-visibility explanation — worth restating concisely here since it belongs in this module: **this scan can never identify open ports at all**, by design. It only distinguishes stateful (filtered) firewalls from stateless (unfiltered) ones, based on whether your out-of-context ACK packet gets an RST back or silence.
```
nmap -sA <target-ip>
```

---

## Idle Scan [-sI] — the genuinely clever one

**What it actually does, step by step (the "zombie" technique):**
1. Nmap sends a probe to a third, uninvolved "zombie" host to learn its current **IPID** (a sequence number that increments predictably with each packet the zombie sends).
2. Nmap sends a SYN packet to the real target, but **spoofs the source address to look like it came from the zombie**, not from you.
3. If the target's port is open, it replies SYN/ACK — but sends that reply to the zombie (since that's the spoofed source), not to you. The zombie, having never actually initiated this connection, responds with an RST — which increments the zombie's IPID by one.
4. Nmap probes the zombie's IPID again. If it went up by one step from the original reading, that confirms the target's port is open — all without the target ever seeing *your* real IP address anywhere in the exchange.

```
nmap -sI <zombie-ip>:<zombie-port> <target-ip>
```

**What it's for, attacker's perspective:** This is about **attribution, not stealth from detection** — the target's logs will show the *zombie's* IP address as the apparent scanner, not yours, since your actual address never appears anywhere in the packets the target receives. Genuinely useful when you specifically need to avoid your own IP appearing anywhere in target-side logs.

---

## Version Detection [-sV] and OS Fingerprinting [-O]

**Version Detection:** After finding an open port via any scan type above, Nmap sends a series of follow-up probes specifically designed to elicit responses that reveal the exact software and version running — connects directly to the vulnerability assessment module, since exact version numbers are what you match against CVE databases.
```
nmap -sV <target-ip>
```

**OS Fingerprinting:** Sends a series of unusual/edge-case packets and compares the target's specific response quirks against a large database of known OS behavior signatures — different operating systems implement subtle parts of the TCP/IP stack slightly differently, and these differences are consistent enough to fingerprint reliably.
```
nmap -O <target-ip>
```

**Combined, comprehensive scan:**
```
nmap -A <target-ip>
```
`-A` enables OS detection, version detection, script scanning (Module 6), and traceroute all together — the standard "give me everything" flag once stealth isn't the primary concern.

---

## Timing Templates [-T0 through -T5]

Controls the speed/aggressiveness tradeoff between "fast but noisy/detectable" and "slow but stealthy."

| Template | Name | When to actually use it |
|---|---|---|
| `-T0` | Paranoid | Maximum IDS evasion — extremely slow, used when detection avoidance matters more than time |
| `-T1` | Sneaky | Still very slow, still IDS-evasion focused |
| `-T2` | Polite | Deliberately reduces bandwidth/target load — useful against fragile/legacy systems that could be disrupted by a fast scan |
| `-T3` | Normal | The default — no special speed adjustment |
| `-T4` | Aggressive | Assumes a fast, reliable network — the common choice for lab/CTF environments where detection isn't a concern |
| `-T5` | Insane | Maximum speed, assumes an extremely fast/reliable network, highest chance of inaccurate results due to dropped packets |

**Practical guidance:** `-T4` for lab environments (like your notebook's walkthroughs) where speed matters more than stealth; `-T0`/`-T1` only when the engagement's rules of engagement specifically call for evading detection systems, since they can genuinely take hours for what a `-T4` scan finishes in seconds.

## Quick-reference table

| Scan | Flag | What it needs | Primary use |
|---|---|---|---|
| TCP Connect | `-sT` | No special privileges | Reliable, but noisy/loggable |
| SYN Stealth | `-sS` | Root/admin | The default — fast, half-open |
| FIN/Null/Xmas | `-sF` / `-sN` / `-sX` | Root/admin | Slipping past older/permissive firewalls; unreliable vs. Windows |
| Ping Scan | `-sn` | None | Host discovery only, no ports |
| UDP Scan | `-sU` | Root/admin | Required for DNS/SNMP/DHCP-type services |
| ACK Scan | `-sA` | Root/admin | Mapping firewall statefulness, not finding open ports |
| Idle Scan | `-sI` | Root/admin + a usable zombie host | Hiding your real IP from target logs |
| Version Detection | `-sV` | — | Exact software/version, feeds into CVE lookup |
| OS Fingerprinting | `-O` | Root/admin | Identifying the target OS |
