# 07 — Mitigations Summary & Quick Reference

## Quick reference: what access do you need for each attack?

| Attack | Access needed to start | What you walk away with |
|---|---|---|
| Kerbrute enumeration | None (no domain access required) | Valid usernames |
| Password Spraying | None (just usernames) | Valid credentials for a weak-password account |
| AS-REP Roasting | None — just a username | Crackable hash → plaintext password |
| Kerberoasting | Any valid domain user | Crackable service-account hash → plaintext password |
| Pass the Hash | Local admin on a machine a target logged into | Authenticated session as target, no password needed |
| Pass the Ticket | Access as a domain user | Authenticated access using a stolen/forged ticket |
| Silver Ticket | A service account's NTLM hash | Forged access to one specific service |
| Golden Ticket | Full Domain Admin (briefly) | Permanent, domain-wide forged access |
| DCSync | Domain replication rights | Password hashes for any account, incl. KRBTGT |

Notice the pattern: **each attack tends to unlock the next one.** A weak service account password (Kerberoasting) can snowball all the way to a Golden Ticket and full domain takeover. This is why defenders can't just fix "the one bug" — they need to break the whole chain.

## Master mitigation checklist

**Passwords & accounts**
- [ ] Strong, unique, randomly generated passwords for **every** service account.
- [ ] Regularly rotate the **KRBTGT** account password (twice, back-to-back, after any suspected compromise).
- [ ] Disable **"Do not require Kerberos pre-authentication"** unless there's a documented legacy reason.
- [ ] Use **Group Managed Service Accounts (gMSA)** so Windows auto-rotates service passwords.
- [ ] Enforce a strong, org-wide password policy to blunt password spraying and offline cracking.

**Privilege & access control**
- [ ] Principle of least privilege everywhere — no one gets more access "just in case".
- [ ] Never use Domain Admin accounts for everyday tasks.
- [ ] Put sensitive accounts in the **Protected Users** security group.
- [ ] Remove standing admin rights on workstations; use controlled/just-in-time privilege elevation.

**Protocols & configuration**
- [ ] Disable legacy protocols like **NTLM** wherever Kerberos can be used instead.
- [ ] Enable **PAC validation** to defeat Silver Ticket forgery.
- [ ] Adopt a **Zero Trust** approach — don't automatically trust anything just because it's "inside" the network.

**Detection & monitoring**
- [ ] Alert on abnormal volumes of TGS ticket requests (possible Kerberoasting).
- [ ] Alert on AS-REP requests for accounts that don't normally receive them.
- [ ] Alert on unusual domain replication traffic (possible DCSync).
- [ ] Alert on Kerberos tickets with abnormally long lifetimes.
- [ ] Alert on sudden changes to privileged group membership.
- [ ] Deploy an Identity Threat Detection & Response (ITDR) tool.
- [ ] Do proactive threat hunting, not just alert-driven response.

## Tools mentioned in these notes

| Tool | What it's for |
|---|---|
| **PowerView** | PowerShell-based AD enumeration (users, groups, computers, policies) |
| **Rubeus** | C# toolkit for requesting/forging/abusing Kerberos tickets |
| **Mimikatz** | Extracts credentials from memory; forges Golden/Silver tickets; performs Pass-the-Hash/DCSync (lab use only) |
| **Impacket (GetNPUsers.py, GetUserSPNs.py)** | Python toolkit for AS-REP Roasting and Kerberoasting, often from a Linux attacker box |
| **Hashcat / John the Ripper** | Offline password/hash cracking |
| **CrackMapExec** | Fast AD/SMB enumeration and credential validation across many hosts at once |
| **BloodHound** | Visual mapping of AD relationships to find the shortest attack path to Domain Admin |
