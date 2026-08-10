# Module 6 - Nmap NSE Scripts, Zenmap & Firewall/IDS Evasion

## Why this comes right after core scan types
Module 5 covered *finding* ports and services. This module covers two separate but related things: **NSE**, which lets Nmap actually interrogate what it finds far beyond a simple open/closed answer, and **evasion techniques**, for when the scan itself needs to avoid being detected or blocked in the first place.

---

## NSE — the Nmap Scripting Engine

**What it is:** A framework built into Nmap that runs Lua scripts against discovered hosts/ports to do far more than basic detection — vulnerability checks, brute-forcing, deeper enumeration, even active exploitation of specific known issues.

**What it's for, attacker's perspective:** Turns Nmap from "here's what's open" into "here's what's open, AND here's a specific known weakness in it, tested automatically" — collapsing what used to be several separate manual steps into one scan.

### Running scripts

**Default script set:**
```
nmap -sC <target-ip>
```
`-sC` runs Nmap's curated "default" script category — scripts considered safe and broadly useful for general discovery, without being aggressive/intrusive.

**A single named script:**
```
nmap --script=banner <target-ip>
```

**A category via wildcard:**
```
nmap --script=http* <target-ip>
```
Runs every script whose name starts with "http" — useful when you know the service type but want comprehensive coverage without naming each script individually.

**Multiple specific scripts:**
```
nmap --script=http,banner <target-ip>
```

**Passing arguments into a script:**
```
nmap --script snmp-sysdescr --script-args snmpcommunity=admin <target-ip>
```
This example specifically feeds a custom SNMP community string into the script, connecting directly to the SNMP enumeration content already covered — instead of manually running `snmpwalk` with a guessed community string, NSE automates the same idea as part of a broader scan.

### Real, practical NSE examples worth knowing individually

**Directly checking for SQL injection:**
```
nmap -p80 --script http-sql-injection <target-ip>
```
This connects NSE directly to the SQL Injection content from the web security series — the same vulnerability class, now checked automatically as part of network-level recon rather than manual browser testing.

**Directly checking for reflected XSS-enabling behavior:**
```
nmap -p80 --script http-unsafe-output-escaping <target-ip>
```

**Brute-forcing DNS subdomains:**
```
nmap -Pn --script=dns-brute domain.com
```
An automated version of manually guessing common subdomain names, connecting back to the DNS reconnaissance content in Module 3.

**Comprehensive, safe SMB enumeration in one command:**
```
nmap -n -Pn -vv -O -sV --script smb-enum*,smb-ls,smb-mbenum,smb-os-discovery,smb-s*,smb-vuln*,smbv2* -vv <target-ip>
```
This single command chains together nearly everything covered manually in the enumeration module's SMB section — shares, OS version, vulnerability checks — into one automated pass.

---

## Zenmap — Nmap's graphical interface

**What it is:** The official GUI front-end for Nmap, generating the same underlying scan commands you'd type manually, but with visual output — including a network topology map showing discovered hosts and their relationships.

**Why it's genuinely useful, not just "Nmap for people who don't like the terminal":**
- **Command construction assistance** — helpful when first learning the tool, since it shows you exactly which flags a given profile builds, which you can then learn to type manually.
- **Result comparison** — Zenmap can visually diff two scan results against each other, immediately highlighting what changed between two points in time (new open port, a service that disappeared) — genuinely useful for tracking a target's changes across a longer engagement, without manually re-reading two full text outputs side by side.
- **Topology view** — for a larger network, seeing discovered hosts laid out visually can reveal network structure/segmentation faster than reading a flat list.

**Practical reality:** most experienced testers still do the majority of real work from the command line (faster, more scriptable, easier to combine with other tools) — Zenmap's real value is learning and result review/comparison, not day-to-day scanning speed.

---

## Firewall / IDS Evasion and Spoofing

**Why this category exists at all:** every technique here directly answers "how do I get useful scan results against a target that's actively trying to detect or block scanning," rather than assuming an undefended target.

