# Module 3 - Reconnaissance In Depth

## Why this comes right after the phases overview
Module 2 named Reconnaissance as phase one. This module is the actual how — real tools, real commands, real reasoning for each one.

---

## OSINT (Open Source Intelligence)

**What it is:** Gathering information from publicly available sources — nothing hidden, nothing hacked, just information that's already out there and legally accessible: websites, social media, public records, DNS records, job postings, code repositories.

**What it's for, attacker's perspective:** Every piece of OSINT reduces the guesswork in later phases. Knowing an employee's job title and the tools mentioned in their LinkedIn profile can reveal the exact software stack to target. Knowing a company's physical address helps plan a physical social engineering attempt (tailgating, badge cloning). None of this requires touching the target's systems at all.

**SOCMINT (Social Media Intelligence)** — a specific subset of OSINT focused entirely on what people reveal on social platforms: location check-ins, photos with visible badge/ID cards in the background, complaints about "IT problems" that reveal what software the company uses, or even just an out-of-office reply revealing who's currently traveling (useful for a pretexting attempt impersonating them).

---

## OSINT DNS — extracting real information from domain records

**Why DNS is a genuine goldmine for recon:** DNS records exist to be publicly queryable by design (that's literally their job — translating names to addresses for anyone who asks) — which means a huge amount of infrastructure information is available with zero authentication required, zero exploitation, completely legal to query.

### The DNS record types worth knowing individually, not as one blob

**A Record** — maps a domain name to an IPv4 address. This is the most basic lookup — "what server does this hostname point to."
```
dig A target-company.com
```

**AAAA Record** — same as A, but for IPv6 addresses.

**MX Record (Mail Exchange)** — reveals which servers handle the domain's email. Directly useful for an attacker: it tells you exactly which mail provider/server to target for phishing infrastructure research, or to check for known mail-server vulnerabilities.
```
dig MX target-company.com
```

**NS Record (Name Server)** — reveals which DNS servers are authoritative for the domain. Useful because it tells you exactly where to direct a zone transfer attempt (covered in enumeration content) or DNS-based attacks.
```
dig NS target-company.com
```

**TXT Record** — a flexible, free-text record that organizations use for many purposes: SPF records (which servers are authorized to send email for this domain — useful for understanding their email security posture), domain verification strings for third-party services (which can reveal exactly which cloud/SaaS tools they use — Google Workspace, Salesforce, etc.), and sometimes accidentally leaked internal information nobody meant to expose here.
```
dig TXT target-company.com
```

**CNAME Record (Canonical Name)** — an alias pointing one hostname to another. Worth checking specifically because of the subdomain takeover risk already covered in the web security series — a CNAME pointing at a deprovisioned third-party service is exactly that vulnerability.

**SOA Record (Start of Authority)** — contains administrative information about the domain zone, including a technical contact email address — occasionally reveals an internal employee's real email format/naming convention, useful for guessing other employees' email addresses later.

### Full DNS enumeration, put together
```
dig ANY target-company.com          # request all record types at once (many modern servers restrict this)
dig A target-company.com
dig MX target-company.com
dig NS target-company.com
dig TXT target-company.com
whois target-company.com            # registrar info, registration dates, sometimes registrant contact details
```

**Why go through each record type individually instead of just running one broad command:** each record type answers a genuinely different recon question — MX tells you about email infrastructure, NS tells you about DNS infrastructure, TXT often reveals third-party tool usage. Treating this as "just run `dig ANY`" misses the reasoning for *why* each piece of information is useful for a specific follow-up action.

---

## Google Hacking / Google Hacking Database (GHDB)

**What it is:** Using Google's own advanced search operators to find specific, often sensitive information that was never meant to be publicly discoverable, but got indexed anyway because it was technically publicly accessible.

**What it's for, attacker's perspective:** Search engines have already done the scanning work for you, across the entire internet, for free. Instead of scanning one target's server for exposed files, you can ask Google directly: "show me every publicly indexed file of this exact type, on this exact domain."

### The actual operators, one at a time

**`site:`** — restricts results to one specific domain.
```
site:target-company.com
```

**`filetype:`** — restricts results to a specific file extension. Combined with `site:`, this becomes genuinely powerful:
```
site:target-company.com filetype:pdf
site:target-company.com filetype:xls
site:target-company.com filetype:env
```
The second and third examples specifically hunt for spreadsheets (which often contain employee lists, financial data) and `.env` files (which, as covered in the hardcoded secrets file in the web security series, frequently contain live credentials, if one was ever accidentally deployed to a public web root).

**`intitle:`** — searches for a specific word in a page's title.
```
intitle:"index of" site:target-company.com
```
This specific combination hunts for exposed directory listings (connecting directly to the directory listing vulnerability covered in the web security misconfiguration file) — Google indexed a raw file listing page because it was left publicly accessible.

**`inurl:`** — searches for a word appearing in the URL itself.
```
inurl:admin site:target-company.com
inurl:login site:target-company.com
```
Useful for discovering admin panels or login pages that aren't linked from anywhere on the main site but were still crawled and indexed.

**`intext:`** — searches for specific text within the page's body content.
```
intext:"password" filetype:log
```
Hunting for log files that were accidentally left web-accessible and happen to contain the literal word "password" in their content — a real, historically common finding.

### The GHDB itself
The **Google Hacking Database** (maintained publicly, historically associated with Offensive Security/Exploit-DB) is a curated, categorized collection of pre-built Google dork queries submitted by the security community — organized by what they're designed to find (exposed login portals, files containing passwords, vulnerable server banners, sensitive directories). Instead of inventing dork queries from scratch, checking the GHDB against a target is often the faster first step.

---

## Recon-ng — automating and organizing OSINT

**What it is:** A modular reconnaissance framework, built with a command interface deliberately similar to Metasploit's, specifically designed to automate the OSINT-gathering process and keep results organized in one place rather than scattered across manual searches.

**Why use a framework instead of just manually googling everything:** Manual OSINT doesn't scale, and results get lost or disorganized quickly. Recon-ng structures everything into **workspaces** (one per target/engagement) and stores discovered data (domains, hosts, contacts, credentials found in breach data) in a queryable local database as you go.

**Basic real workflow:**
```
recon-ng
[recon-ng][default] > workspaces create target-company
[recon-ng][target-company] > marketplace install recon/domains-hosts/hackertarget
[recon-ng][target-company] > modules load recon/domains-hosts/hackertarget
[recon-ng][target-company][hackertarget] > options set SOURCE target-company.com
[recon-ng][target-company][hackertarget] > run
[recon-ng][target-company] > show hosts
```
This sequence: creates a dedicated workspace for this specific target (keeping data separate from any other engagement), installs and loads a module that discovers subdomains/hosts via a third-party data source, points it at the target domain, runs it, then displays everything discovered so far — all stored and queryable for the rest of the engagement, not just printed once and lost.

---

## Dark Web / Dark Net — the specific recon use case
**What it actually is for security purposes:** Not primarily a place to "hack from" — its actual OSINT value is checking whether the target organization's data (leaked credentials, internal documents, employee information) has already appeared in breach dumps or is being sold/discussed on dark web marketplaces and forums. This tells you whether previously-compromised, potentially still-valid credentials exist for use in the credential stuffing techniques covered in earlier system-hacking content.

## Quick-reference table

| Technique | What it reveals |
|---|---|
| `dig MX` | Email infrastructure provider |
| `dig NS` | Authoritative DNS servers (zone transfer target) |
| `dig TXT` | Third-party tools in use, SPF/email security posture |
| `whois` | Registrar, registration dates, sometimes contact info |
| `site: + filetype:` | Specific exposed file types on the target domain |
| `intitle:"index of"` | Exposed directory listings |
| `inurl:admin/login` | Undiscoverable-by-navigation admin/login pages |
| Recon-ng | Organized, automated, database-backed OSINT collection |
| Dark web monitoring | Whether target's data/credentials are already compromised elsewhere |
