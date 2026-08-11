# Module 17 - Aircrack-ng Suite: Practical WPA/WPA2 Cracking

## Why this comes right after Wi-Fi fundamentals
Module 16 gave you the vocabulary — BSSID, channels, deauth frames, the four-way handshake concept from earlier wireless content. This module is the actual hands-on execution, tool by tool, command by command.

---

## Step 1: Enabling Monitor Mode

**Why this is required before anything else works:** a wireless adapter normally operates in "managed mode" — connected to one specific network, only processing traffic addressed to it. **Monitor mode** puts the adapter into a state where it captures *all* wireless traffic within range, on whatever channel it's tuned to, regardless of whether that traffic is addressed to your device at all — this is a fundamental prerequisite for any of the capture-based techniques in this module.

```
airmon-ng check kill
```
Kills background processes (NetworkManager, wpa_supplicant) that commonly interfere with monitor mode by trying to manage the wireless adapter simultaneously.

```
airmon-ng start wlan0
```
Puts the adapter into monitor mode, typically creating a new interface named something like `wlan0mon`.

```
iwconfig
```
Verify the new monitor-mode interface exists and confirm its exact name before proceeding.

---

## Step 2: Discovering Networks and Targeting One

```
airodump-ng wlan0mon
```
Lists every visible network in range — BSSID, channel, encryption type, and SSID, exactly as introduced in Module 16. Identify your specific target's BSSID and channel from this output before continuing.

**Locking onto the specific target, and saving the capture to a file:**
```
airodump-ng --bssid <target-bssid> --channel <target-channel> -w capture wlan0mon
```
- `--bssid` and `--channel` — restricts capture to only this one specific access point, on its actual channel (connects directly to why channel-locking matters, from Module 16)
- `-w capture` — writes the captured traffic to files with the prefix `capture` (e.g. `capture-01.cap`), so it can be reused for cracking after the live capture session ends

**Leave this running** — this is the window that will show you when the actual handshake gets captured, and it must remain active throughout the next step.

---

## Step 3: Capturing the Four-Way Handshake

**The passive option — simply waiting:** if you leave `airodump-ng` running long enough, a legitimate device will eventually connect (or reconnect) to the target network on its own, and the handshake will be captured automatically, with zero further action needed. **The problem with waiting:** this could take an unpredictable, potentially very long amount of time, which is exactly why the active option below is used in almost every real/practical scenario.

**The active option — forcing it with a deauthentication attack (the real mechanism behind Module 16's explanation of why deauth frames work):**
```
aireplay-ng --deauth 10 -a <target-bssid> wlan0mon
```
- `--deauth 10` — send 10 forged deauthentication frames
- `-a <target-bssid>` — specifies which access point to impersonate when sending these frames

**Why this actually works, tying directly back to Module 16's frame-type explanation:** because deauthentication frames aren't authenticated on WPA2, this forged frame is indistinguishable to a connected device from a real disconnect instruction from the actual access point. The targeted device disconnects, and — because it still wants to be connected — immediately attempts to reconnect, which means it performs a brand-new four-way handshake, which your still-running `airodump-ng` capture window catches in real time. You'll see a **"WPA handshake: <bssid>"** notification appear in the capture window the moment this succeeds.

---

## Step 4: Cracking the Captured Handshake Offline

**Why this step is entirely offline, and why that matters:** once you have the `.cap` file containing the handshake, no further interaction with the target network is required at all — this connects directly back to the online-vs-offline distinction from the password cracking module. Everything from here happens entirely on your own hardware.

**Using Aircrack-ng itself, with a wordlist:**
```
aircrack-ng capture-01.cap -w /usr/share/wordlists/rockyou.txt
```
Aircrack-ng takes each candidate password from the wordlist, performs the same cryptographic computation the real handshake process uses, and checks whether the result matches what was captured — a match confirms that candidate is the actual network password.

**Using Hashcat instead, for GPU-accelerated speed (connects directly to the Hashcat content from the password cracking module):**

First, convert the capture into a format Hashcat understands:
```
hcxpcapngtool -o hash.hc22000 capture-01.cap
```
Then crack it:
```
hashcat -m 22000 hash.hc22000 /usr/share/wordlists/rockyou.txt
```
`-m 22000` is the specific Hashcat mode number for WPA/WPA2 handshakes — the exact same "you must know the correct mode number" principle already emphasized in the password cracking module, just a new mode value for this new hash type.

**Why Hashcat is frequently the better real choice here specifically:** WPA2 handshake cracking is computationally expensive per attempt (deliberately, by design of the protocol, to slow down exactly this kind of attack) — GPU acceleration provides a genuinely significant, practical speed advantage over CPU-based cracking for this specific workload, more so than for many other, lighter hash types.

---

## PMKID Attack — cracking WPA2 without needing a client device or a handshake at all

**Why this is a fundamentally different, often better technique:** every step above required a connected client device to deauth and force a reconnection. The PMKID attack targets a different piece of data — the **PMKID**, which some access points include in the very first frame of the authentication process, before any client device is involved at all. This means the attack can be performed against a completely empty network, with zero connected clients, something the handshake-capture method simply cannot do.

**Capturing the PMKID:**
```
hcxdumptool -o pmkid.pcapng -i wlan0mon --enable_status=1
```
**Converting and cracking, same tools as before:**
```
hcxpcapngtool -o pmkid_hash.hc22000 pmkid.pcapng
hashcat -m 22000 pmkid_hash.hc22000 /usr/share/wordlists/rockyou.txt
```

**Why this matters as a real technique, not just a curiosity:** it removes the dependency on a connected, active client device entirely, and — since no deauthentication frames need to be sent at all — it's also meaningfully stealthier, since there's no disruptive, detectable deauth attack happening on the network at any point.

## Quick-reference table

| Step | Tool/Command | Purpose |
|---|---|---|
| Enable monitor mode | `airmon-ng start wlan0` | Prerequisite for capturing all nearby traffic |
| Discover/target | `airodump-ng` (with `--bssid`/`--channel`) | Identify and lock onto the specific target |
| Force a handshake | `aireplay-ng --deauth` | Exploits unauthenticated deauth frames (Module 16) |
| Crack offline (CPU) | `aircrack-ng` + wordlist | Standard offline dictionary attack |
| Crack offline (GPU) | `hashcat -m 22000` | Faster, GPU-accelerated cracking |
| Client-less alternative | `hcxdumptool` (PMKID) | Works without any connected client device, stealthier |
