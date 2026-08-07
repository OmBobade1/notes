# 03 - Sniffing & Man-in-the-Middle (Explained Simply)

## Why this comes right after enumeration
Enumeration (file `02`) is *asking* a service questions directly. Sniffing is different — you're not asking anyone anything, you're just quietly **listening to traffic that's already flowing past you** on the network, the same way you'd overhear a conversation at the next table without either person talking to you.

---

## What is sniffing, really?
Imagine a hallway where everyone's conversations echo a little. Normally you only pay attention to your own conversation — but if you *chose* to listen carefully, you could pick up pieces of everyone else's too. **Network sniffing** is a tool doing exactly that: capturing all the data packets flowing across a network segment, not just the ones meant for your device.

**Wireshark** is the standard tool for this — it captures every packet your network card can see and lets you inspect each one in detail: source, destination, protocol, and the actual content if it isn't encrypted.

## Why sniffing works at all — the "shared hallway" problem
On many networks (especially older setups using hubs, or Wi-Fi in certain configurations), data doesn't travel directly from sender to receiver like a private letter — it's more like everyone in the hallway hears everything, and each device is just supposed to *politely ignore* traffic that isn't addressed to it. Sniffing tools simply stop being polite and read everything that passes by.

Modern wired networks mostly use **switches** instead of hubs, which are smarter and only send traffic to the intended device — this significantly reduces what a sniffer can passively see just by being connected. Which is exactly why attackers developed a way to force traffic to come to them anyway — that's ARP spoofing, below.

## ARP Spoofing — tricking devices into sending you their mail
**ARP (Address Resolution Protocol)** is how devices on a local network find each other. When Device A wants to talk to Device B, it basically shouts "who has this IP address?" and the real owner replies "that's me, here's my hardware address (MAC address)."

**ARP Spoofing (or ARP Poisoning)** is an attacker lying in response to that shout — telling both the victim's computer and the router "I'm the other one," positioning themselves in the middle of the conversation without either side realizing it.

```
arpspoof -i eth0 -t 192.168.1.10 192.168.1.1
```
This tells the victim (`192.168.1.10`) that the attacker's machine is actually the router (`192.168.1.1`). Now, every bit of traffic the victim sends "to the router" actually goes to the attacker first, who can read it, and then quietly forward it on to the real router so nothing looks broken from the victim's side.

## Man-in-the-Middle (MITM) — the bigger picture
ARP spoofing is just *one way* to get into the middle of a conversation. **Man-in-the-Middle** is the general concept: an attacker secretly sits between two parties who believe they're talking directly to each other, and can read, and sometimes alter, everything passing between them.

**Other common ways to end up in the middle:**
- **Rogue Wi-Fi Access Point** — setting up a fake Wi-Fi network with a trustworthy-looking name ("Free_Airport_WiFi"), so victims connect directly through the attacker's own device without any spoofing trick needed at all.
- **DNS Spoofing** — instead of intercepting existing traffic, tricking a victim's device into thinking a fake DNS server (which the attacker controls) is the legitimate one, so every "what's the address for bank.com?" question gets answered with the attacker's own fake address instead.

## Why encryption is the actual defense (and why it isn't always enough)
This is exactly why HTTPS (encrypted web traffic) matters so much, and directly connects to the weak TLS and HSTS content already covered in the web security series. Even if an attacker successfully gets in the middle of a conversation via ARP spoofing, if the traffic itself is properly encrypted, all they see is scrambled, unreadable data — being in the middle doesn't help if you can't understand what you intercepted.

**Why it's not a complete defense on its own:** an attacker in the middle can still attempt a **downgrade attack** (forcing a weaker, breakable form of encryption — covered in the weak TLS configuration file in the web series), or present a fake certificate hoping the victim clicks through a browser warning without reading it carefully.

## What sniffing captures in practice
- **Plaintext protocols** (unencrypted HTTP, FTP, Telnet) — sniffing captures everything readable, including passwords typed in directly, with zero effort.
- **Encrypted protocols** (HTTPS, SSH) — sniffing still captures *that* a connection happened, when, and roughly how much data moved (called **metadata**), but not the actual content, assuming the encryption itself is properly configured.

## Quick-reference table

| Technique | What it does |
|---|---|
| Passive sniffing | Listening to traffic already visible on the network segment |
| ARP Spoofing | Tricking two devices into routing their traffic through the attacker first |
| Rogue Access Point | Victim connects directly to an attacker-controlled Wi-Fi network |
| DNS Spoofing | Feeding a victim fake DNS answers to redirect their traffic |

## Ethical note
Same rule as every file before this — only perform sniffing or MITM techniques on a network you own or have explicit written authorization to test. Intercepting someone else's traffic without consent is both a serious privacy violation and illegal in essentially every jurisdiction.
