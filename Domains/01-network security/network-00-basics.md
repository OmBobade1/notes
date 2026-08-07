# 00 - Networking Basics (Explained Simply)

## Why this is file 00, before anything else
You can't scan, attack, or defend a network without first knowing what's actually happening when two computers talk to each other. This file is that groundwork — no security stuff yet, just "how does the internet even work."

---

## What is an IP address?
Every device on a network needs an address, the same way every house needs a street address so mail can find it. An **IP address** is that — a set of numbers like `192.168.1.10` that identifies one specific device on a network.

- **Public IP** = your house's real street address, visible to the whole world (this is the address your home router uses to talk to the internet).
- **Private IP** = like a room number *inside* your house, only meaningful to people already inside — devices on your home Wi-Fi have private IPs like `192.168.x.x` that only make sense within your own network.

## What is a port?
If the IP address is the street address of a building, a **port** is the specific door or window into that building. A single device (one IP address) can run many different services at once — a web server, an email server, a file-sharing service — and each one "listens" on its own numbered door, from 0 to 65535.

**A few doors you'll see constantly:**
| Port | What normally lives there |
|---|---|
| 21 | FTP (file transfer) |
| 22 | SSH (secure remote login) |
| 23 | Telnet (old, insecure remote login) |
| 25 | Email sending (SMTP) |
| 53 | DNS (translates website names into IP addresses) |
| 80 | Regular websites (HTTP) |
| 443 | Secure websites (HTTPS) |
| 3389 | Remote Desktop (Windows) |

Knowing which door is open on a target tells you what kind of service you're dealing with, before you even try to interact with it.

## What is a protocol?
A **protocol** is just an agreed-upon set of rules for how two things communicate — like a language both sides agree to speak. HTTP is the protocol (the language) that web browsers and websites speak to each other. SSH is the protocol for securely logging into a remote machine. Every "door" (port) above typically has one specific protocol/language spoken through it.

## The OSI Model — the 7-layer cake of networking
This is the standard mental map for "what actually happens" when data travels from one device to another. Think of it as a **7-layer cake**, where each layer does one specific job and hands things off to the next layer.

| Layer | Name | Plain-English job | Real example |
|---|---|---|---|
| 7 | Application | What the user actually interacts with | Your web browser, email app |
| 6 | Presentation | Makes sure data looks the same on both ends (formatting, encryption) | SSL/TLS encryption happens conceptually here |
| 5 | Session | Keeps track of an ongoing conversation between two devices | Login sessions |
| 4 | Transport | Breaks data into chunks and makes sure they all arrive | TCP, UDP |
| 3 | Network | Figures out the best path/route across networks | IP addresses, routers |
| 2 | Data Link | Handles communication between devices on the *same* local network | MAC addresses, switches |
| 1 | Physical | The actual physical wires/signals/Wi-Fi radio waves | Ethernet cables, Wi-Fi |

You don't need to memorize this like a school test — the useful part is: **when something breaks or gets attacked, you can ask "which layer is this actually happening at?"** A cable getting physically cut is Layer 1. A fake Wi-Fi network tricking your laptop is Layer 2. A hacker intercepting your website traffic is happening up around Layer 4-7.

## TCP vs UDP — the two ways data actually travels

**TCP (Transmission Control Protocol)** — like sending a package with tracking and a signature required. Every piece of data is confirmed as received; if something's missing, it gets resent. Slower, but reliable. Used for things where you can't afford to lose data — loading a webpage, sending a file.

**UDP (User Datagram Protocol)** — like shouting something across a room. It's fast, but nobody confirms you heard it. Used where speed matters more than perfection — video calls, online gaming (a tiny bit of lost data just means a brief glitch, not a broken connection).

## Why all of this matters for security testing
Every single technique in the files that follow — scanning, sniffing, exploiting — is really just: *finding out which doors (ports) are open, what language (protocol) is being spoken through them, and whether the rules of that conversation can be tricked, intercepted, or broken.* This file is the map everything else gets plotted on.
