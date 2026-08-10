# Module 7 - Enumeration of Services

## Why this comes right after Nmap
Nmap (Modules 5-6) tells you a port is open and roughly what's running there. This module is what you actually *do* once you know that — service-by-service, with real commands, going deeper than the conceptual enumeration content covered earlier in this repo.

---

## FTP Enumeration (Port 21)

**Step 1 — confirm the service and version:**
```
nmap -sV -p 21 <target-ip>
```

**Step 2 — check for anonymous login (the exact technique from your own notebook):**
```
ftp <target-ip>
Username: anonymous
Password: anonymous
```
**Why this matters, attacker's perspective:** if it succeeds, you have unauthenticated file access before any exploitation has happened at all — as your own notes demonstrated, whatever files sit in that directory are now yours to read, and sometimes (as with your `.pcap` example) those files themselves contain the next set of credentials.

**Step 3 — once connected, basic FTP commands:**
```
ls -la          # list files, including hidden ones
get <filename>  # download a file
put <filename>  # upload a file (only works if write access is also permitted — a much more severe finding)
```

**Step 4 — check for a known vulnerable version directly:**
```
searchsploit vsftpd 2.3.4
searchsploit ProFTPD 1.3.5
```
**Why this specific check matters:** some FTP server versions have publicly known, severe vulnerabilities (vsftpd 2.3.4 has a well-known backdoor vulnerability that gives a root shell directly) — confirming the exact version via `-sV` first is what makes this targeted search possible instead of guessing.

---

## HTTP Enumeration (Port 80/443)

**Step 1 — confirm version and check for immediate information disclosure:**
```
nmap -sV -p 80,443 <target-ip>
curl -I http://<target-ip>
```
`curl -I` fetches only the HTTP headers — a fast way to see the exact server software/version string without loading the full page.

**Step 2 — check for `phpinfo()` exposure (from your own notes):**
```
http://<target-ip>/phpinfo.php
```
**Why this is a real finding, not a curiosity:** `phpinfo()` dumps the entire PHP configuration — file paths, loaded modules, sometimes environment variables containing credentials, and the exact PHP version, all in one page. This connects directly to the sensitive data exposure and security misconfiguration content in the web security series — it's the same category of "the application is telling you far more than it should."

**Step 3 — directory/file discovery:**
```
gobuster dir -u http://<target-ip> -w /usr/share/wordlists/dirb/common.txt
```
Systematically requests every word in the wordlist as a path, revealing directories/files that exist but aren't linked from anywhere visible — the automated, comprehensive version of the `inurl:admin` Google dork from Module 3, run directly against the target instead of relying on what's already indexed.

**Step 4 — check for known vulnerable web technology:**
```
searchsploit php-cgi
searchsploit apache 2.4
```

**Step 5 — Metasploit-based enumeration (from your own notes):**
```
msfconsole
search http-version
use 0
options
set rhost <target-ip>
run
```

---

## SMTP Enumeration (Port 25)

**Why SMTP is worth enumerating specifically:** SMTP servers historically respond differently to valid vs. invalid usernames on certain commands, which can be abused to enumerate real employee usernames/email addresses one at a time — directly useful input for the password-spraying and phishing techniques covered elsewhere in this series.

**Manual banner grab first:**
```
nc <target-ip> 25
```
Many SMTP servers announce their exact software and version immediately on connection, the same banner-grabbing principle from earlier enumeration content.

**The VRFY command — the actual username enumeration technique:**
```
VRFY root
VRFY nonexistentuser12345
```
A properly configured, hardened SMTP server should respond identically regardless of whether the username is real. A **misconfigured** one responds differently — e.g. "250 OK" for a real user vs. "550 No such user" for a fake one — directly confirming which usernames are valid, one guess at a time.

**Automated version, using Nmap's NSE:**
```
nmap -p 25 --script smtp-enum-users <target-ip>
```

**Mitigation (worth knowing, since it's a common finding):** disable or restrict the VRFY command, and ensure consistent response behavior regardless of username validity.

---

## SNMP Enumeration (Port 161)

**Already introduced conceptually in earlier enumeration content — here's the full practical depth.**

**Step 1 — check if a default community string works at all:**
```
snmpwalk -c public -v1 <target-ip>
snmpwalk -c private -v1 <target-ip>
```
`-c` specifies the community string (the "password" for SNMP v1/v2c — sent in plaintext, worth noting), `-v1` specifies SNMP version 1.

**Step 2 — if that works, pull specific, high-value information using MIB (Management Information Base) object identifiers:**
```
snmpwalk -c public -v1 <target-ip> 1.3.6.1.2.1.25.4.2.1.2   # running processes
snmpwalk -c public -v1 <target-ip> 1.3.6.1.2.1.25.6.3.1.2   # installed software
snmpwalk -c public -v1 <target-ip> 1.3.6.1.4.1.77.1.2.25    # Windows user accounts (specific to certain SNMP implementations)
```
**Why specific OIDs matter, not just a blanket dump:** a full `snmpwalk` with no OID specified returns an enormous, unfiltered amount of data — knowing the specific OID for "user accounts" or "running processes" lets you pull exactly the high-value information you actually want, rather than manually scrolling through everything.

**Step 3 — via Metasploit and NSE, as automated alternatives:**
```
msfconsole
search snmp_enum
use 0
set rhosts <target-ip>
run
```
```
nmap -sU -p 161 --script snmp-brute <target-ip>
```
`snmp-brute` specifically attempts a list of common community strings automatically, rather than manually guessing `public`/`private` one at a time.

## Quick-reference table

| Service | Port | Key technique | What it reveals |
|---|---|---|---|
| FTP | 21 | Anonymous login attempt | Unauthenticated file access |
| HTTP | 80/443 | `phpinfo()` check, directory brute-force | Config details, hidden paths |
| SMTP | 25 | `VRFY` command | Valid employee usernames |
| SNMP | 161 | Default community strings (`public`/`private`) | Processes, software, sometimes user accounts |
