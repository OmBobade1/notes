# Module 0 - OSI Model & TCP/IP Fundamentals

## Why this had to exist, and why it's numbered 0
Every module from 1 onward assumes you already know what a "Layer 2 attack" or "Layer 3 device" means. This module is that missing foundation — genuinely basic, but not skippable, since the specific layer something operates at is *why* the attack/defense in later modules works the way it does. Numbered `0` deliberately, to sit before Module 1 without renumbering everything else already built.

---

## The OSI Model — seven layers, what each one actually does

The OSI (Open Systems Interconnection) model is a conceptual framework describing how data travels from one device to another, broken into seven distinct jobs, stacked on top of each other. Data starts at Layer 7 on the sending device, gets wrapped with additional information as it passes down through each layer, physically travels across Layer 1, then gets unwrapped back up through the layers on the receiving device.

**Layer 7 - Application:** What the user/software actually interacts with directly — a web browser, an email client. Protocols like HTTP, FTP, SMTP live here. When you use `curl` (Module 7) or a web browser, you're operating at this layer.

**Layer 6 - Presentation:** Handles data formatting, translation, and encryption/decryption so both ends can understand the data regardless of how it's internally represented. SSL/TLS encryption conceptually operates here — the "S" in HTTPS.

**Layer 5 - Session:** Manages the lifetime of a connection between two devices — establishing, maintaining, and tearing down a session. Login sessions, and the "state" a stateful firewall (Module 6, ACK scans) tracks, are session-layer concepts.

**Layer 4 - Transport:** Responsible for breaking data into manageable chunks (segments) and ensuring reliable delivery — this is exactly where **TCP and UDP** live (detailed below). Port numbers are a Layer 4 concept — they're what lets a single device run many services simultaneously, each identified by its own port.

**Layer 3 - Network:** Responsible for logical addressing (IP addresses) and routing — determining the best path for data across multiple networks. **Routers operate here** (Module 1) — this is exactly why a router "decides how to forward traffic between different networks": that decision-making is fundamentally a Layer 3 function.

**Layer 2 - Data Link:** Responsible for communication between devices on the *same* local network segment, using MAC addresses rather than IP addresses. **Switches operate here** (Module 1) — a switch's entire job (learning which MAC address sits on which port) is a Layer 2 function. **Wi-Fi frame types** (Module 16 — beacon, deauthentication, association frames) are also Layer 2 — this is exactly why deauthentication attacks are described as exploiting a Layer 2 weakness specifically.

**Layer 1 - Physical:** The actual physical transmission medium — cables, radio waves, voltage levels, light pulses in fiber optics. **Hubs operate here** (Module 1) — a hub has no intelligence precisely because it doesn't even process anything above Layer 1; it just repeats the electrical signal to every port.

## Why memorizing the layer numbers isn't really the point
The actual practical value: when something in a later module is described as a "Layer 2 attack" (MAC flooding, ARP poisoning, deauthentication) versus a "Layer 3 attack" (IP spoofing) versus a "Layer 4 concept" (a port scan), knowing which layer you're operating at tells you **which devices and which defenses are even relevant** — a Layer 3 firewall rule does nothing to stop a Layer 2 MAC flooding attack, because they're not operating at the same level of the stack at all.

---

## TCP vs UDP — the two Layer 4 protocols, in real depth

### TCP (Transmission Control Protocol)

**What makes it "reliable":** Before any actual data is sent, TCP performs the **three-way handshake** — this is the exact same handshake referenced throughout Module 5's scan types:
```
Client -------- SYN --------> Server
Client <---- SYN/ACK -------- Server
Client -------- ACK --------> Server
```
- **SYN** — "I want to start a connection"
- **SYN/ACK** — "Acknowledged, I also want to connect"
- **ACK** — "Confirmed, connection established"

Only after this three-step exchange completes does actual data start flowing. Every subsequent piece of data sent is individually acknowledged by the receiver — if an acknowledgment doesn't arrive within an expected time, TCP automatically resends that specific piece of data. This is precisely why TCP is called reliable: the protocol itself guarantees that data either arrives correctly, or the sender knows it didn't and retries.

**Why this matters directly for Module 5:** every scan type in Module 5 (`-sT`, `-sS`, `-sF`/`-sN`/`-sX`) is really just a different, deliberate manipulation of this exact three-step process — completing it fully (`-sT`), stopping partway through (`-sS`), or sending flags that don't belong in this handshake at all (`-sF`/`-sN`/`-sX`).

### UDP (User Datagram Protocol)

**What makes it "unreliable" (by design, not by flaw):** UDP has no handshake at all — data is simply sent, with no confirmation that it arrived, no automatic retry, and no guaranteed ordering if multiple pieces are sent.

**Why this "unreliability" is actually a deliberate, useful trade-off:** the overhead of TCP's handshake and constant acknowledgment tracking adds real latency — for use cases where a small amount of lost data is an acceptable, minor glitch (a live video call frame, an online game's position update) but *delay* would be far worse than loss, UDP's speed is the better trade-off. This is exactly why DNS (mostly), video streaming, and VoIP commonly use UDP.

**Why this directly matters for Module 5's UDP scan:** because there's no handshake to manipulate or observe, a UDP scan (`-sU`) has to work completely differently from every TCP scan type — it can only infer a port's state from whether an ICMP "unreachable" error comes back, which is exactly why UDP scanning was described in Module 5 as slower and more ambiguous by nature, not as a limitation of Nmap itself.

---

## Ports — why they exist, and how they connect everything

**The core problem ports solve:** a single device has one IP address, but needs to run many different services simultaneously (a web server, an SSH server, a mail server, all on one machine) — ports are the mechanism that lets many different services share one IP address, each identified by its own number, 0 to 65535.

**Well-known ports (0-1023):** reserved for standard, common services — port 21 (FTP), 22 (SSH), 80 (HTTP), 443 (HTTPS) — this is exactly why Module 7's per-service enumeration is organized around specific, predictable port numbers; these assignments are standardized, not arbitrary.

**Why every single module from 1 onward ultimately traces back to this concept:** scanning (Module 5) is fundamentally "which ports are open." Enumeration (Module 7) is "what's actually listening on this specific open port." Netcat (Module 4) manually connects to a specific port. Firewall rules (Module 15) are largely expressed in terms of which ports are allowed through. Nothing in the modules that follow makes sense without this Layer 4 concept as the foundation underneath it.

## Quick-reference table

| Layer | Number | What operates here (from this repo's modules) |
|---|---|---|
| Application | 7 | HTTP, FTP, SMTP — Netcat's raw interaction target |
| Presentation | 6 | TLS/SSL encryption |
| Session | 5 | Connection state — what stateful firewalls (Module 6 ACK scan) track |
| Transport | 4 | TCP/UDP, ports — the basis of every scan type in Module 5 |
| Network | 3 | IP addressing, routing — Routers (Module 1) |
| Data Link | 2 | MAC addresses — Switches (Module 1), Wi-Fi frames (Module 16) |
| Physical | 1 | Raw signal transmission — Hubs (Module 1) |

## This module now properly precedes Module 1
With this in place, the sequence genuinely starts from zero-assumed-knowledge: Module 0 (this file) → Module 1 (devices, now makes sense in terms of which OSI layer each operates at) → everything else, unchanged.
