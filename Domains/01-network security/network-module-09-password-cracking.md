# Module 9 - Password Cracking

## Why this comes right after packet crafting
Modules 4-8 covered getting information about a target and manipulating traffic. Password cracking is often the actual door-opening step — this module goes far deeper than the conceptual "brute force vs dictionary" overview from earlier system-hacking content, with real tools and real commands.

---

## Online vs Offline Cracking — the distinction that determines everything else

**Online cracking** — repeatedly attempting logins directly against a live service (SSH, FTP, a web login form). Every single attempt is a real network request to the target.

**Offline cracking** — you already have the actual password *hash* (stolen from a compromised database, a captured Wi-Fi handshake, an extracted `/etc/shadow` file), and you're cracking it on your own hardware, with zero further interaction with the target at all.

**Why this distinction changes your entire approach:**
- Online attacks are **slow and loud** — every attempt is a real, loggable event, and most services implement rate limiting/lockout (the exact defense covered in the web security authentication file) specifically to blunt this.
- Offline attacks are **fast and completely silent** to the target — once you have the hash, the target has zero visibility into how many attempts you make, since none of them touch their systems at all.

---

## Attack Types, in real depth

### Dictionary Attack
**What it is:** Trying a curated list of likely passwords (common passwords, leaked password lists, or a target-specific custom list) rather than every possible combination.

**Building a target-specific wordlist — a real, practical technique:**
```
cewl http://target-company.com -d 3 -m 5 -w custom_wordlist.txt
```
`cewl` crawls a website (`-d 3` = crawl depth of 3 links deep, `-m 5` = minimum word length of 5) and extracts every word it finds into a wordlist — the reasoning being that company-specific terms, product names, and jargon are more likely to appear in employee passwords than generic dictionary words.

### Brute Force Attack
**What it is:** Trying every possible character combination within a defined character set and length range — guaranteed to eventually succeed, but the time cost grows exponentially with password length/complexity.

### Hybrid Attack
**What it is:** Combining a dictionary word with brute-force-style variations — appending numbers, symbols, or common substitutions (`Password` → `Password1`, `Password123`, `P@ssword`). This reflects genuine, well-documented human password-creation habits, making it far more efficient than pure brute force for real-world passwords.

### Rainbow Table Attack
**What it is:** Instead of computing hashes in real time during the attack, using a **precomputed** massive table mapping possible passwords directly to their resulting hash values — cracking becomes a fast lookup instead of a slow computation.
**Why salting defeats this entirely:** a salt (a unique random value added before hashing, covered in the sensitive data exposure file in the web series) means the same password produces a *different* hash for every user — a precomputed table built for unsalted hashes becomes completely useless, since it would need a separate table for every possible salt value, which isn't practically feasible.

### Credential Stuffing and Password Spraying
Already covered conceptually in the system hacking module — here they connect directly to real tooling below (Hydra can perform both).

---

## Hydra — the real online cracking tool

**Basic syntax:**
```
hydra -l <username> -P <password-wordlist> <target-ip> <service>
```
- `-l` — a single specific username
- `-L` — a file containing multiple usernames to try (instead of `-l`)
- `-P` — a wordlist file of passwords to try
- `-p` — a single specific password (instead of `-P`)

**SSH brute force (exactly matching your own notebook's workflow):**
```
hydra -L users.txt -P passwords.txt <target-ip> ssh
```

**FTP brute force:**
```
hydra -L users.txt -P passwords.txt <target-ip> ftp
```

**HTTP POST form brute force (web login pages):**
```
hydra -L users.txt -P passwords.txt <target-ip> http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials"
```
This one needs unpacking: `/login` is the form's submission path, `username=^USER^&password=^PASS^` tells Hydra where to insert each username/password guess in the POST body, and the final piece (`"Invalid credentials"`) is the **failure string** — the exact text Hydra should look for in the response to know an attempt failed, so it can correctly identify a *successful* login as whatever response does *not* contain that string.

**Adding speed/thread control:**
```
hydra -L users.txt -P passwords.txt -t 4 <target-ip> ssh
```
`-t 4` limits Hydra to 4 parallel connection threads — deliberately throttling speed to avoid triggering account lockout policies or overwhelming the target service, a real trade-off between speed and both stealth and avoiding unintentionally locking out legitimate accounts.

---

## John the Ripper — offline hash cracking

**Basic usage against a captured hash file:**
```
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

**Cracking Linux system password hashes (once you have local/root access to extract them):**
```
unshadow /etc/passwd /etc/shadow > combined.txt
john combined.txt
```
`unshadow` merges the two files together, since `/etc/passwd` alone doesn't contain the actual password hashes (only user account info) — those live separately in `/etc/shadow`, readable only by root, which is exactly why getting this file at all already implies significant access has been achieved.

**Viewing already-cracked results:**
```
john --show combined.txt
```

**Specifying a particular hash format explicitly (when auto-detection isn't reliable):**
```
john --format=md5crypt hashes.txt
```

---

## Hashcat — GPU-accelerated cracking

**Why Hashcat specifically, beyond "it's another cracking tool":** Hashcat is built to leverage GPU processing power, which can be **orders of magnitude faster** than CPU-based cracking (what John primarily relies on) for many hash types — a meaningful, practical difference when working through large wordlists or complex brute-force ranges.

**Basic dictionary attack syntax:**
```
hashcat -m 0 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt
```
- `-m 0` — specifies the hash mode/type (mode `0` is raw MD5; every hash algorithm has its own specific mode number that must be set correctly for cracking to work at all)
- `-a 0` — specifies the attack mode (mode `0` is a straight dictionary attack)

**Common hash mode numbers worth knowing by heart, since using the wrong one silently fails:**
| Mode | Hash type |
|---|---|
| 0 | MD5 |
| 100 | SHA1 |
| 1400 | SHA256 |
| 3200 | bcrypt |
| 1000 | NTLM (Windows) |

**Brute-force/mask attack (trying a defined pattern rather than a full wordlist):**
```
hashcat -m 0 -a 3 hashes.txt ?a?a?a?a?a?a
```
`-a 3` is mask attack mode; `?a?a?a?a?a?a` defines a 6-character pattern where each `?a` means "any character" (letters, numbers, symbols) — useful when you have some structural knowledge of the likely password (e.g. "always 6 characters, always ends in two digits" would become `?a?a?a?a?d?d`).

## Quick-reference table

| Tool | Type | Primary use |
|---|---|---|
| Hydra | Online | Brute-forcing live login services (SSH, FTP, HTTP forms) |
| John the Ripper | Offline | Cracking extracted/captured hashes, especially Linux `/etc/shadow` |
| Hashcat | Offline (GPU-accelerated) | Fast cracking of large hash sets, mask-based attacks |
| cewl | Wordlist generation | Building target-specific wordlists from a company's own website |

| Attack Type | How it works | Defeated by |
|---|---|---|
| Dictionary | Curated likely-password list | Genuinely random, non-dictionary passwords |
| Brute Force | Every possible combination | Sufficient length/complexity (exponential time cost) |
| Hybrid | Dictionary word + variations | Same as dictionary, plus avoiding predictable substitutions |
| Rainbow Table | Precomputed hash lookup | Salting |
