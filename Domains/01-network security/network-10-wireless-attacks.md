# 10 - Wireless Network Attacks (Explained Simply)

## Why this comes right after firewalls/IDS/IPS
Everything before this assumed a wired-style connection. Wi-Fi is fundamentally different in one important way: **the "cable" is just air** — anyone within radio range can technically receive the signal, whether they're authorized to or not. This is why wireless security has its own distinct set of attacks, separate from everything covered so far.

---

## The core idea, in one sentence
Since Wi-Fi signals travel through open air rather than a physical wire, wireless security depends entirely on encryption and authentication doing the job that a physical cable used to do for free — and every wireless attack is really about attacking that encryption or authentication layer.

## A quick history of Wi-Fi security (why this matters)
**WEP (Wired Equivalent Privacy)** — the original Wi-Fi encryption standard, now considered completely broken. It has fundamental cryptographic flaws that let an attacker recover the password within minutes using freely available tools, regardless of how strong or long the password is. If you ever see a network still using WEP, it's not "weak" — it's functionally the same as having no encryption at all.

**WPA (Wi-Fi Protected Access)** — an emergency fix rolled out after WEP was broken, meant as a stopgap. Better than WEP, but still has known weaknesses of its own.

**WPA2** — the standard for most of the last decade, and still widely deployed today. Significantly stronger than WEP/WPA, but not immune to attack, as covered below.

**WPA3** — the current standard, addressing several of WPA2's specific weaknesses (particularly around offline password-guessing attacks). Adoption is still ongoing, meaning WPA2 remains extremely common in real assessments.

## The Four-Way Handshake — what actually happens when you connect to Wi-Fi
When a device connects to a WPA2 network, it and the router perform a **four-way handshake** — an exchange of messages that proves both sides know the network password, without ever actually transmitting the password itself in plain text. This handshake is the specific thing most WPA2 attacks target.

## Common attack types

**Handshake Capture + Offline Cracking** — an attacker captures the four-way handshake (by passively waiting for a device to connect, or by forcing a reconnection — see deauthentication below), then takes that captured data away and attempts to crack the password *offline*, on their own hardware, with no further interaction with the target network needed. This connects directly to the password-attack concepts from file `05` — dictionary and brute-force attacks, just applied to a captured Wi-Fi handshake instead of an online login form.

**Deauthentication Attack** — sending forged deauthentication packets that trick a connected device into thinking it's been disconnected from the network, forcing it to reconnect. Why an attacker wants this: reconnection means a fresh four-way handshake happens, which the attacker can then capture — this is often used specifically to *force* a handshake to occur immediately, rather than waiting for one to happen naturally.

**Evil Twin Attack** — setting up a rogue access point using the *exact same network name* (SSID) as a legitimate one, tricking devices into connecting to the attacker's fake network instead of the real one. Once connected, the attacker sits in the middle of all the victim's traffic — this is the wireless-specific version of the Man-in-the-Middle concept from file `03`.

**WPS (Wi-Fi Protected Setup) Attacks** — WPS was designed as a convenience feature (push a button or enter a short PIN to connect, instead of typing the full password). The PIN-based version has a well-known design flaw that allows the PIN to be brute-forced in a practical amount of time, which then reveals the actual Wi-Fi password — a completely separate weakness from the password strength itself, since it bypasses the password entirely.

**KRACK (Key Reinstallation Attack)** — a more advanced attack targeting a flaw in how WPA2 itself handles part of the four-way handshake process, potentially allowing an attacker to intercept traffic even on a network with a genuinely strong password — notable specifically because it showed that even correctly-configured WPA2 had a fundamental protocol-level weakness, not just a "your password was too weak" issue.

## Why a strong password alone isn't a complete defense
Several of the attacks above (Evil Twin, KRACK, WPS flaws) don't actually depend on the password being weak at all — they exploit the protocol or the human tendency to trust a familiar-looking network name, not a guessable password. This is the same "defense in depth" lesson repeated throughout this whole series: one strong control (a good password) doesn't cover every possible attack path.

## Defensive takeaways
- Use WPA3 where available; WPA2 with a genuinely long, random passphrase (not a dictionary word) as the realistic minimum today.
- Disable WPS entirely, given its known PIN-brute-force weakness.
- Be cautious of networks with familiar names in public places — always worth double-checking with venue staff whether "Free_Airport_WiFi" is actually theirs.
- Enterprise networks should use **WPA2/WPA3-Enterprise** (individual per-user credentials via a RADIUS server) rather than one shared password for everyone — this also means a departing employee's access can be individually revoked, instead of requiring the whole company's Wi-Fi password to be changed.

## Quick-reference table

| Attack | What it actually targets |
|---|---|
| Handshake Capture + Cracking | The password itself, via offline guessing |
| Deauthentication Attack | Forces a fresh handshake to capture |
| Evil Twin | Tricks victims into connecting to a fake network |
| WPS PIN Attack | The convenience PIN feature, bypassing the real password |
| KRACK | A protocol-level flaw in WPA2 itself |
