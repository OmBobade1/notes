# 01 - Network Scanning (Explained Simply)

## Why this comes right after the basics
Now that you know what IP addresses, ports, and protocols are (file `00`), the first real thing a tester actually *does* is find out: which devices exist on this network, and which doors (ports) are open on them? That's scanning.

---

## The core idea, in one sentence
Scanning is just **knocking on every door of every house on a street, and writing down which doors actually answer.**

## Step 1: Finding devices (Host Discovery)
Before you can check doors, you need to know which houses (devices) even exist on the street (network). This is called **host discovery** or a **ping sweep** — quickly checking a whole range of IP addresses to see which ones respond at all, without yet checking any specific ports.

```
nmap -sn 192.168.1.0/24
```
This says: "check every address from 192.168.1.1 to 192.168.1.254, and just tell me which ones are alive" — like walking down a street and noting which houses have their lights on, before knocking on any doors.

## Step 2: Checking which doors are open (Port Scanning)
Once you know a device exists, the next question is: which ports (doors) are open on it? Nmap (Network Mapper) is the standard tool for this.

```
nmap 192.168.1.10
```
This checks the most common ~1000 ports on that one device and reports back which ones answered.

## The three answers a port can give you

| Answer | What it means in plain terms |
|---|---|
| **Open** | Someone answered the door — a service is actively running and listening here |
| **Closed** | Nobody's home, but the *house* exists — the device responded, just says "nothing here" |
| **Filtered** | You knocked and got total silence — a firewall is likely blocking your knock entirely, so you can't even tell if anyone's home |

## Common scan types, explained simply

**TCP Connect Scan (`-sT`)** — the "polite knock." Completes a full, real connection to the door, exactly like a normal visitor would. Reliable, but noisy — it leaves a clear record in the target's logs, since it's indistinguishable from a real connection attempt.

```
nmap -sT 192.168.1.10
```

**SYN Scan / "Half-Open" Scan (`-sS`)** — the "knock and run." Starts the connection process but never finishes it — just enough to see if the door *would* open, then backs away. Faster and stealthier than a full connect scan, since many logging systems only record fully-completed connections. This is Nmap's default scan when run with root/administrator privileges.

```
nmap -sS 192.168.1.10
```

**UDP Scan (`-sU`)** — checking the doors that use the "shout across the room" method (UDP, from file `00`) instead of the reliable knock (TCP). Slower and less reliable to scan, because UDP doesn't confirm anything by design — silence could mean "closed" or could just mean "no one felt like shouting back."

```
nmap -sU 192.168.1.10
```

**Service/Version Detection (`-sV`)** — once you know a door is open, this asks "okay, but *what exactly* is behind this door?" — not just "port 80 is open" but "this is specifically Apache 2.4.29" — which matters enormously, since knowing the exact version is what lets you check it against known vulnerabilities later (connects to file `18` in the web series — outdated components).

```
nmap -sV 192.168.1.10
```

**OS Detection (`-O`)** — tries to guess what operating system the device is running, based on subtle differences in how different OSes respond to network probes (every OS implements networking slightly differently under the hood, and Nmap has a large fingerprint database of these differences).

```
nmap -O 192.168.1.10
```

## Putting it together — a realistic first scan
```
nmap -sS -sV -O 192.168.1.10
```
This single command: does a stealthy SYN scan, figures out what service is running on each open port, and guesses the operating system — a solid, standard starting point for any target.

## Why scanning matters, and why it's step one
Everything that comes after this in a network assessment depends on knowing what's actually there first. You can't test a login page for weak passwords if you don't know a login service even exists on that device. You can't check for an outdated file-sharing service if you don't know port 445 is open. Scanning is the map — everything else is deciding what to do once you're looking at that map.

## Ethical note
Scanning a network you don't own or don't have explicit written permission to test is illegal in most places, even if no actual "attack" happens — the scan itself is often enough to constitute unauthorized access under computer misuse laws. Only ever scan your own lab environment or something you have clear authorization for.
