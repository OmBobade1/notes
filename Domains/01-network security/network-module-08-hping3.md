# Module 8 - Packet Crafting with Hping3

## Why this comes right after service enumeration
Everything so far has used tools that construct packets *for* you, following normal protocol rules. Hping3 goes a level lower — it lets you build **completely custom packets by hand**, setting any flag, any flag combination, any content you want. This matters because some tests (firewall rule verification, precise DoS testing, evading detection that specifically looks for standard tool signatures) require packets no standard tool will construct for you.

---

## What Hping3 actually is
A command-line packet crafting and analysis tool, often described as "`ping` on steroids" — where regular `ping` only sends basic ICMP Echo Requests, Hping3 can construct and send essentially any TCP, UDP, or ICMP packet with fully custom flags, ports, and payloads.

**What it's for, attacker's perspective:** Standard tools like Nmap, while flexible, still produce recognizable, somewhat standardized packet patterns. Hping3 lets you build something genuinely unusual and specific to a test scenario — verifying one exact firewall rule, simulating one exact attack pattern, or crafting packets that don't match any known tool's fingerprint.

---

## Basic Syntax and Modes

**Default mode — TCP:**
```
hping3 <target-ip>
```
By default, Hping3 sends TCP packets to port 0, mostly used just to confirm basic reachability/response behavior.

**Specifying a target port:**
```
hping3 -p 80 <target-ip>
```

**Setting specific TCP flags manually — this is the real power of the tool:**
```
hping3 -S -p 80 <target-ip>       # SYN flag only
hping3 -A -p 80 <target-ip>       # ACK flag only
hping3 -F -p 80 <target-ip>       # FIN flag only
hping3 -R -p 80 <target-ip>       # RST flag only
hping3 -P -p 80 <target-ip>       # PSH flag only
```
**Why manually setting flags matters:** this is the exact same underlying logic as Nmap's SYN/FIN/Null/Xmas scans (Module 5), except here you have complete manual control — useful when you need a flag combination Nmap's built-in scan types don't offer, or when testing one very specific firewall rule's exact behavior rather than running a broad scan.

---

## Port Scanning with Hping3

**Scanning a range of ports manually:**
```
hping3 -S <target-ip> -p ++1 -c 1000
```
`-p ++1` tells Hping3 to increment the destination port by 1 with each packet sent, starting from an initial port — `-c 1000` sends 1000 packets total, effectively scanning 1000 sequential ports one at a time. This is a genuinely manual, low-level equivalent to what Nmap automates — worth understanding specifically because it demonstrates *what Nmap is actually doing under the hood* when it performs a SYN scan.

---

## Using Hping3 for Denial of Service Testing

**Important context before this section:** these techniques directly connect to the DoS/DDoS content already covered conceptually — this is the practical, hands-on version, meant strictly for authorized lab/testing environments where you have explicit permission to test service resilience, never against anything you don't own or have written authorization for.

### SYN Flood
```
hping3 -S <target-ip> -p 80 --flood --rand-source
```
- `-S` — SYN flag only, deliberately never completing the handshake
- `--flood` — send packets as fast as possible, without waiting for replies at all
- `--rand-source` — randomizes the apparent source IP on every packet

**Why this actually causes a denial of service:** this is the exact mechanism described conceptually in the earlier DoS module — the target allocates resources for each half-open connection attempt, and with `--rand-source` randomizing the apparent source constantly, the target can't simply block "one troublemaker IP," since every packet appears to come from a different address.

### Land Attack
```
hping3 -S <target-ip> -a <target-ip> -p 80
```
`-a` spoofs the source address to be **identical to the target's own address** — the target ends up sending its SYN/ACK reply to itself, which can confuse or crash improperly-hardened network stacks that don't expect to receive their own traffic back at themselves. Largely a legacy technique against modern, patched systems, but worth knowing as a specific, named technique.

### Smurf Attack (conceptual + Hping3's role)
**The core technique:** send an ICMP Echo Request to a network's **broadcast address**, with the source address spoofed to be the actual victim's address. Every device on that network receives the broadcast ping and replies — but sends every single reply directly to the spoofed victim address, not back to the real sender. One packet in, potentially hundreds of replies out, all aimed at the victim — this is the exact amplification-attack concept from the earlier DoS module, given its specific historical name and mechanism.
```
hping3 -1 <broadcast-address> -a <victim-ip> --flood
```
`-1` specifies ICMP mode. **Modern relevance:** most networks today are configured to not respond to broadcast pings at all specifically because Smurf attacks were such a well-known historical problem — this technique is now largely of historical/educational value rather than a live real-world threat, but understanding it is what makes modern amplification attacks (DNS/NTP amplification, covered conceptually in the DoS module) make sense, since they're the same core idea applied to different protocols.

---

## Firewall Rule Testing with Hping3
Beyond attacks, Hping3 is genuinely useful defensively too — precisely verifying that a firewall rule behaves exactly as intended:
```
hping3 -S <target-ip> -p 22 -c 1
```
Send exactly one crafted SYN packet at port 22 and observe the exact response — confirms definitively whether that specific port/rule is open, closed, or filtered, with total control over exactly what was sent, useful when you need to document precisely reproducible evidence for a report rather than relying on a broader tool's summarized output.

## Quick-reference table

| Command pattern | Purpose |
|---|---|
| `hping3 -S -p <port> <ip>` | Send a single custom SYN packet to a specific port |
| `hping3 -S <ip> -p ++1 -c 1000` | Manual, low-level port scan |
| `hping3 -S <ip> -p 80 --flood --rand-source` | SYN flood with randomized source (DoS testing) |
| `hping3 -S <ip> -a <ip> -p 80` | Land attack (spoofed self-source) |
| `hping3 -1 <broadcast> -a <victim> --flood` | Smurf attack (ICMP amplification) |
