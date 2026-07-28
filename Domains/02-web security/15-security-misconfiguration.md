# Security Misconfiguration & Information Disclosure

## Why this comes next
Every vulnerability so far required finding a specific flaw in how the application handles input. This category is different — it's about what the application *reveals or leaves exposed* through default settings, verbose errors, or forgotten configuration, often without any input manipulation needed at all. It's frequently the very first thing found in an assessment, since it requires no exploitation technique — just paying attention to what the application is already telling you.

---

## Verbose Error Messages / Stack Traces

**What it is:** Detailed technical error information (full stack traces, database error messages, internal file paths) shown directly to the user instead of a generic error page.

**Example:**
```
VULNERABLE response:
Error: MySQLSyntaxErrorException: You have an error in your SQL syntax near
'username' = ''' at line 1. File: /var/www/app/models/user.py, line 47,
in authenticate() — DB user: app_prod_user, Host: db-primary-01.internal
```
```
SECURE response:
"Something went wrong. Please try again or contact support. (Reference: a1b2c3)"
```

**Why it matters:** The vulnerable response hands an attacker enormous reconnaissance value for free — it confirms the exact database type (useful for crafting SQLi payloads), reveals internal file paths and server structure, and even leaks an internal hostname (`db-primary-01.internal`) that shouldn't be visible outside the network at all.

**Business Impact:** Doesn't cause a breach by itself, but dramatically speeds up every other attack in this series — an attacker with detailed error information can craft more precise SQLi or path traversal payloads far faster than working blind, shortening the time from "found a bug" to "successfully exploited it."

**Mitigation:** Show generic error messages to users; log the full technical detail server-side only, tied to a reference ID the user can quote to support if needed.

---

## Directory Listing Enabled

**What it is:** A web server configured to show a full file listing when a folder with no index page is requested directly, instead of returning a 403/404.

**Example:** Requesting `https://bank.com/uploads/` with directory listing enabled returns a clickable list of every file in that folder — potentially exposing files never meant to be publicly browsable (old backups, configuration files, other users' uploaded documents).

**Business Impact:** Can directly expose sensitive files (backups, configs with credentials, other customers' uploaded KYC documents) with zero exploitation skill required — just browsing to a folder URL.

**Mitigation:** Explicitly disable directory listing at the web server level (`Options -Indexes` in Apache, `autoindex off` in Nginx); place an empty `index.html` in every publicly accessible folder as a backup measure.

---

## Default Credentials / Unremoved Test Accounts

**What it is:** Admin panels, databases, or third-party software left with factory-default usernames/passwords, or test accounts created during development that were never removed before going live.

**Example:** An admin panel at `/admin` still accepting `admin`/`admin`, or a `test_user`/`password123` account with elevated privileges left in the production database from development.

**Business Impact:** This is functionally identical to having no authentication at all — the "vulnerability" isn't a coding flaw, it's simply that the front door was never locked. For a bank, an unremoved test account with admin privileges discovered in a production system is a severe, embarrassing finding precisely because it required zero technical skill to exploit.

**Mitigation:** Enforce mandatory password changes on all default accounts before deployment; maintain a checklist that explicitly verifies test/development accounts are removed as part of the production release process, not left to individual memory.

---

## Exposed `.git`, Backup Files, and Config Files

**What it is:** Development artifacts accidentally left accessible on the live web server — an exposed `.git` folder (which can be used to reconstruct entire source code history), `.env` files containing secrets, or backup files like `config.php.bak`.

**Example:** `https://bank.com/.git/config` being directly accessible lets an attacker use standard tools to download the entire Git history — including any credentials or secrets ever committed, even ones later "removed" in a subsequent commit (they still exist in history).

**Business Impact:** A single exposed `.env` or `.git` folder can leak database credentials, API keys, or signing secrets directly — often a faster, more complete path to full compromise than any of the technical vulnerabilities covered earlier, since it skips exploitation entirely and hands over the credentials directly.

**Mitigation:** Ensure the web server's document root only contains files genuinely meant to be public — build/deployment processes should copy only the necessary compiled output to the server, never the full source repository including `.git`. Explicitly block access to dotfiles and common backup extensions at the web server level as a backup layer.

---

## Unnecessary HTTP Methods Enabled

**What it is:** HTTP methods beyond GET/POST left enabled on the server when they're not actually needed — most notably `PUT`, `DELETE`, or `TRACE`.

**Business Impact:** `PUT` left enabled without proper access control can potentially allow uploading files directly; `TRACE` can be combined with other flaws to bypass `HttpOnly` cookie protections in some legacy attack scenarios. Low likelihood individually, but a completely unnecessary extra attack surface that provides no legitimate value if unused.

**Mitigation:** Explicitly restrict allowed HTTP methods to only those the application actually uses, at the web server or application framework level.

---

## Quick-reference table

| Issue | What it hands the attacker | Effort to exploit |
|---|---|---|
| Verbose error messages | Database type, file paths, internal hostnames | None — just trigger an error |
| Directory listing | Full file browsing of exposed folders | None — just visit the URL |
| Default credentials | Direct access, no exploitation needed | None — try known defaults |
| Exposed `.git`/`.env` | Source code and/or credentials directly | Low — standard public tools |
| Unnecessary HTTP methods | Extra attack surface for other techniques | Varies |

## Why this category matters in an interview
Every one of these requires essentially zero exploitation skill — they're about attention to detail and deployment hygiene, not clever payload crafting. Being able to say *"before I even start looking for injection flaws, I check what the application is already telling me for free — error messages, exposed files, default settings"* signals a methodical, complete approach rather than jumping straight to the "interesting" vulnerabilities.

## Explaining it to a developer
*"None of these are bugs in the traditional sense — the code works fine. The problem is configuration and deployment hygiene: verbose errors that leak internal details, directories that shouldn't be browsable, accounts that were never meant to reach production, and files that shouldn't have been deployed at all. The fix for most of these is a pre-deployment checklist, not a code change — but that checklist has to actually be enforced, not just documented."*
