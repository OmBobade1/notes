# Hardcoded Secrets

## Why this comes right after Sensitive Data Exposure
That file covered data being inadequately protected once collected. This one is about a specific, extremely common variant of the same problem: the credentials/keys needed to *access* sensitive systems being embedded directly in source code, config files, or client-side JavaScript — rather than being protected at all.

---

## What it is (in plain terms)
Hardcoded secrets are API keys, database passwords, encryption keys, or cloud credentials written directly into source code (as a literal string) instead of being loaded from a secure, separate configuration source at runtime. Once committed to a codebase — especially one under version control — the secret effectively becomes permanent history, discoverable by anyone with access to the repository, even long after the line is "removed" in a later commit.

## Why it exists — the real-life cause

```python
# VULNERABLE — real database credentials written directly into the source file
DB_HOST = "prod-db.internal.bank.com"
DB_USER = "app_admin"
DB_PASSWORD = "Summer2024!"  # committed directly into version control
```
Anyone with read access to this repository — a current employee, a former employee whose access wasn't fully revoked, or anyone who gains access to the repository through any other means — now has the actual production database password in plain text. If the repository is ever made public (even briefly, or through a misconfigured permission), or if it's exposed via one of the misconfiguration issues in file `15` (an exposed `.git` folder), the secret is fully compromised.

```python
# SECURE — credentials loaded from environment variables at runtime, never in source code
import os

DB_HOST = os.environ.get("DB_HOST")
DB_USER = os.environ.get("DB_USER")
DB_PASSWORD = os.environ.get("DB_PASSWORD")  # actual value lives in the deployment environment, not the code
```
The actual secret value now lives in the deployment environment's configuration (set via the hosting platform, a secrets manager, or a `.env` file explicitly excluded from version control) — the source code itself contains no sensitive value at all, so anyone who reads the code learns nothing.

## The particularly dangerous variant: client-side secrets

```javascript
// VULNERABLE — API key embedded directly in frontend JavaScript
const STRIPE_SECRET_KEY = "sk_live_51H8x...";  // this is the SECRET key, meant for server-side use only
fetch('https://api.stripe.com/charges', {
  headers: { 'Authorization': `Bearer ${STRIPE_SECRET_KEY}` }
});
```
Client-side JavaScript is, by definition, delivered to and readable by every visitor's browser — there is no such thing as a "hidden" secret in frontend code, since anyone can view the page source or the network requests directly. A secret key embedded this way is effectively public the moment the page is deployed, regardless of any obfuscation/minification applied (which slows down but never prevents extraction).

## How an attacker actually does it, step by step
1. **Source code access:** if a repository is public, briefly misconfigured, or exposed via a `.git` folder leak (file `15`), search the entire commit history — not just the current state — for patterns resembling API keys, passwords, or connection strings. Automated tools (e.g. `gitleaks`, `trufflehog`) scan repositories specifically for this.
2. **Client-side exposure:** simply view the page source or inspect network requests in browser developer tools — any secret embedded in frontend code is directly visible with zero special tooling required.
3. Use the discovered credential directly against the service it belongs to — a database connection string grants direct database access, a cloud API key grants whatever permissions that key holds in the cloud account.

## Technical Impact
- Direct, valid access to whatever system the secret protects — a database, a cloud account, a third-party API — with no further exploitation needed once the secret is found
- Secrets committed to version control remain discoverable in history indefinitely, even after being "removed" in a later commit, unless the repository history itself is rewritten (a disruptive, rarely-done remediation step)

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | A leaked cloud credential or payment-processor secret key can be used directly to access billing systems, process fraudulent charges, or reach the full scope of whatever that credential is authorized for — often far broader than the original application itself |
| **Regulatory / compliance** | Hardcoded secrets discovered during a security assessment are consistently flagged as a basic, foundational finding — their presence often suggests broader gaps in secure development practices, not an isolated oversight |
| **Reputational damage** | Public exposure of hardcoded secrets (e.g. via a public GitHub repository) is a common, easily-discoverable finding that security researchers and automated bots actively scan for continuously across the entire internet — discovery by an outside party rather than internally is a realistic, frequent scenario |
| **Legal liability** | An avoidable, well-known bad practice (secrets belong in a secrets manager or environment variables, not source code) that directly enabled unauthorized access is difficult to characterize as anything but negligence |
| **Operational cost** | Requires immediately rotating every exposed credential, auditing exactly what that credential had access to and whether it was actually used maliciously, and — for version-controlled secrets — potentially rewriting repository history entirely, which is disruptive for every developer working on that codebase |

**One-line interview answer:** *"Technically, hardcoded secrets means credentials or API keys are written directly into source code instead of loaded from a secure runtime configuration — and once committed to version control, they remain discoverable in history indefinitely. For a bank, the real danger is scope: a single leaked cloud or payment-processor credential can grant access far broader than just the original application, so remediation means rotating the secret and auditing everything it could have touched, not just fixing the code."*

## Mitigation

1. **Never write real secrets directly into source code (the real fix)** — load them from environment variables, a dedicated secrets manager (AWS Secrets Manager, HashiCorp Vault), or a `.env` file explicitly excluded from version control via `.gitignore`.
2. **Automated secret scanning in CI/CD** — tools like `gitleaks` or `trufflehog` run automatically on every commit/pull request, catching accidentally committed secrets before they're merged, and can also scan existing repository history to find what's already been exposed.
3. **Rotate any secret that was ever committed to version control**, even if later removed — treat it as compromised permanently, since history retains it regardless of subsequent "removal."
4. **Never place secrets in client-side/frontend code under any circumstance** — anything the browser can access is public by definition; secret operations must happen server-side, with the frontend only ever calling your own backend, which then holds the actual secret.
5. **Least-privilege for every credential** — scope each secret to only the specific permissions it actually needs, limiting the damage if it's ever exposed despite the above precautions.

## Explaining it to a developer
*"This database password is written directly in the source file, which means anyone with read access to this repository — now or at any point in the future, since it stays in git history even after being removed — has the real credential. The fix is to load it from an environment variable instead, with the actual value set only in the deployment environment, never committed anywhere. And separately, this API key needs to never appear in frontend JavaScript at all — anything sent to the browser is visible to every visitor, with no exceptions, regardless of how it's obfuscated."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Environment variables / secrets manager | Secrets never appear in source code at all |
| Automated secret scanning (gitleaks, trufflehog) | Catches accidental commits before/after merge |
| Rotate any ever-committed secret | Treats history exposure as permanent compromise |
| Never embed secrets in client-side code | Removes the "anyone can view source" exposure entirely |
| Least-privilege scoping per credential | Limits damage if a secret is exposed regardless |
