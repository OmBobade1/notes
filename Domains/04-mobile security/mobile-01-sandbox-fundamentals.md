# Mobile Security Fundamentals: The Platform Sandbox Model

## Why this comes first
Before testing any specific mobile vulnerability, understanding how Android and iOS isolate apps from each other and from the OS itself — the "sandbox" — explains why almost every mobile vulnerability is really about an app breaking out of, or misusing, that isolation.

---

## What it is (in plain terms)
Both Android and iOS run every app in its own isolated sandbox — a restricted environment where, by default, an app can only access its own files and data, not another app's. This is fundamentally different from a traditional desktop OS, where programs historically had much broader access to the whole filesystem. Every mobile vulnerability category that follows in this series is really a specific way this sandbox gets bypassed, weakened, or simply never used correctly by the developer in the first place.

## The two platforms, briefly

**Android:**
- Each app runs under its own unique Linux user ID — one app literally cannot read another app's files at the OS level, enforced by standard Linux file permissions.
- Apps request **permissions** (camera, location, contacts) that the user must grant, either at install time (older Android) or at first use (modern Android, "runtime permissions").
- Apps are distributed as `.apk` files, which can be installed from the Play Store or, if the user allows it, from any other source ("sideloading") — a meaningfully larger attack surface than iOS.

**iOS:**
- Enforces sandboxing at a stricter OS level, and historically only allows app installation through the App Store (excluding developer/enterprise provisioning), giving Apple a single review checkpoint before any app reaches a user.
- Uses a similar permission-prompt model for sensitive capabilities (camera, location, contacts).
- Generally considered to have a smaller sideloading-related attack surface than Android, precisely because of this distribution control.

## Where the sandbox commonly breaks down in practice
Every file that follows in this series is really one of these patterns:
- **The app itself stores data insecurely** *within* its own sandbox (e.g. plaintext credentials in local storage) — the sandbox correctly kept other apps out, but the data was never protected from anyone who gets access to the device itself (physical access, malware with root/jailbreak access, a backup extraction).
- **The app requests more permissions than it needs**, and a user grants them, expanding what the app itself could misuse or leak if compromised.
- **The app talks to a server insecurely**, and the sandbox has nothing to do with protecting that channel at all — that's a transport-security problem (covered in file `02`).
- **The device itself is rooted/jailbroken**, which deliberately weakens or removes the sandboxing protections the OS is designed to enforce — an app has to explicitly detect and respond to this if it wants to maintain trust in its own environment.

## Business Impact of misunderstanding the sandbox model

| Angle | What it actually means |
|---|---|
| **Financial loss** | Assuming "the OS sandbox protects my data" is a common developer misconception — the sandbox protects data from *other apps*, not from insecure storage practices within your own app, or from a rooted device where sandbox enforcement itself is compromised |
| **Regulatory / compliance** | Mobile banking apps are frequently required (by regulators and payment card industry standards) to implement root/jailbreak detection and additional data-at-rest protection specifically *because* the OS sandbox alone isn't considered sufficient protection for financial data |
| **Reputational damage** | "Our app followed platform defaults" is a weak defense if platform defaults were never enough for the sensitivity of the data being handled — banking apps are held to a materially higher bar than the average consumer app |
| **Legal liability** | Failing to implement protections beyond the baseline sandbox (encryption at rest, root detection, certificate pinning) for financial data is increasingly treated as a due-diligence failure, not just a missed nice-to-have |
| **Operational cost** | Retrofitting proper data protection after a mobile app is already in production and widely installed is significantly more disruptive than building it in from the first release |

**One-line interview answer:** *"The sandbox model means Android and iOS isolate every app's data from every other app by default — but that's a ceiling on what other apps can reach, not a guarantee that your own app's data is actually protected. For a banking app specifically, regulators expect protections that go beyond the OS default — encryption at rest, root/jailbreak detection, certificate pinning — precisely because the baseline sandbox was never designed with that threat model in mind."*

## How this connects to the rest of this repo
Every file that follows — insecure local storage (`02`), insecure communication (`03`), reverse engineering/hardcoded secrets (`04`), root/jailbreak detection (`05`) — is a deeper look at one specific way an app either misuses the sandbox it's given, or fails to add protection the sandbox was never meant to provide in the first place.
