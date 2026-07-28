# SQL Injection

## What it is (in plain terms)
A login form, search box, or any input field usually sends your text straight into a database question (a "query"). SQL Injection happens when that text gets pasted directly into the query instead of being treated as pure data — so instead of just searching for what you typed, the database ends up running instructions the attacker snuck in.

## Why it exists — the real-life cause
It exists because of how the query is built in the code. Look at the difference:

```python
# VULNERABLE
username = request.form['username']
query = "SELECT * FROM users WHERE username = '" + username + "'"
cursor.execute(query)
```
Here, whatever the user types gets glued directly into the query text. The database can't tell "this is just a name to search for" from "this is a new instruction" — it just sees one long string and runs it.

```python
# SECURE
username = request.form['username']
query = "SELECT * FROM users WHERE username = %s"
cursor.execute(query, (username,))
```
Here, the query shape (`%s`) and the actual value are sent to the database **separately**. No matter what the user types — even `' OR '1'='1` — the database treats it only as a value to search for, never as new instructions. This is called a **parameterized query**, and it's the actual fix, not just a patch.

## Payload alternatives (what you'd actually type in)

| Payload | What it does |
|---|---|
| `' OR '1'='1` | Makes the WHERE condition always true — bypasses a login check entirely |
| `admin'--` | Comments out the rest of the query (password check included) — logs in as `admin` with no password needed |
| `' UNION SELECT username, password FROM users-- ` | Pulls data from another table into the visible results |
| `'; DROP TABLE users;--` | Ends the original query and runs a second, destructive one |
| `' AND SLEEP(5)-- ` | No visible change on screen, but a 5-second delay proves the injection worked ("blind" injection) |

## How an attacker actually does it, step by step
1. Find a field that talks to a database — login box, search bar, or a URL like `page.php?id=5`.
2. Type a single quote (`'`) into it. A database error message back is the first sign it's vulnerable.
3. Try `' OR '1'='1` — if a login suddenly works with a fake password, injection is confirmed.
4. Escalate: use UNION-based payloads to pull data directly, or automated tools like SQLmap to dump entire databases fast.

## Technical Impact (what the database itself suffers)
- Login bypass — accessing an account, or an admin account, with no real password
- Data theft — dumping usernames, password hashes, personal details, payment data
- Data destruction — deleting or altering entire tables
- Full server takeover — using database features to run OS-level commands on the server

## Business Impact — this is the answer an interviewer actually wants
Technical impact is only half the answer. What an interviewer at a bank is really asking is: **"translate that into money, regulation, and trust — what does this cost us?"**

Using a banking application as the example:

| Angle | What it actually means |
|---|---|
| **Financial loss** | If the vulnerable query sits behind a funds-transfer or account-balance feature, an attacker bypassing auth could initiate unauthorized transactions or view/manipulate account balances directly — direct monetary loss, not just "data theft" |
| **Regulatory / compliance** | Banks are bound by standards like PCI-DSS (if card data is touched) and country-specific regulators (e.g. RBI in India, FFIEC in the US). A confirmed SQLi breach exposing customer data triggers mandatory breach disclosure, regulatory investigation, and potential fines — this is a legal/compliance event, not just an IT incident |
| **Reputational damage** | Customers trust a bank specifically with their money and data. A public breach disclosure directly drives account closures and makes new customer acquisition harder — trust, once publicly broken, is expensive to rebuild |
| **Legal liability** | Affected customers (or shareholders, if it hits stock price) can pursue legal action against the bank for failing to protect their data under applicable data protection law |
| **Operational cost** | Incident response, forensic investigation, customer notification, credit monitoring services for affected customers, and post-incident security overhaul are all direct, often multi-million-rupee/dollar costs — separate from any fines |

**The one-line version to give in an interview:** *"Technically, it's [login bypass / data theft / whatever the specific finding is] — but the business impact is that it could lead to unauthorized transactions or exposed customer financial data, which triggers regulatory reporting obligations, potential fines, and real reputational and financial cost to the bank, well beyond the technical fix itself."*

This structure — technical impact, then translate to financial/regulatory/reputational/legal/operational — is the one to reuse for every vulnerability, not just this one.

## Mitigation — more than one layer, not just one fix

1. **Parameterized queries / prepared statements (the real fix)** — shown above. This alone stops the vast majority of SQL injection.
2. **ORM frameworks** (e.g. SQLAlchemy, Django ORM) — build queries safely under the hood, so raw SQL is rarely written by hand at all.
3. **Least privilege database accounts** — the app's database login should only be allowed to do what it actually needs (it should never have permission to `DROP` a table just because a login form uses that connection).
4. **Input validation as a backup layer** — e.g. if a field should only ever be a number, reject anything that isn't. Doesn't replace parameterized queries, just adds an extra wall.
5. **Web Application Firewall (WAF)** — catches known attack patterns before they reach the app, but should never be the only defense — it's a filter, not a fix for the underlying code.

## Explaining this to a developer who doesn't know the fix
If you're the one who found this and the developer on the other end doesn't know SQL injection: don't lead with the exploit — lead with the one-line root cause. *"Right now the query is built by joining the user's input directly into the SQL string — that means whatever the user types can change what the query actually does. The fix is to use a parameterized query instead, where the query's structure is fixed and the user's input is only ever treated as a value, never as part of the instruction."* Then show the two code blocks side by side, exactly as above — most developers get it immediately once they see the concatenation vs. parameter side by side.
