# Network Security — Index

\# Network Security Labs



Write-ups from network and system exploitation wargames — starting with OverTheWire: Bandit, a level-by-level Linux/networking fundamentals wargame.



| Section | Covers |

|---|---|

| \[Bandit](./bandit/) | OverTheWire Bandit — 30+ levels of privilege escalation, file discovery, and Linux fundamentals |



\## Methodology

Each level gets its own short write-up: the challenge, the technique used to find the next password, and the command(s) that solved it. Consistency matters more than length here — recruiters skim these fast.



19 modules (`00` through `18`), built from basics to advanced, each with real commands, the attacker's actual reasoning, and mitigation — not just concept summaries. Every module explicitly connects back to specific earlier modules where the underlying idea first showed up, so nothing sits in isolation.

\---

## 🧭 What to Reference, Based on What You're Doing

|If you're...|Reference these modules|
|-|-|
|**Confused about OSI layers, TCP/UDP, or ports referenced elsewhere**|`00`|
|**Trying to understand a specific device's role/weakness (router, switch, hub)**|`01`|
|**Planning an engagement and need the overall attack structure**|`02`|
|**Gathering info before touching the target at all (OSINT, DNS, Google dorking)**|`03`|
|**Need raw TCP/UDP interaction, a quick port check, or a reverse/bind shell**|`04`|
|**Scanning a target — which scan type, and why**|`05`|
|**Need deeper automated checks (NSE) or the scan itself is being blocked/detected**|`06`|
|**Found an open port and need to actually enumerate that specific service**|`07` (FTP/HTTP/SMTP/SNMP)|
|**Need to hand-craft packets, or test/perform a DoS scenario in a lab**|`08`|
|**Have credentials to guess, or a hash to crack**|`09`|
|**Need to intercept/alter live traffic, or understand replay attack risk**|`10`|
|**Found a vulnerable service that needs a custom exploit, or need to reach an internal-only system**|`11`|
|**Standard evasion (Module 6) isn't enough — need a full covert channel through the firewall, or facing a WAF**|`12`|
|**Ready to actually exploit something with a known Metasploit module**|`13`|
|**Already have a session and need privesc, credentials, lateral movement, or persistence**|`14`|
|**Reviewing network design/segmentation/firewall rules rather than actively exploiting**|`15`|
|**New to Wi-Fi testing — need the vocabulary first (BSSID/SSID/channels/frames)**|`16`|
|**Ready to actually capture and crack a WPA/WPA2 handshake**|`17`|
|**Target is WPA3, or need to know what's actually different/still risky about it**|`18`|

## 🧭 A Practical Engagement Order

1. `00`-`01` — make sure the fundamentals and device landscape are solid before touching the target.
2. `02` — plan the engagement around the phases; know what "done" looks like.
3. `03` — passive/OSINT recon first, zero footprint on target systems.
4. `05`-`07` — scan, then enumerate every open service found, one at a time.
5. `09` — attempt credential attacks on whatever services allow login.
6. `11`, `13`-`14` — exploit, gain a foothold, escalate, move laterally, persist.
7. `10`, `08` — as needed, intercept traffic or craft custom packets for specific tests.
8. `06`, `12` — apply evasion techniques throughout, wherever detection/blocking becomes a factor.
9. `15` — a separate, architecture-level review pass, independent of active exploitation.
10. `16`-`18` — Wi-Fi as its own, self-contained track, run whenever wireless is in scope.

\---

## 📖 Full Module Index

|#|File|Covers|
|-|-|-|
|00|`network-module-00-osi-tcpip.md`|OSI 7 layers, TCP three-way handshake, UDP, ports|
|01|`network-module-01-devices.md`|Router, switch, hub, bridge — what each is, attacker angle, mitigation|
|02|`network-module-02-phases-mitre-attack.md`|The 7 phases of hacking, MITRE ATT\&CK framework|
|03|`network-module-03-reconnaissance.md`|OSINT, DNS record types, Google Hacking/GHDB, Recon-ng|
|04|`network-module-04-netcat.md`|Netcat/Ncat commands, port scanning, file transfer, bind vs reverse shells|
|05|`network-module-05-nmap-scans.md`|Every Nmap scan type with packet-level mechanics|
|06|`network-module-06-nse-evasion.md`|NSE scripts, Zenmap, Nmap-level firewall/IDS evasion|
|07|`network-module-07-service-enumeration.md`|FTP, HTTP, SMTP, SNMP — real per-service commands|
|08|`network-module-08-hping3.md`|Manual packet crafting, port scanning, SYN flood, Land, Smurf|
|09|`network-module-09-password-cracking.md`|Online vs offline, Hydra, John the Ripper, Hashcat|
|10|`network-module-10-mitm-arp-impersonation-replay.md`|Ettercap, Bettercap, DHCP spoofing, replay attacks|
|11|`network-module-11-bufferoverflow-pivoting.md`|Full buffer overflow methodology, Metasploit/SSH pivoting|
|12|`network-module-12-firewall-evasion-tunneling.md`|DNS/ICMP tunneling, port knocking, WAF bypass|
|13|`network-module-13-metasploit-fundamentals.md`|Module types, full workflow, sessions, Meterpreter|
|14|`network-module-14-metasploit-post-exploitation.md`|Privesc suggester, hashdump/Mimikatz, Pass-the-Hash, persistence|
|15|`network-module-15-architecture-review.md`|DMZ, segmentation, firewall ruleset audit|
|16|`network-module-16-wifi-fundamentals.md`|802.11, BSSID vs SSID vs ESSID, channels, frame types|
|17|`network-module-17-aircrack-wpa-cracking.md`|Monitor mode, handshake capture, deauth, offline cracking, PMKID|
|18|`network-module-18-wpa3-dragonblood.md`|SAE/PMF, and the real Dragonblood vulnerabilities|

## How to use this

Each module is self-contained but built to be read in order — later modules assume earlier ones as known context and explicitly reference them by number. Every module includes real commands, the attacker's specific reasoning (not just "what," but "why" and "when"), and mitigation.

