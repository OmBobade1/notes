# Module 18 - WPA3 & Dragonblood

## Why this closes the network sequence
Module 17 showed exactly how WPA2 gets broken in practice. This final module covers what WPA3 was actually built to fix, how it fixes it technically, and — being honest rather than presenting WPA3 as a solved problem — the real vulnerabilities discovered in early implementations.

---

## What WPA2 Actually Got Wrong (the specific problems WPA3 targets)

Directly connecting back to Module 17's practical attack: WPA2's four-way handshake is vulnerable to **offline dictionary attacks** once captured — an attacker captures the handshake once, then can attempt unlimited password guesses on their own hardware, with the target having zero visibility into how many attempts are happening. This single weakness is the entire basis of everything demonstrated in Module 17.

---

## SAE (Simultaneous Authentication of Equals) — WPA3's actual fix

**What it replaces:** WPA2's four-way handshake used PSK (Pre-Shared Key) authentication directly. WPA3 replaces this with **SAE**, also known by its more descriptive name: **Dragonfly Handshake**.

**How SAE actually prevents offline cracking, at a conceptual level:** SAE is built on a cryptographic technique that makes each authentication attempt require actual interaction with the access point in real time — there's no static piece of data (like the captured PSK-based handshake in WPA2) that can simply be taken away and brute-forced offline at leisure. Every single password guess would require a live, real-time exchange with the actual access point, which the access point can detect and rate-limit.

**Why this directly defeats Module 17's exact attack:** the entire premise of capturing a handshake and cracking it offline with Hashcat depends on the handshake itself containing enough information to verify a guessed password locally, without needing the access point involved again. SAE is specifically designed so that this offline verification isn't possible even if you do capture the exchange — making the entire "capture once, crack forever offline" model that Module 17 relied on fundamentally not work the same way against a properly implemented WPA3 network.

**Forward Secrecy — a second, related SAE benefit:** WPA3/SAE also provides forward secrecy, meaning even if the network's password is later discovered or changed, previously captured encrypted traffic from *before* that point still cannot be decrypted retroactively. WPA2 lacked this — if you ever learned a WPA2 network's password, you could decrypt any previously captured traffic from that network, even traffic captured before you knew the password, as long as you'd saved it.

---

## Protected Management Frames (PMF) — directly fixing the deauth weakness from Module 16-17

**What it addresses:** Module 16 explained that management frames (including deauthentication frames) are unauthenticated on WPA2, which is exactly what makes the forced-handshake technique in Module 17 possible at all. **PMF is WPA3's fix for precisely this gap** — management frames are now cryptographically authenticated, meaning a forged deauthentication frame from an attacker who doesn't know the network's actual credentials can be detected and rejected by client devices, rather than blindly trusted.

**Practical consequence for Module 17's technique specifically:** the `aireplay-ng --deauth` technique that forces a fresh handshake capture is directly, specifically defeated by PMF — a forged deauth frame sent without proper authentication is simply ignored by a PMF-compliant client, closing off the "active" handshake-forcing option that made Module 17's attack practical rather than a matter of indefinite passive waiting.

---

## Dragonblood — the real vulnerabilities found in early WPA3 implementations

**Why this section matters, and why it's important to be honest about it:** WPA3 fixing WPA2's specific known weaknesses doesn't mean WPA3 itself launched flawless — **Dragonblood** is the name given to a set of real, documented vulnerabilities discovered in 2019 in early SAE/Dragonfly implementations, a genuinely important reminder that a new standard fixing old problems doesn't automatically mean the new standard has zero problems of its own.

**Side-channel attacks (timing and cache-based):** Certain early implementations of the SAE handshake took a measurably different amount of time to process, or accessed CPU cache in subtly different patterns, depending on characteristics of the password being processed — an attacker capable of precisely measuring these tiny differences could use them as a side channel to narrow down password guesses, partially undermining SAE's core "no offline brute-forcing" design goal in specific flawed implementations, even though the protocol's underlying cryptographic design was itself sound.

**Downgrade attacks:** Many real-world devices were configured to support both WPA3 and WPA2 simultaneously, for backward compatibility with older client devices that didn't yet support WPA3 at all (a genuinely necessary transition accommodation). Dragonblood research demonstrated that an attacker could, in some configurations, force a connection to downgrade to the weaker WPA2 handshake specifically to exploit WPA2's known weaknesses — directly reusing every technique from Module 17 — even though the network was technically "a WPA3 network."

**Denial-of-Service via the handshake itself:** Because SAE's design requires more computational work per authentication attempt than WPA2's simpler PSK handshake (part of what makes offline brute-forcing harder), Dragonblood research also demonstrated that flooding an access point with a large number of simultaneous SAE authentication attempts could exhaust its processing resources — a genuine, protocol-level DoS vector traced directly back to the extra computational cost that was deliberately added specifically to prevent offline cracking, showing a real security-tradeoff: the same design choice that blocks one attack class opened a different one.

**Why these findings mattered practically, and what happened afterward:** these vulnerabilities were responsibly disclosed and led to real patches and implementation guidance updates across affected vendors — Dragonblood is a strong, concrete example of why "the newer standard" isn't automatically synonymous with "fully solved," and why a real security assessment should test the actual deployed implementation rather than simply trusting a network's labeled encryption standard at face value.

---

## Practical Takeaway for an Assessment
When encountering a WPA3 network during testing: don't assume it's automatically unbreakable just because it's the newer standard, but also don't assume the exact same Module 17 handshake-capture workflow will work unmodified — check specifically whether transition-mode/downgrade support is enabled (a real, testable configuration question), whether the specific access point's SAE implementation has been patched against known Dragonblood-class timing issues, and treat "WPA3" as a starting point for more nuanced testing, not an automatic pass.

## Quick-reference table

| WPA2 Weakness | WPA3 Fix | Real-world caveat |
|---|---|---|
| Offline handshake cracking (Module 17) | SAE/Dragonfly Handshake — requires live interaction per attempt | Early implementations had timing/cache side-channel leaks |
| No forward secrecy | SAE provides forward secrecy | — |
| Unauthenticated deauth frames (Module 16-17) | Protected Management Frames (PMF) | Only effective if PMF is actually enabled/enforced, not just supported |
| — | — | Downgrade attacks can force a fallback to vulnerable WPA2 in mixed-mode networks |
| — | — | SAE's extra computational cost introduced its own DoS vector |

---

## This closes the full 18-module network security sequence
Together, Modules 1-18 form a genuine, real-depth methodology: infrastructure fundamentals → the phases attackers actually follow → deep reconnaissance → raw network interaction (Netcat) → comprehensive scanning (Nmap) → automated deeper enumeration and evasion → per-service enumeration → manual packet crafting → password attacks (online and offline) → active traffic interception → memory-corruption exploitation and pivoting → firewall/detection evasion beyond scanning → the exploitation framework tying it together → real post-exploitation → architectural/governance review → and finally the entire wireless attack surface, fundamentals through practical cracking through the current state of its defense. Every module built on specific, named concepts from the ones before it, exactly as intended.
