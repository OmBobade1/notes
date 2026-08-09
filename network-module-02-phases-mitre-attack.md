# Module 2 - Phases of Hacking & the MITRE ATT&CK Framework

## Why this comes right after Network Devices
Module 1 covered *what* exists on a network. This module covers the *order of operations* an attacker actually follows once they start targeting it — every technique in every later module fits into one of these phases. Without this framework, individual techniques (a scan here, an exploit there) look like a random toolbox instead of a structured process.

---

## The Seven Phases, in order

### 1. Reconnaissance (Information Gathering)
**What it is:** Gathering everything possible about the target before touching it directly.

**What it's for, attacker's perspective:** The more you know before you attack, the fewer wasted attempts, and the less noise you generate — a well-recon'd attack looks like one precise action; a poorly-recon'd one looks like flailing, which gets noticed faster.

**Two sub-types, and the real difference:**

- **Passive Reconnaissance** — gathering information **without ever directly interacting with the target's systems**. Examples: reading a company's public website, checking employee LinkedIn profiles, searching WHOIS records for a domain, reading job postings that reveal what technology stack they use (a job listing for "Django developer" tells you their backend is Python/Django).
  ```
  whois target-company.com
  ```
  **Why attackers prefer starting here:** none of this touches the target's actual systems, so there's nothing for their logs or IDS to detect. You're invisible to them at this stage.

- **Active Reconnaissance** — actually interacting with target systems to gather information (this is where scanning, covered in Module 3, technically begins). The trade-off: more accurate, current information, but now you're generating traffic the target *can* potentially detect and log.

**Mitigation (defender's side):** Limit what's publicly exposed — job postings that overshare tech stack details, employee social media oversharing project details, WHOIS privacy protection on domain registration. None of this is a technical control; it's information discipline.

### 2. Scanning
**What it is:** Actively identifying live hosts, open ports, and running services — covered in full depth in Module 3.

### 3. Gaining Access (Gaining a Foothold)
**What it is:** Actually exploiting a discovered vulnerability or misconfiguration to get *some* level of unauthorized access — even a low-privilege one.

**What it's for, attacker's perspective:** This is the actual "break-in" moment — everything before this was preparation, everything after this is what you do with the access you now have.

### 4. Maintaining Access (Persistence)
**What it is:** Making sure the access gained in phase 3 survives — a reboot, a patch, the original vulnerability being fixed.

**How it's beneficial to the attacker:** Without persistence, every bit of access is one system restart away from being lost, forcing the attacker to repeat the entire break-in process. Backdoors, scheduled tasks, and rootkits (covered later) all exist specifically to solve this problem.

### 5. Lateral Movement
**What it is:** Once inside one system, actively moving to *other* systems on the same network, expanding the compromise beyond the original foothold.

**What it's for, attacker's perspective:** The first system compromised is rarely the actual target — it's often just the easiest way in. Lateral movement is how an attacker gets from "I have access to one low-value machine" to "I have access to the domain controller/finance database/whatever was actually valuable."

### 6. Covering Tracks
**What it is:** Removing or altering evidence of everything done in the previous phases — deleting logs, altering timestamps, hiding files/processes.

**Why an attacker bothers:** The longer an intrusion goes undetected, the more damage can be done and the harder attribution becomes — this directly connects to the insufficient logging & monitoring content in the web security series; an attacker actively wants to become the exact kind of invisible activity that weak logging fails to catch.

### 7. Reporting
**What it is:** This phase is specific to **ethical hackers/penetration testers**, not malicious attackers — documenting everything found, its impact, and remediation recommendations, delivered to the organization that authorized the test.

**Why this phase matters just as much as the technical ones:** A perfect technical compromise that never gets written up clearly is worthless to the client — this is exactly why the cloud lab report and the web vulnerability files in this repo spend as much effort on "business impact" and "how to explain this to a developer" as they do on the technical exploit itself.

---

## The MITRE ATT&CK Framework

**What it is:** A publicly maintained, structured knowledge base (built by the MITRE Corporation) that catalogs real-world attacker **Tactics, Techniques, and Procedures (TTPs)** — essentially a massive, standardized reference of "here's every known way attackers actually do things," organized so security professionals worldwide can speak the same language about threats.

**Why it exists, and why it's genuinely useful (not just a reference chart):** Before ATT&CK, different security teams and vendors described the same attacker behavior in inconsistent, incompatible ways. ATT&CK gives every technique a standard ID (e.g. `T1078` for "Valid Accounts") that means the exact same thing no matter who's using it — a shared vocabulary the same way CVE IDs (Module 4/vulnerability assessment content) standardize vulnerability references.

**How it's structured:** Organized into **matrices** for different environments — Enterprise (traditional corporate networks), Mobile, and Cloud — since attacker techniques genuinely differ across these environments. Within each matrix, techniques are organized under **Tactics** (the *why* — e.g. "Privilege Escalation," "Persistence," "Exfiltration") and each Tactic contains specific **Techniques** (the *how* — e.g. under Persistence, `T1053` is "Scheduled Task/Job").

**How this connects to the seven phases above:** ATT&CK's tactics map closely onto the phases already covered — "Initial Access" maps to gaining a foothold, "Persistence" maps to maintaining access, "Exfiltration" maps to actually stealing data during lateral movement. The seven-phase model is the simple mental map; ATT&CK is the detailed, real-world-validated reference for exactly *how* each phase gets carried out in practice.

**Practical use for a tester:** Instead of writing a report finding as "attacker could escalate privileges," referencing the specific ATT&CK technique ID (e.g. `T1548` - Abuse Elevation Control Mechanism) gives the finding a precise, industry-standard reference that any other security professional can immediately look up and understand exactly what's being described — this is the same precision-through-standardization idea as citing a CVE number instead of just describing a vulnerability in your own words.

## Quick-reference table

| Phase | One-line purpose |
|---|---|
| Reconnaissance | Gather information before touching the target |
| Scanning | Actively identify live hosts, ports, services |
| Gaining Access | Exploit a vulnerability to get a foothold |
| Maintaining Access | Ensure the foothold survives reboots/patches |
| Lateral Movement | Expand from one system to others on the network |
| Covering Tracks | Hide evidence of the intrusion |
| Reporting | (Ethical testers only) Document findings and impact |
