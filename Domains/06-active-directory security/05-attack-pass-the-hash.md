# 05 — Attack: Pass the Hash (PtH)

**Access required:** administrative/local access on at least one machine where a target user has logged in (to extract the hash from memory).

## The core idea, explained simply

When you log in to Windows with a password, Windows doesn't keep re-checking your typed password over and over for every action. Instead, it converts your password into a scrambled value called an **NTLM hash**, and reuses *that* hash behind the scenes to prove who you are to other computers on the network.

Here's the vulnerability: **many older/legacy authentication systems (NTLM) don't actually require the real password — they just need the hash.** If an attacker can somehow steal that hash, they can present it directly to another system and get logged in — **without ever knowing the real password at all.**

It's like a hotel where the front desk doesn't check your ID, they just check if you have a photocopy of *someone's* key card — they never bothered verifying the actual key card is real, or that you know what its "real" secret was, just that the photocopy matches an authorized code.

## Step-by-step

**1. Get administrative access to a machine** (this is a prerequisite — PtH is a follow-on attack after you already have a foothold).

**2. Extract password hashes from memory using Mimikatz:**
```
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpassword
```
- `privilege::debug` elevates Mimikatz's own permissions so it can read protected memory.
- `sekurlsa::logonpassword` dumps every credential (including NTLM hashes) currently cached in memory for logged-in users on this machine.

**3. Use the extracted NTLM hash to authenticate as that user elsewhere on the network:**
```
mimikatz # sekurlsa::pth /user:<username> /domain:<domain name> /ntlm:<ntlm hash>
```
This opens a new command prompt that is *already authenticated* as that user — using only the hash, never the plaintext password.

## Why this is so dangerous in AD environments

If IT admins log in to many different machines using the same privileged domain admin account (very common — e.g. an admin RDPs into every server to do maintenance), their hash gets cached in memory on **every single one** of those machines. An attacker who compromises just ONE low-value machine that the admin happened to log into can extract that hash and now move ("pivot") to any other machine that trusts that hash — this chain is called **lateral movement**.

## Defenses (see also `07-mitigations.md`)
- **Principle of least privilege** — don't use Domain Admin accounts for routine tasks; use separate, lower-privilege accounts for day-to-day admin work.
- **Zero Trust** approach — never assume a device or hash is trustworthy just because it's "inside" the network.
- **Disable legacy protocols like NTLM** wherever Kerberos can be used instead.
- Restrict which applications/resources an identity can reach, based on actual need.
- Deploy an **Identity Threat Detection and Response (ITDR)** solution to catch abnormal authentication patterns.
- Practice **proactive threat hunting** rather than only reacting to alerts.
