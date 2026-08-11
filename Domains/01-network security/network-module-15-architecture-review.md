# Module 15 - Network Architecture Review

## Why this module is different from everything before it
Modules 1-14 were about actively attacking a network. This one is different in kind, not just topic — it's a **review/audit discipline**, looking at how a network is *designed* and asking "does this design itself create risk," independent of whether any specific exploit works today. This is the kind of finding that shows up in a governance-level assessment, not just a technical pentest report.

---

## DMZ (Demilitarized Zone) — the core architectural concept

**What it is:** A separate network segment, sitting between the fully untrusted internet and the fully trusted internal network, specifically for hosting anything that *must* be reachable from the outside world (a public website, a mail server, a VPN endpoint).

**Why it exists, and what problem it actually solves:** Without a DMZ, an internet-facing server sits directly on the same network as everything else — if that public-facing server is ever compromised, the attacker's very next step is trivial lateral movement straight into the sensitive internal network, since there was never any separation to begin with. A DMZ means that even if the public-facing server is fully compromised, the attacker is still only inside an isolated segment, with a **second** set of firewall rules standing between them and the actual internal network.

**What a proper DMZ review actually checks:**
- Is there a genuinely separate DMZ segment at all, or are "public-facing" servers actually just sitting on the internal network with a single firewall rule allowing inbound traffic (a common, dangerous shortcut)?
- What specific traffic is allowed **from the DMZ into the internal network**? This is the single most important rule to review — a DMZ provides zero protection if the firewall rules between DMZ and internal network are overly permissive (e.g. "DMZ can reach anything on the internal network on any port"), since that defeats the entire purpose of the separation.
- Is traffic **from the internal network into the DMZ** similarly restricted, not just DMZ-to-internal? A genuinely secure architecture restricts both directions, not just the "obvious" one.

---

## Network Segmentation — the broader principle the DMZ is one example of

**What it is:** Dividing a network into multiple smaller, isolated segments (via VLANs, subnets, or dedicated physical separation), each with its own access controls, rather than one large "flat" network where every device can potentially reach every other device.

**Why flat networks are a genuine architectural risk, not just theoretical:** connects directly back to the lateral movement content from Module 2 and Module 14 — a flat network means that compromising *any single device* (even a low-value one, like a random employee's workstation via phishing) potentially grants a path to *every other device* on the network, since there's no internal boundary limiting movement at all.

**What a segmentation review actually checks:**
- Are genuinely sensitive systems (databases holding customer financial data, domain controllers, payment processing systems) isolated on their own restricted segment, reachable only from specific, defined sources — or can they be reached from general user workstation subnets?
- Is guest/visitor Wi-Fi properly isolated from the internal corporate network, or does "Guest Wi-Fi" only mean a different SSID while still landing on the same underlying network?
- Are IoT/OT devices (cameras, badge readers, HVAC controllers, printers — devices that are frequently poorly patched or effectively unpatchable) segmented away from the main corporate network, given how commonly these specific device categories are targeted precisely because they're overlooked and under-secured compared to "real" computers?

---

## Firewall Ruleset Review — the actual line-by-line audit work

**Why this is a distinct, dedicated skill, not just "check the firewall exists":** A firewall being *present* says nothing about whether its actual rule configuration provides meaningful protection — a firewall with an overly permissive ruleset offers a false sense of security that can be worse than having no firewall at all, since it invites unwarranted confidence.

**What a proper ruleset review looks for, specifically:**

**Overly broad rules:** any rule using `ANY` as the source, destination, or port should be individually justified — a rule reading "allow ANY to ANY on ANY port" provides zero actual filtering while giving the appearance of a security control.

**Rule ordering issues:** firewall rules are typically evaluated top-to-bottom, with the first match winning — a broad "allow" rule placed *above* a more specific "deny" rule for the same traffic means the deny rule is effectively dead code, never actually reached, regardless of how correctly it was written.

**Unused/stale rules:** rules referencing decommissioned servers, old IP ranges no longer in use, or projects that ended long ago — these accumulate over time in any organization that doesn't periodically audit and clean up its ruleset, and every stale rule is additional, unreviewed attack surface that nobody is actively thinking about anymore.

**Default-deny vs default-allow posture:** the single most important high-level question — does the firewall's default behavior for anything not explicitly matched by a rule **deny** the traffic, or **allow** it? A default-deny posture (deny everything except what's explicitly permitted) is the standard, correct professional posture; default-allow means every single risk has to be individually and correctly anticipated in advance, which is a far weaker security model that fails open rather than fails closed.

**A practical way to test the *effective* ruleset, not just read the configuration file:**
```
nmap -sS -Pn <target-ip> -p-
```
Running a comprehensive scan against the actual live firewall/network, independent of reading the configuration on paper, verifies what's *actually* reachable in practice — configuration documentation and real-world behavior can drift apart over time (a rule added for a temporary reason and never removed, a misconfiguration during a later change) — the scan reveals ground truth, the documentation reveals intent, and a proper review checks both and compares them.

---

## Why this module matters for a portfolio specifically
Everything from Modules 1-14 demonstrates the ability to find and exploit specific technical flaws. This module demonstrates a different, complementary skill — the ability to evaluate whether a *design* creates risk, which is exactly the kind of judgment shown in the cloud misconfiguration lab report elsewhere in this repo (the same "is this the right default posture" reasoning applied to AWS IAM/S3 configuration, here applied to network architecture instead). Being able to speak to both — specific exploits *and* architectural review — is what separates a tool operator from someone who actually understands why the findings matter.

## Quick-reference table

| Review Area | Key question |
|---|---|
| DMZ | Are inbound AND outbound rules between DMZ and internal network both properly restricted? |
| Segmentation | Can a compromise of one low-value device reach genuinely sensitive systems? |
| Firewall Ruleset | Any `ANY`-to-`ANY` rules? Correct rule ordering? Stale rules? Default-deny or default-allow? |
| Verification | Does the actual live-scanned reachability match the documented configuration? |
