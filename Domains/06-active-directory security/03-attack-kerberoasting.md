# 03 — Attack: Kerberoasting

**Access required:** any valid domain user account (even a low-privilege one).

## First — what is Kerberos, in plain English?

Kerberos is the "ID card checking system" that Active Directory uses for logins. Instead of typing your password every single time you access a new resource, you get a **ticket** (like a concert wristband) once you log in, and you just show that ticket to access things afterward.

Some tickets are for accessing **services** — e.g. a SQL database service, a web app, a file share. These are called **service tickets (TGS)**, and they're tied to a **Service Principal Name (SPN)** — basically the "name tag" of that service in AD (e.g. `mssql_svc/cs.org:1433`).

## The core idea behind Kerberoasting

Here's the trick: **any regular domain user is allowed to request a service ticket for any service** — that's normal, expected behavior. The service ticket is encrypted using a key derived from the **service account's password**.

So an attacker who is any logged-in domain user can simply *ask* for a ticket to a juicy service (like a SQL server running under a "svc_sql" account), take that ticket home, and try to crack it offline — no need to touch the real server again, and no alarms triggered on the Domain Controller.

**In short:**
1. Any domain user can request a Kerberos service ticket for any service account with an SPN.
2. That ticket is encrypted with a key based on the *service account's* password.
3. The attacker exports the ticket and cracks it **offline** (on their own laptop, at their own pace — no lockouts, no logs on the target).
4. If the service account's password is weak, the crack succeeds and the attacker now has that account's real password.

This works especially well against **service accounts**, because those are often set up once, forgotten about, and given passwords that never expire and are rarely as strong as regular human accounts.

## Step-by-step (as shown in the lab)

**1. Set up an SPN** (this step simulates how a real service account normally gets configured):
```powershell
setspn -S mssql_svc/cs.org:1433 cs\ethel.carla
```
This tells AD "the account `ethel.carla` is responsible for running the SQL service at `cs.org:1433`".

**2. Confirm the SPN was registered correctly:**
```powershell
setspn -T cs.org -Q */*
```

**3. Find "roastable" accounts** (accounts with SPNs set — i.e. potential targets):
```powershell
# Using PowerView
./Find-PotentiallyCrackableAccounts -FullData -Verbose

# Or the classic Kerberoast script
./GetUserSPNs.ps1
```

**4. Request the actual service ticket and format it for cracking:**
```powershell
./Get-TGSCipher.ps1 -SPN "mssql_svc/cs.org:1433" -Format John
```

**Or, using Rubeus** (a popular C# toolkit purpose-built for Kerberos attacks):
```powershell
Rubeus.exe kerberoast /outfile:hash.txt
```

**5. Crack the extracted hash offline with Hashcat:**
```bash
hashcat -m 13100 -a 0 hash.txt pass.txt
```
- `-m 13100` tells Hashcat "this is a Kerberos TGS-REP hash".
- `-a 0` means a straightforward dictionary attack.
- `hash.txt` is the captured ticket, `pass.txt` is your password wordlist (e.g. `rockyou.txt`).

If the service account's password is in your wordlist (or weak enough to guess), Hashcat cracks it and you now have valid credentials for that service account — often with much higher privileges than a normal user.

## Why this attack works (root cause)

Kerberos service tickets are encrypted using a key derived directly from the service account's **NTLM password hash**. Weak service account password = weak encryption key = crackable offline.

## Defenses (see also `07-mitigations.md`)
- Use **long, random, complex passwords** for every service account (25+ characters, generated — never typed by a human).
- Put sensitive service accounts in the **"Protected Users"** security group, which restricts how Kerberos can be used with them.
- Where possible, use **Group Managed Service Accounts (gMSA)** — Windows auto-rotates their passwords, making offline cracking essentially useless.
- Monitor for unusual volumes of TGS ticket requests (Event ID 4769) — a sudden burst is a Kerberoasting red flag.
