# 02 — Active Directory Enumeration

**Enumeration** just means "looking around and taking notes before you do anything else." Before attacking anything, you need to know: What domain am I in? Who are the users? Which groups matter? Who is a Domain Admin? This step is 80% of real AD hacking — most attacks are trivial *once* you've found the right target.

## The mindset

Think of it like being a new employee on your first day, quietly reading the office directory board:
- What's the company (domain) called?
- Who works here (users)?
- Which department is IT/Admin (privileged groups)?
- Which computers exist, and what do they run?

## Built-in Windows commands (no extra tools needed)

```powershell
whoami                     # who am I?
whoami /priv               # what special permissions does my current login have?
net user                   # list of user accounts
```

```powershell
# .NET way to ask "what domain am I currently part of?"
$ADClass = [System.DirectoryServices.ActiveDirectory.Domain]
$ADClass::GetCurrentDomain()
```

## PowerView — the go-to enumeration tool

**PowerView** is a PowerShell tool that replaces a lot of old-school `net *` Windows commands with much more powerful AD-aware versions. It's part of the [PowerSploit](https://github.com/PowerShellMafia/PowerSploit) toolkit.

Think of PowerView as "Google Search, but for the internals of the AD database" — instead of clicking through a GUI, you ask it direct questions.

```powershell
# Basic domain info
Get-NetDomain                       # what domain am I in?
Get-DomainSID                       # unique ID number of this domain
Get-NetDomainController             # find the Domain Controller(s)

# Domain-wide password policy
Get-DomainPolicy                            # full policy object
(Get-DomainPolicy)."system access"          # login/lockout settings
(Get-DomainPolicy)."kerberospolicy"         # Kerberos ticket lifetime settings

# Users
Get-NetUser                                        # list ALL domain users (lots of detail)
Get-NetUser -UserName delphine.katha               # look at one specific user
Get-UserProperty                                    # list what fields exist on user objects
Get-UserProperty -Property pwdlastset               # e.g. see when passwords were last changed

# Search for juicy info hidden in "Description" fields
# (Admins sometimes carelessly write passwords/hints in a user's description!)
Find-UserField -SearchField Description -SearchTerm "built"

# Computers
Get-NetComputer                                     # list all computers in the domain
Get-NetComputer -OperatingSystem "*Server 2019*"    # filter by OS
Get-NetComputer -Ping                               # only show computers that respond to ping
Get-NetComputer -FullData                           # get every detail about each computer

# Groups (this is where the good stuff is — who's an Admin?)
Get-NetGroup                                # list all groups
Get-NetGroup -Filter * | select name        # just the group names, tidy output
Get-NetGroup 'Domain Admins' -FullData      # full detail on the most powerful group
Get-NetGroup -GroupName *admin*             # find any group with "admin" in the name
Get-NetGroupMember                          # who belongs to a given group?
```

### With PowerView, you can also enumerate domain users directly:

```powershell
Get-DomainUser
```

**Bonus trick — finding passwords admins accidentally left in plain sight:**

```powershell
Get-DomainUser | Where-Object {$_.description -like "*password*"} |
  Select-Object -Property description, samaccountname
```
This checks every user's "Description" field for the word "password". You'd be surprised how often a lazy admin writes something like `Description: temp pwd is Summer2023!` — enumeration catches these easy wins.

## Password Spraying

**What it is (in plain words):** instead of guessing 1000 passwords for ONE account (which locks you out fast), you try ONE common password (like `Summer2024!`) against MANY accounts. Since each account only gets tried once, you avoid tripping account-lockout policies — and because it uses real, valid-looking login attempts, it's harder for defenders to spot than a classic brute-force attack.

```powershell
Invoke-DomainPasswordSpray -PasswordList pass.txt -UserList user.txt
```
This takes a list of usernames (`user.txt`) and a list of candidate passwords (`pass.txt`), and quietly tries combinations across the whole domain. If even one employee reused a weak/common password, you get a valid login.

**Why this matters for defenders too:** this is why companies enforce strong, unique passwords and monitor for many failed logins spread across many accounts in a short time (a "spray" pattern), not just many failed logins on one account.

---

Once enumeration gives you a target (a user, a service account, or valid credentials), you move on to the actual attacks — see the `03` through `06` files.