### Fragmentation [-f]
**What it does:** Splits your scan's packets into smaller fragments before sending.
```
nmap -f <target-ip>
```
**Why this works as evasion:** Older/simpler packet-filtering firewalls and IDS systems inspect traffic packet by packet, and some don't properly reassemble fragments before applying their inspection rules — a signature that would trigger on one complete packet can slip through when split across several smaller fragments the inspecting device never reassembles before deciding whether to allow it.

### Custom MTU [--mtu]
```
nmap --mtu 32 <target-ip>
```
Lets you set a specific fragment size manually (must be a multiple of 8) rather than relying on Nmap's default fragmentation size — useful for fine-tuning evasion against a specific known filtering device's inspection behavior.

### Decoy Scan [-D]
**What it does:** Makes the scan appear to originate from multiple IP addresses simultaneously — your real one, hidden among several decoys.
```
nmap -D 192.168.1.101,192.168.1.102,192.168.1.103,192.168.1.23 <target-ip>
```
**Why this works as evasion:** The target's logs now show scan traffic from five different apparent sources at once. Even if the scan itself is detected, an analyst now has to determine *which* of the five IPs was the real attacker and which were decoys — genuinely slowing down attribution and response, even though it doesn't prevent detection of the scan activity itself.

**Important practical note:** decoy IPs should generally be genuinely live hosts (not just made-up addresses) for the technique to look realistic and not immediately stand out as obviously fake traffic patterns.

### Scan from a spoofed source [-S]
```
nmap -S www.spoofed-source.com <target-ip>
```
Explicitly sets the apparent source of the scan to a specified address — often requires `-e <interface> -Pn` alongside it, since the true reply traffic won't actually route back to you, so you frequently need packet capture on the specified interface to see results at all rather than relying on normal reply routing.

### Custom source port [-g]
```
nmap -g 53 <target-ip>
```
Forces the scan to originate from a specific source port — port 53 specifically is a common choice, since many firewalls are configured to trust traffic that *appears* to be DNS responses (source port 53) more readily than arbitrary ports, exploiting an overly trusting firewall rule rather than a technical protocol weakness.

### Proxy relaying [--proxies]
```
nmap --proxies http://192.168.1.1:8080,http://192.168.1.2:8080 <target-ip>
```
Routes the scan traffic through one or more HTTP/SOCKS4 proxies before it reaches the target — the target sees the proxy's address, not yours, directly.

### Appending random data [--data-length]
```
nmap --data-length 200 <target-ip>
```
Pads packets with extra random data, altering their size/signature — some detection systems specifically fingerprint scans partly by their default, predictable packet sizes, and this simple change can be enough to avoid matching that specific fingerprint.

### Bad checksums [--badsum]
Sends packets with an intentionally incorrect checksum. A real, properly-implemented TCP/IP stack will silently discard a packet with a bad checksum — but some firewalls/IDS systems, in an effort to be permissive or due to simpler implementation, respond to packets regardless of checksum validity. **Why this is genuinely useful:** if you get a response despite an invalid checksum, that's a strong signal you're not actually talking to a real, properly-implemented host stack at all — you're talking to a security device (a firewall/IDS) that's answering on the real host's behalf, revealing its presence.

### Putting it together — a real combined evasion command
```
nmap -f -T0 -n -Pn --data-length 200 -D 192.168.1.101,192.168.1.102,192.168.1.103,192.168.1.23 <target-ip>
```
This combines fragmentation, the slowest/most paranoid timing template, disabled DNS resolution and host discovery (reducing extra traffic that could itself be detected separately), padded packet size, and a decoy scan — a genuinely stealth-focused scan, at the direct cost of being dramatically slower than a normal `-T4` scan.

## Quick-reference table

| Technique | Flag | What it actually defeats |
|---|---|---|
| Fragmentation | `-f` | Firewalls/IDS that don't reassemble fragments before inspecting |
| Custom MTU | `--mtu` | Same as above, with fine-tuned fragment size |
| Decoy scan | `-D` | Attribution — hides your real IP among fake ones |
| Spoofed source | `-S` | Hides your real source address entirely |
| Custom source port | `-g` | Firewall rules that over-trust specific "known" ports |
| Proxy relay | `--proxies` | Direct traceability to your real IP |
| Random data padding | `--data-length` | Signature-based detection on default packet size |
| Bad checksum | `--badsum` | Reveals whether a security device is answering on the host's behalf |
