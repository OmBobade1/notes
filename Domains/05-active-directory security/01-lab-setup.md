# 01 — Building Your Own Active Directory Lab

You can't (and shouldn't) practice AD attacks on a real company. So the first step is building your own **toy Active Directory** on your own computer, inside a virtual machine. Nobody else is affected, and you can break things freely.

## What you need

1. **VMware Workstation/Player** or **VirtualBox** — free virtualization software to run a "computer inside your computer".
2. **Windows Server 2019 evaluation ISO** — free 180-day trial from Microsoft:
   `https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2019`
3. Enough RAM/disk space on your host machine (Windows Server VMs are heavy — 4GB+ RAM for the VM is a good starting point).

## Step 1 — Install Windows Server as a VM

1. Download the evaluation ISO from the Microsoft link above (fill in the short form, then download).
2. Open VMware, create a new VM, and point it at the ISO.
3. Install Windows Server like a normal OS install (Next → Next → set an Administrator password).

## Step 2 — Turn that plain Windows Server into a Domain Controller

Once Windows Server is installed and running, open **PowerShell as Administrator** and run:

```powershell
# Install the Active Directory Domain Services role
Install-WindowsFeature AD-domain-services

# Load the module that lets us actually create a domain
Import-Module ADDSDeployment

# Promote this server into a brand-new Active Directory forest
Install-ADDSForest `
  -CreateDnsDelegation:$false `
  -DatabasePath "C:\Windows\NTDS" `
  -DomainMode "7" `
  -DomainName "cs.org" `
  -DomainNetbiosName "cs" `
  -ForestMode "7" `
  -InstallDns:$true `
  -LogPath "C:\Windows\NTDS" `
  -NoRebootOnCompletion:$false `
  -SysvolPath "C:\Windows\SYSVOL" `
  -Force:$true
```

**What did that just do, in plain English?**
- `Install-WindowsFeature AD-domain-services` — installs the AD software on this Windows Server.
- `Install-ADDSForest` — creates a brand new forest (remember: forest = the biggest container) with a domain called `cs.org` (internal short name `cs`).
- The server reboots and comes back up as a **Domain Controller** — the "principal's office" for your fake school/company.

## Step 3 — Make the lab "vulnerable on purpose"

A freshly installed AD has almost no users and nothing interesting to attack. To have a realistic playground, security researchers built a free script that auto-populates a domain with realistic (and intentionally weak/vulnerable) users, groups, and misconfigurations — great for practicing attacks.

```powershell
IEX((New-Object Net.WebClient).DownloadString("https://raw.githubusercontent.com/wazehell/vulnerable-AD/master/vulnad.ps1"))

Invoke-VulnAD -UsersLimit 100 -DomainName "cs.org"
```

**What did that do?**
- `IEX(... DownloadString(...))` downloads a PowerShell script from GitHub and runs it directly in memory. (This is a common technique — good to recognize it, both for attacking and for detecting it as a defender, since "download and run a script straight into memory" is a classic red flag in logs.)
- `Invoke-VulnAD` then creates 100 fake users and some deliberately weak configurations in your `cs.org` domain, so you have something realistic to enumerate and attack.

## Step 4 — Confirm your lab is alive

From the Domain Controller, open Command Prompt / PowerShell and check:

```powershell
net user                      # lists local/domain user accounts
whoami                        # who am I logged in as?
whoami /priv                  # what privileges does my current session have?
```

If you see a list of users, congratulations — your lab domain is up and running. You're ready for `02-enumeration.md`.
