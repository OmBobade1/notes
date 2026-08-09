# Module 1 - Network Devices

## Why this is the actual starting point
Before any scanning or exploitation, you need to know what physical/logical devices actually make up a network, because **different devices fail in different ways** — a hub leaks everything to everyone by design, a switch doesn't, but can be tricked into behaving like one. Attacking a network without knowing which device you're dealing with means attacking blind.

---

## Router

**What it is:** A device that connects two or more *different* networks together and decides how to forward traffic between them — e.g. your home network to the wider internet. Operates at Layer 3 (Network layer) of the OSI model, meaning it makes decisions based on IP addresses.

**What it's for, from an attacker's perspective:** A router is the single chokepoint all traffic passes through to reach the outside world — compromising it doesn't just give access to one device, it gives visibility and control over **everything flowing through the entire network**.

**How it's actually beneficial to an attacker:**
- **Default credentials** — a huge number of routers are deployed and left with factory-default admin logins (`admin`/`admin`, `admin`/`password`). Getting into the router's admin panel means the attacker can view/change DNS settings (redirecting *every* device on the network to malicious sites), view connected device lists, or open ports to expose internal devices to the internet directly.
- **Firmware vulnerabilities** — routers frequently run outdated firmware with known CVEs (connects back to the vulnerability assessment content), and unlike a laptop, most people never manually update their router's firmware — it just runs the same vulnerable version for years.
- **Once compromised, it's a pivot point** — an attacker with control of the router sits at the same "everything passes through me" position as a Man-in-the-Middle attack, without needing ARP spoofing at all, since the router is legitimately in that position already.

**Real commands to check a router's exposure:**
```
nmap -p 80,443,8080,7547 <router-ip>
```
Port `7547` specifically is TR-069, a remote management protocol that has been the target of real-world large-scale router botnet attacks (Mirai variants) — worth explicitly checking, since it's not one of the "obvious" ports people think to test.

**Mitigation:**
- Change default admin credentials immediately on deployment — this single step blocks the most common real-world router compromise path.
- Keep firmware updated; enable auto-update where the router supports it.
- Disable remote/WAN-side management (don't allow the admin panel to be reachable from the internet at all, only from inside the local network).
- Disable TR-069/CWMP unless the ISP genuinely requires it and it's properly secured.

---

## Switch

**What it is:** A device that connects multiple devices *within the same local network* and forwards data intelligently, based on MAC addresses — Layer 2 (Data Link layer). Unlike a hub, a switch learns which device is on which physical port and only sends traffic to the specific port that device is on.

**What it's for, from an attacker's perspective:** Because a switch is *smart* about where it sends traffic, it's specifically designed to defeat the kind of passive sniffing that works trivially on a hub. This means an attacker on a switched network has to actively *trick* the switch, rather than just passively listening.

**How it's actually beneficial to an attacker (once tricked):**
- **MAC Flooding** — a switch keeps a table (the CAM table) mapping MAC addresses to physical ports, with limited memory. Flooding the switch with an enormous number of fake, random MAC addresses overflows this table. When it's full, many switches **fail open** — reverting to hub-like behavior, broadcasting all traffic to all ports, specifically because it no longer knows where anything is supposed to go. This turns a switch into exactly the sniffing-friendly environment it was designed to prevent.
```
macof -i eth0
```
This tool (part of the dsniff suite) floods the local switch with random source MAC addresses, attempting to trigger this fail-open condition.

- **VLAN Hopping** — VLANs (Virtual LANs) are used to logically separate traffic on the same physical switch (e.g. keeping the Finance department's traffic separate from Guest Wi-Fi, even on shared hardware). VLAN hopping is a technique to send traffic that escapes its assigned VLAN and reaches a different one it shouldn't be able to reach — most commonly via a technique called "double tagging," exploiting how some switches process nested VLAN tags.

**Mitigation:**
- **Port security** — configure switches to limit how many MAC addresses are allowed per port, and lock/shut down a port automatically if that limit is exceeded (directly defeats MAC flooding).
- Disable unused switch ports entirely.
- Explicitly configure VLAN trunking settings rather than relying on defaults — many VLAN hopping techniques exploit default/auto-negotiated trunk settings that were never deliberately configured.

---

## Hub

**What it is:** The simplest and oldest connecting device — Layer 1 (Physical layer). A hub has **no intelligence at all**: whatever data arrives on one port gets broadcast out to *every other port*, regardless of who it's actually meant for.

**What it's for, from an attacker's perspective:** This is the easiest possible sniffing environment — an attacker plugged into any port on a hub receives a copy of every single packet flowing through the entire hub, with zero effort and zero trickery required, since that's simply how the hub is designed to work.

**How it's actually beneficial to an attacker:**
- Simply running Wireshark while connected to a hub captures everyone's traffic automatically — no ARP spoofing (file `03`), no MAC flooding, nothing — the device does the work for you by design.

**Mitigation:** The real mitigation is **don't use hubs** — they're obsolete in any environment that takes security seriously, replaced entirely by switches. If you ever encounter one during an assessment, its mere presence is itself a finding worth flagging.

---

## Bridge

**What it is:** A device that connects two network *segments* together, operating at Layer 2 like a switch, filtering traffic between the segments based on MAC address. Functionally a simpler, often two-port predecessor to what modern multi-port switches do.

**What it's for, from an attacker's perspective:** Largely the same considerations as a switch, since a bridge is really an earlier, less sophisticated version of the same Layer 2 filtering concept — most bridge-specific attacks are really just switch attacks (MAC-table exhaustion) applied to older/simpler hardware.

**Mitigation:** Same principles as switches — port security and proper segmentation configuration; modern networks rarely deploy standalone bridges anymore, having been almost entirely replaced by switches.

---

## Quick-reference table

| Device | OSI Layer | Key weakness an attacker exploits |
|---|---|---|
| Router | 3 (Network) | Default credentials, outdated firmware, remote management exposure |
| Switch | 2 (Data Link) | MAC flooding (fail-open), VLAN hopping |
| Hub | 1 (Physical) | No filtering at all — passive sniffing works by default |
| Bridge | 2 (Data Link) | Same MAC-table exhaustion risk as switches, simpler/older hardware |

## Why this module had to come before anything else
Every technique in the modules that follow assumes you already know: is there a switch here I need to trick, or a hub where I don't need to bother? Is the router itself in-scope and worth targeting directly, or just the path traffic travels through? Skipping this step means not understanding *why* a given attack technique works at all — you'd be running commands without knowing what makes them effective against this specific piece of infrastructure.
