# 04 — Attack: AS-REP Roasting

**Access required:** none, technically — you just need to know (or guess) a valid username. You don't even need a password to *attempt* this attack, though you need a domain user account to run some tools comfortably.

## Background — what is "pre-authentication"?

Normally, when you log in to Kerberos:
1. You type your password.
2. Your computer encrypts a timestamp with a key derived from your password and sends it to the Domain Controller (this proves "I actually know the password" — this step is called **pre-authentication**).
3. The Domain Controller checks it, and if correct, replies with an **AS-REP** message (a starter ticket, like handing you your first wristband for the day).

Pre-authentication exists specifically to stop attackers from being able to just *ask* for that starter ticket without proving they know the password first.

## The vulnerability

Some accounts have the setting **"Do not require Kerberos pre-authentication"** turned on — sometimes because of a misconfiguration, sometimes for legacy/compatibility reasons.

For those accounts, **anyone** can request an AS-REP message for that username — no password needed to make the request! The AS-REP that comes back contains data that's encrypted using a key derived from that user's password. The attacker just... asks for it, downloads it, and cracks it offline. No lockouts, since you never actually "tried a password" against the real server.

## Step-by-step

**1. Confirm the target account has pre-auth disabled** — this is the setting you're looking for:
> Account setting: **"Do not require Kerberos pre-authentication"** = enabled

**2. Request the AS-REP hash** using Rubeus:
```powershell
Rubeus.exe asreproast /format:john /outfile:hash.txt
```
(Impacket's `GetNPUsers.py` is another common tool for this same job, often used from a Linux attack machine.)

**3. Crack the captured hash offline:**
```bash
john --wordlist=rockyou.txt hash.txt
```

If the account's password is in the wordlist, `john` cracks it and prints the plaintext password.

## Kerberoasting vs. AS-REP Roasting — what's the difference?

| | Kerberoasting | AS-REP Roasting |
|---|---|---|
| Target | Service accounts (have an SPN) | User accounts with pre-auth disabled |
| Requires valid login first? | Yes (any domain user) | No — can be attempted with just a username |
| What you crack | TGS (service ticket) | AS-REP (initial authentication reply) |

## Defenses
- Make sure **"Do not require Kerberos pre-authentication"** is disabled for every account unless there's a very specific legacy reason it's needed.
- Enforce strong password policies for all users — this defeats the offline cracking step even if pre-auth is accidentally left off.
- Monitor for AS-REP requests for accounts that don't normally get them (Event ID 4768 without pre-auth data).
