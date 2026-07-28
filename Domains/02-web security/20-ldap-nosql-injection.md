# LDAP & NoSQL Injection

## Why this comes right after Clickjacking, and connects back to file 04
This is the exact same root cause as SQL Injection — untrusted input concatenated directly into a query — just against a different kind of database. If SQLi made sense, this one will too; it's the same lesson applied to LDAP directories (often used for corporate/employee authentication) and NoSQL databases (like MongoDB, common in modern web APIs).

---

## LDAP Injection

**What it is:** LDAP (Lightweight Directory Access Protocol) is often used for authentication against a corporate directory (e.g. checking employee credentials). LDAP Injection happens the same way SQLi does — user input gets concatenated directly into an LDAP query filter, letting an attacker change the query's logic.

**Example:**
```python
# VULNERABLE — username concatenated directly into the LDAP filter
username = request.form['username']
ldap_filter = f"(&(uid={username})(objectClass=person))"
```
A normal login sends `username=jdoe`, producing `(&(uid=jdoe)(objectClass=person))`. But if an attacker submits `*)(uid=*))(|(uid=*` as the username, the filter's logic changes entirely — LDAP wildcard characters (`*`) and logical operators can be injected the same way SQL syntax can, potentially matching every user in the directory or bypassing the intended filter.

```python
# SECURE — escape special LDAP filter characters before use
import ldap.filter
username = request.form['username']
safe_username = ldap.filter.escape_filter_chars(username)
ldap_filter = f"(&(uid={safe_username})(objectClass=person))"
```
Here, characters with special meaning in LDAP filter syntax (`*`, `(`, `)`, `\`, null byte) are escaped before being inserted, so they're treated as literal characters to search for, not as filter syntax.

**Business Impact:** For a bank's internal systems, LDAP often authenticates employees and internal applications against corporate Active Directory — LDAP injection here could mean bypassing employee authentication entirely, a direct path toward internal systems and, from there, potentially customer-facing systems as well (this connects to the Active Directory domain covered separately in the network-security-labs repo).

---

## NoSQL Injection

**What it is:** NoSQL databases (MongoDB being the most common example) don't use SQL syntax, so traditional SQLi payloads don't apply — but they're vulnerable to a different form of the same underlying problem: passing structured objects (not just strings) as input, which can override the intended query logic entirely.

**Example:**
```javascript
// VULNERABLE — directly uses whatever the client sends as the query condition
const user = await db.collection('users').findOne({
  username: req.body.username,
  password: req.body.password
});
```
A normal login sends `{"username": "jdoe", "password": "correcthorse"}` as JSON. But if an attacker sends `{"username": "jdoe", "password": {"$ne": null}}` instead, MongoDB's `$ne` (not-equal) operator turns the password check into "match any password that is not null" — which is true for basically any account with a password set at all, bypassing authentication without knowing the actual password.

```javascript
// SECURE — explicitly cast/validate input types before using them in a query
const username = String(req.body.username);
const password = String(req.body.password);  // forces the object-injection attempt to become a harmless string instead
const user = await db.collection('users').findOne({ username, password });
```
By explicitly casting the input to a string type before it reaches the query, an attacker's injected object (`{"$ne": null}`) gets converted to a harmless string representation instead of being interpreted as a query operator.

**Business Impact:** Same authentication-bypass severity as SQLi's `' OR '1'='1` — for a banking application built on a NoSQL backend (common in newer microservices architectures), this is a direct login bypass, translating to the exact same account-takeover and regulatory risk already covered for SQL Injection in file `04`.

## How an attacker actually does it, step by step
1. Identify the backend technology — LDAP is common for enterprise/corporate authentication, NoSQL (especially MongoDB) is common in modern JavaScript/Node.js-based APIs.
2. For LDAP: try wildcard and filter-breaking characters (`*`, `)`, `(`, `|`) in login fields.
3. For NoSQL: try sending JSON objects instead of plain strings in fields the frontend normally sends as text — e.g. changing a form submission to send `{"$ne": null}` or `{"$gt": ""}` as the password value directly via an intercepted request (Burp Suite).
4. If authentication succeeds without valid credentials, injection is confirmed — from there, the same escalation path as SQLi applies (extracting data, bypassing further checks).

## Mitigation — the same core lesson as SQLi, applied to each format

1. **LDAP:** always escape special filter characters using the language/library's built-in escaping function — never build filter strings via raw concatenation.
2. **NoSQL:** explicitly validate and cast input types before using them in a query — a field expected to be a string should never be allowed to remain a JSON object/array by the time it reaches the database call. Many NoSQL drivers/ORMs also offer built-in query-builder methods that avoid this risk the same way parameterized queries do for SQL.
3. **Input validation as a backup layer** in both cases — reject unexpected characters/types outright where the expected format is known, as defense-in-depth.

## Explaining it to a developer
*"This is really the same lesson as SQL injection, just for a different kind of database. For LDAP, special characters in the filter syntax need to be escaped before use, the same way SQL syntax characters do. For NoSQL like MongoDB, the specific risk is different — it's about making sure the fields we expect to be simple text can't secretly be JSON objects instead, since operators like `$ne` can hijack the query's logic entirely if we don't explicitly force the input into the type we expect."*

## Quick-reference table

| Type | Injection mechanism | Fix |
|---|---|---|
| LDAP Injection | Special filter characters (`*`, `(`, `)`, `\|`) alter query logic | Escape filter characters before use |
| NoSQL Injection | JSON operators (`$ne`, `$gt`, etc.) sent instead of plain strings | Explicitly cast/validate input type before querying |
