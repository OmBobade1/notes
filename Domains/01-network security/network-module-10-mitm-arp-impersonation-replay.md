# Module 10 - MITM, ARP Poisoning, Impersonation & Replay Attacks

## Why this comes right after password cracking
Once credentials or a foothold exist, positioning yourself in the middle of network traffic becomes far more valuable — you're no longer guessing passwords, you're watching (or altering) real, live traffic. This module goes far deeper than the earlier conceptual sniffing/MITM content, with real tools.

---

## Ettercap — full ARP poisoning workflow

**Why Ettercap over manually crafting ARP packets:** it handles the entire ARP poisoning process (both directions of the lie — telling the victim you're the router, and telling the router you're the victim) plus traffic capture and filtering, all in one tool, rather than juggling several manual steps.

**Graphical mode (easier to learn on first use):**
```
ettercap -G
```
From the GUI: Sniff → Unified Sniffing → select interface → Hosts → Scan for Hosts → select victim IP as Target 1, gateway/router IP as Target 2 → Mitm → ARP Poisoning → check "Sniff remote connections" → Start.

**Command-line mode (the real, scriptable version):**
```
ettercap -T -M arp:remote /<victim-ip>// /<gateway-ip>//
```
- `-T` — text-mode interface (no GUI)
- `-M arp:remote` — perform ARP poisoning (MITM), including remote connections (traffic the victim sends onward, not just traffic addressed to them directly)
- `/<victim-ip>//` and `/<gateway-ip>//` — the two targets between which you're inserting yourself; the double slash syntax means "any port" for that target

**Why both targets matter, not just the victim:** poisoning only the victim's ARP table lets you see their outbound traffic, but poisoning the gateway's ARP table too is what lets return traffic flow back through you as well — a genuine two-way interception, not a one-directional leak.

---

## Bettercap — the modern successor

**Why Bettercap over Ettercap in current practice:** actively maintained (Ettercap's development has been comparatively slower in recent years), with a more modular architecture and stronger support for modern attack modules beyond basic ARP poisoning — HTTPS interception attempts, Wi-Fi attacks, BLE (Bluetooth Low Energy) reconnaissance, all within one framework.

**Basic interactive usage:**
```
bettercap -iface eth0
```
Launches Bettercap's interactive shell on the specified interface.

**Inside the Bettercap shell — running ARP spoofing as a module:**
```
net.probe on
net.recon on
set arp.spoof.targets <victim-ip>
arp.spoof on
net.sniff on
```
This sequence: discovers live hosts on the network, sets a specific victim as the ARP spoofing target, activates the spoofing, then activates traffic sniffing — a modular, step-by-step version of what Ettercap does in one combined command.

---

## Impersonation Attacks

**What this actually means at a technical level, distinct from ARP spoofing specifically:** ARP spoofing is one *mechanism* for getting into a MITM position. Impersonation is the broader category — pretending to be a trusted entity (a device, a service, sometimes a person in the context of session data) to get a victim or system to trust you when they shouldn't.

**DHCP Spoofing — a distinct impersonation technique:** Setting up a rogue DHCP server that races the legitimate one to respond to a new device's DHCP request first. Since DHCP assigns not just an IP address but also the **default gateway** and **DNS server** a device will use, winning this race means the victim's device is configured, from the moment it joins the network, to route all its traffic through the attacker's chosen gateway and resolve DNS through the attacker's chosen server — a MITM position established without needing ARP spoofing at all, achieved instead by simply being faster than the real DHCP server.
```
# Using a tool like Yersinia or a custom rogue DHCP server (e.g. dnsmasq configured maliciously)
dnsmasq --interface=eth0 --dhcp-range=192.168.1.100,192.168.1.200 --dhcp-option=3,<attacker-ip> --dhcp-option=6,<attacker-ip>
```
`--dhcp-option=3` sets the default gateway option, `--dhcp-option=6` sets the DNS server option — both pointed at the attacker's own address.

---

## Replay Attacks

**What it is:** Capturing a legitimate, valid piece of network traffic (an authentication token, a session cookie, an entire authentication exchange) and **resending it later**, exactly as captured, to fraudulently repeat the action it originally represented — without needing to understand or crack anything about the data itself, since it's already a genuinely valid piece of traffic.

**Why this is a distinct risk from simple eavesdropping:** even if the captured data is encrypted and unreadable to the attacker, a replay attack doesn't require *reading* the content — it only requires resending the exact same bytes again, and if the receiving system doesn't verify that each transaction is genuinely new and unique, it will process the replayed data as if it were a fresh, legitimate request.

**A concrete example scenario:** an attacker captures the exact network traffic of a legitimate funds transfer authorization. Even without ever understanding what's inside the encrypted payload, simply resending that exact captured traffic again could trigger a second, fraudulent transfer — if the receiving system has no protection against this.

**Real mitigation, and why it matters specifically for this attack type:**
- **Nonces** — a unique, single-use random value included in each legitimate request; the server tracks which nonces it's already seen and rejects any request reusing one, making a captured-and-resent request immediately detectable as a replay.
- **Timestamps with a tight validity window** — rejecting any request that's "too old" by the time it's received, limiting how long a captured request remains usable at all.
- **Sequence numbers** — each request in a session includes an incrementing number; a repeated or out-of-order number gets flagged and rejected.

**Why encryption alone (TLS/HTTPS) does not stop replay attacks by itself:** encryption protects the *confidentiality* of data in transit, but says nothing about whether the *same encrypted blob* being sent twice should be honored twice — replay protection has to be built into the application-level protocol logic itself (nonces, timestamps, sequence numbers), not assumed to come for free just because the connection uses HTTPS.

## Quick-reference table

| Technique | Tool/Method | What it establishes |
|---|---|---|
| ARP Poisoning | Ettercap, Bettercap | Classic two-way MITM position via ARP table lies |
| DHCP Spoofing | Rogue DHCP server (dnsmasq, Yersinia) | MITM via controlling gateway/DNS assignment at connection time |
| Replay Attack | Captured traffic, resent as-is | Fraudulently repeats a previously valid action, no decryption needed |

## Mitigation summary across this whole module
- **Static ARP entries** or **Dynamic ARP Inspection** (on managed switches) for critical infrastructure, preventing ARP table poisoning entirely.
- **DHCP Snooping** (a managed-switch feature) — only allows DHCP responses from explicitly trusted, designated ports, blocking rogue DHCP servers connected elsewhere on the network.
- **Nonces/timestamps/sequence numbers** at the application layer specifically to defeat replay attacks, independent of whatever transport encryption is already in place.
