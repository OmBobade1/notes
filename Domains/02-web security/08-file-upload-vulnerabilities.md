# File Upload Vulnerabilities

## Why this comes right after Broken Access Control
Access control governs *who* can reach a resource. File upload is where an attacker gets to introduce a completely new resource of their own choosing into the application — often the most direct path from "web bug" to "full server compromise" of anything covered so far, since a successful malicious upload can lead straight to remote code execution.

---

## What it is (in plain terms)
File upload vulnerabilities happen when an application accepts a file from a user (a profile picture, a KYC document, a support attachment) without properly restricting *what kind* of file it actually is, *where* it gets stored, or *what happens* when it's later accessed — allowing an attacker to upload something that isn't what it claims to be, and get the server (or other users' browsers) to execute it.

## Why it exists — the real-life cause

```python
# VULNERABLE — trusts the filename's extension and does nothing else
@app.route('/upload', methods=['POST'])
def upload():
    file = request.files['profile_picture']
    filename = file.filename  # e.g. "photo.jpg" — but is it TRUSTED?
    file.save(f"/var/www/uploads/{filename}")
```
This code trusts the filename entirely. An attacker can name a file `shell.php.jpg`, `shell.php%00.jpg` (null-byte trick on older systems), or simply `shell.php` if the server doesn't check the extension at all — and if the uploads folder is inside the web root and the server executes `.php` files, visiting that uploaded file's URL directly runs it as a script on the server.

```python
# SECURE — validates actual file content, generates a safe filename, stores outside web root
import magic, uuid

@app.route('/upload', methods=['POST'])
def upload():
    file = request.files['profile_picture']
    file_type = magic.from_buffer(file.read(2048), mime=True)  # check ACTUAL content, not just the name
    if file_type not in ['image/jpeg', 'image/png']:
        abort(400)
    safe_filename = f"{uuid.uuid4()}.jpg"  # attacker-chosen filename is discarded entirely
    file.save(f"/var/uploads_private/{safe_filename}")  # stored OUTSIDE the web-servable directory
```
This checks the file's actual binary content (not just the name the attacker typed), generates a completely new filename so nothing attacker-controlled survives, and stores it somewhere the web server won't directly execute even if something slipped through.

## How an attacker actually does it, step by step
1. Find any upload feature — profile picture, document upload, support ticket attachment.
2. Try uploading a file with a dangerous extension directly (`shell.php`) — if rejected, try bypass tricks: double extensions (`shell.php.jpg`), case variation (`shell.PHP`), or a valid image file with malicious code appended to it (polyglot files).
3. If the upload succeeds, try to find where the file was stored (often a predictable path like `/uploads/<filename>`).
4. Visit the uploaded file's URL directly — if the server executes it instead of just displaying/downloading it, the attacker now has code execution on the server.

## Technical Impact
- **Remote Code Execution (RCE)** — if an executable script gets uploaded and the server runs it, the attacker gets a foothold directly on the server itself, not just the application
- **Stored XSS** — an uploaded file disguised as an image but containing HTML/script content, served back with the wrong `Content-Type`, can execute in other users' browsers (see file `05` and the related `X-Content-Type-Options` header in file `01`)
- **Denial of Service** — uploading extremely large files or huge quantities of files to exhaust server storage
- **Path traversal** — a malicious filename like `../../etc/passwd` (if not sanitized) could overwrite unintended files elsewhere on the server

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | Remote code execution via a malicious upload is often the single worst-case outcome possible — from here, an attacker can pivot to the database, internal systems, or anything else the compromised server can reach, making the eventual financial impact essentially open-ended rather than limited to one feature |
| **Regulatory / compliance** | A confirmed RCE finding on a banking system is treated as a critical, immediate-remediation-required incident by any serious auditor or regulator — this is the category of finding that can trigger emergency disclosure obligations |
| **Reputational damage** | If exploited in the wild (not just found during testing), server compromise can lead to full data breaches, site defacement, or the bank's infrastructure being used to attack others — reputational damage well beyond a single feature's failure |
| **Legal liability** | Full server compromise stemming from an insufficiently validated upload feature is a severe negligence claim — "we let anyone upload any file type to a production server" is difficult for a bank to defend |
| **Operational cost** | A confirmed RCE typically triggers a full incident response process — assume breach, rebuild affected systems from clean images, rotate all credentials/secrets the compromised server had access to — this is one of the most expensive incident categories to recover from |

**One-line interview answer:** *"Technically, an unvalidated file upload can let an attacker upload a script instead of the expected file type, and if the server executes it, that's remote code execution. For a bank, this is one of the most severe possible findings because it's not limited to one feature or one customer's data — a compromised server can potentially be used to pivot into the database or other internal systems, so the actual business impact is effectively open-ended until contained."*

## Mitigation — layered, not just one fix

1. **Validate actual file content, not just the extension or filename** — check the file's real MIME type/magic bytes (as shown above), since an attacker fully controls the filename and can lie about it.
2. **Generate a new, random filename server-side** — never use the attacker-supplied filename directly, removing any path-traversal or double-extension tricks entirely.
3. **Store uploads outside the web-servable directory** (or on separate storage like S3 with no execute permissions) — even if a malicious file slips through validation, the web server should never be capable of executing it.
4. **Set the correct `Content-Type` and `Content-Disposition: attachment` header when serving uploaded files back** — forces the browser to download the file rather than render/execute it inline, working alongside `X-Content-Type-Options: nosniff` from file `01`.
5. **Enforce file size limits** — as a straightforward defense against storage-exhaustion denial of service.
6. **Scan uploaded files with antivirus/malware scanning** where feasible, as an additional layer — not a replacement for the above, since scanners can miss novel payloads.

## Explaining it to a developer
*"Right now this upload endpoint trusts whatever filename and extension the user's browser sends — but that's completely attacker-controlled, so it can't be trusted at all. The fix has three parts: check what the file actually is by reading its content, not its name; generate a brand-new filename ourselves so nothing the attacker chose survives; and store the file somewhere the web server can't directly execute, even as a backup in case something still gets through the first check."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Content-based validation (magic bytes) | Files lying about their own type via extension |
| Server-generated filenames | Path traversal, double-extension tricks |
| Storage outside web root | Even a successful bad upload can't be directly executed |
| Correct Content-Type / Content-Disposition | Uploaded files rendering as HTML/script instead of downloading |
| File size limits | Storage-exhaustion denial of service |
