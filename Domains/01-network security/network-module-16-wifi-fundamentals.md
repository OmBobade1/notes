# Module 16 - Wi-Fi Fundamentals: 802.11, BSSID, SSID & Frame Types

## Why this comes right after architecture review
Everything so far has assumed a wired or already-connected network. Wi-Fi introduces its own physical layer, its own terminology, and its own attack surface entirely — this module builds the vocabulary from scratch before any actual attacking happens in Modules 17-18, exactly the small-things-first approach you asked for.

---

## 802.11 — what this actually refers to
**802.11** is the IEEE standard that defines how wireless local area networking (Wi-Fi) works at the physical and data-link layers. Different letter suffixes (802.11a/b/g/n/ac/ax) represent different generations, each improving speed, range, or efficiency — "Wi-Fi 6" as a marketing name corresponds to 802.11ax specifically. For security purposes, what matters far more than the letter suffix is the **encryption standard** running on top (WEP/WPA/WPA2/WPA3, covered in Module 17-18), since 802.11 itself defines how devices communicate, not how that communication gets secured.

---

## SSID (Service Set Identifier) — the network's name

**What it is:** The human-readable name of a wireless network — "Home_WiFi_5G", "CompanyGuest" — the name you actually see and select on your device.

**Why it matters for security, specifically:** an SSID is just a label, chosen by whoever set up the network — it provides **zero security on its own**. A network can hide its SSID from being broadcast (a "hidden network"), but this is a very weak, easily-defeated protection (covered below), not genuine security.

---

## BSSID (Basic Service Set Identifier) — the access point's actual hardware identity

**What it is:** The **MAC address of the specific physical access point (or specific radio)** broadcasting a given wireless network — a unique hardware identifier, fundamentally different from the SSID.

**Why this distinction actually matters, and where it trips people up:** Multiple access points (e.g. several Wi-Fi extenders throughout a large office, all configured to broadcast the same network name for seamless roaming) can all share the **same SSID** — "CompanyWiFi" — while each individual access point has its **own unique BSSID**. This means SSID tells you *which network*, while BSSID tells you *which specific physical device* you're actually talking to.

**Why an attacker specifically cares about BSSID, not just SSID:** every attack in Modules 17-18 (deauthentication, handshake capture) is targeted at one specific access point, identified by its BSSID — you can't meaningfully attack "the SSID," since that might represent five different physical devices; you attack one specific BSSID at a time.

**Seeing both together, in practice:**
```
airodump-ng wlan0mon
```
This tool's output lists every detected network with both fields shown side by side — BSSID (the actual hardware MAC address) and ESSID (SSID) in the same row, making the distinction between "this specific device" and "this network name" immediately visible in real output, not just in theory.

---

## ESSID vs BSSID vs SSID — clearing up one more common confusion
Strictly: **SSID** is the general term for a network name. **BSSID** identifies one specific access point. **ESSID** (Extended SSID) is the network name specifically in the context of *multiple* access points sharing one logical network (the roaming-extenders scenario above) — in casual/practical use, most tools and most people just say "SSID" to mean the network name regardless of which of these precise terms technically applies, but knowing the precise distinction matters when reading tool output that specifically separates them.

---

## Channels and Frequency Bands

**What a channel is:** Wi-Fi operates within specific frequency bands (commonly 2.4GHz and 5GHz), and each band is further divided into numbered **channels** — different frequency slices multiple networks can operate on simultaneously without interfering with each other, similar to different radio stations occupying different frequencies.

**Why this matters for actually attacking anything:** your wireless adapter, when capturing traffic, can typically only listen to **one channel at a time**. Before any of the attacks in Module 17 work, you need to know which specific channel your target access point is actually broadcasting on, and lock your capture tool onto that exact channel — otherwise you're listening to the wrong slice of spectrum entirely and will capture nothing relevant.
```
airodump-ng --channel 6 --bssid <target-bssid> wlan0mon
```

---

## Frame Types — the actual messages devices exchange

Wi-Fi communication happens through distinct categories of frames, each serving a different purpose — worth knowing individually since specific attacks target specific frame types.

**Management Frames** — handle the connection process itself: **Beacon frames** (an access point continuously broadcasting "I exist, here's my SSID and capabilities," roughly ten times per second by default — this is literally how your device's Wi-Fi list populates with visible networks), **Probe Request/Response frames** (a device actively asking "is a specific network nearby?" and the access point replying if so), **Authentication** and **Association frames** (the actual handshake process of a device joining a network), and critically, **Deauthentication frames** — a frame whose entire legitimate purpose is telling a device "you are now disconnected." This last one is exactly what gets abused in the deauthentication attack technique from the earlier wireless-attacks content, now understood at the actual frame-type level rather than as an abstract concept.

**Control Frames** — manage the flow of data transmission itself (acknowledgment frames confirming a packet was received correctly, request-to-send/clear-to-send frames coordinating who's allowed to transmit next on a shared channel).

**Data Frames** — the actual payload — the real data (web traffic, emails, whatever the user is actually doing) being carried over the established, authenticated connection.

**Why this categorization matters, specifically for the deauthentication attack covered next module:** Management frames — including deauthentication frames — are, on WPA2 and earlier, **not encrypted or authenticated the same way data frames are**. This is precisely why a deauthentication attack works at all: an attacker doesn't need to know the network's password to send a forged deauthentication frame, since the network was never designed to verify that management frames genuinely came from the legitimate access point in the first place. (This specific weakness is one of the things **Protected Management Frames**, part of WPA3, was built to directly address — covered in Module 18.)

---

## Hidden Networks — why hiding the SSID isn't real security

**What "hidden" actually means technically:** the access point simply stops broadcasting its SSID inside beacon frames — beacon frames are still sent constantly (the network still announces its *existence*, just without the name attached).

**Why this is trivially defeated, not genuine protection:** the moment any legitimate device that already knows the network tries to connect, it sends probe requests that **do** contain the actual SSID in plaintext — passively capturing traffic for even a short time (using the same `airodump-ng` capture from earlier) typically reveals the "hidden" name the first time any authorized device attempts to reconnect. Hiding the SSID inconveniences casual browsing far more than it inconveniences an actual attacker running a capture tool.

## Quick-reference table

| Term | What it actually is |
|---|---|
| SSID | The network's human-readable name |
| BSSID | The specific access point's MAC address (hardware identity) |
| ESSID | The network name in a multi-access-point/roaming context |
| Channel | The specific frequency slice a network operates on |
| Beacon Frame | Continuous "I exist" broadcast from an access point |
| Deauthentication Frame | Unencrypted/unauthenticated "disconnect" message — the basis of the deauth attack |
| Hidden Network | SSID omitted from beacons, but still trivially revealed via probe requests |
