# 08 - Denial of Service / DDoS (Explained Simply)

## Why this comes right after social engineering
Everything so far — scanning, exploiting, malware, social engineering — is about *getting in* or *getting information*. Denial of Service is different: the goal isn't access at all, it's making a service **stop working** for legitimate users. This connects directly back to the Application-Layer DoS file in the web security series — that file covered clever single-request attacks; this one covers the broader, more brute-force network-level version.

---

## The core idea, in one sentence
Denial of Service is **overwhelming a target with more than it can handle, so it can't serve real, legitimate requests anymore** — like so many people crowding into a small shop's doorway that actual paying customers can't get in at all.

## DoS vs DDoS — the one-letter difference that matters
**DoS (Denial of Service)** — the attack comes from **one** source (one machine, one internet connection).

**DDoS (Distributed Denial of Service)** — the attack comes from **many** sources simultaneously, usually thousands of compromised devices (a "botnet") all attacking the same target at once. This is far harder to defend against, because you can't just block one troublemaker's IP address — the traffic is coming from thousands of different, often legitimate-looking addresses all at once, many of which are just innocent people's infected devices being used without their knowledge.

## What is a botnet?
A **botnet** is a network of compromised devices (computers, routers, even IoT devices like smart cameras) that have been infected with malware giving an attacker remote control over them — without their real owners knowing anything's wrong. The attacker can then command all of them simultaneously to send traffic at one target, turning thousands of innocent people's devices into an army for a DDoS attack.

## Common attack types, from simple to sophisticated

**Volumetric Attacks (flooding)** — the simplest concept: just send an overwhelming *volume* of traffic at the target's network connection, so much that legitimate traffic simply can't get through, the same way a highway becomes unusable at extreme traffic volume regardless of whether any individual car is doing anything wrong.

**SYN Flood** — exploits how TCP connections are normally established (a three-step handshake). The attacker sends a flood of connection *start* requests (SYN) but never completes the handshake, leaving the target holding open a huge number of half-finished connections, exhausting its capacity to accept any new, real ones.

**Amplification Attacks** — a clever trick where the attacker sends a small request to a third-party server, but spoofs the source address to make it look like the *victim* sent it — the third-party server then sends a much *larger* response directly to the victim. This means the attacker only needs a small amount of their own bandwidth to generate a much larger flood at the target, since the "amplification" happens at the innocent third-party server. DNS and NTP servers have historically been common tools for this specific technique.

**Application-Layer DoS** — instead of brute-force flooding, targeting a specific expensive operation the application itself performs (this is exactly what the ReDoS and resource-exhaustion content in the web security series covers) — fewer requests needed, since each one is deliberately expensive for the server to process.

## Why DDoS is hard to defend against
- Traffic often comes from thousands of different IP addresses simultaneously — can't just block "the attacker's IP."
- Malicious traffic can look very similar to a genuine, sudden spike in legitimate interest (a product going viral, a news event) — distinguishing the two in real time is genuinely difficult.
- Amplification attacks mean a relatively small attacker can generate a disproportionately massive amount of traffic.

## Defensive approaches (at a conceptual level)
**Rate limiting** — capping how many requests a single source can make in a given time window (connects directly to the rate-limiting content already covered in the web security and authentication files).

**Traffic scrubbing services** — specialized third-party services that sit in front of a target, absorbing and filtering out malicious traffic before it ever reaches the real infrastructure, only passing legitimate-looking traffic through.

**Content Delivery Networks (CDNs)** — distributing traffic across many geographically spread servers, so a flood aimed at "the server" actually has to overwhelm many servers spread globally, not just one central point.

**Over-provisioning** — simply having more capacity than a typical attack would require to exhaust, though this is an expensive, not fully reliable defense on its own against a sufficiently large, well-resourced attack.

## Quick-reference table

| Term | Plain meaning |
|---|---|
| DoS | One source overwhelming a target |
| DDoS | Many distributed sources overwhelming a target at once |
| Botnet | A network of secretly compromised devices under an attacker's control |
| SYN Flood | Exhausting a server's capacity via incomplete connection requests |
| Amplification Attack | Using a third-party server to multiply a small request into a large flood |
