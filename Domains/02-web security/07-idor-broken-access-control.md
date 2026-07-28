# Broken Access Control (IDOR)

## Why this comes right after CSRF
CSRF is about forging *who* is making a request — tricking the browser into sending it. Broken Access Control is a different problem: the request is genuinely coming from a real, logged-in user — just not for the resource they're supposed to be limited to. The server correctly confirms "you are logged in" but never checks "should *you* specifically be allowed to see *this* particular record."

---

## What it is (in plain terms)
Insecure Direct Object Reference (IDOR) is the most common form of broken access control. It happens when an application uses a direct, guessable reference (like a plain numeric ID) to fetch a record, and never checks whether the logged-in user actually owns or is allowed to access that specific record — it only checks that *some* valid user is logged in.

## Why it exists — the real-life cause

```python
# VULNERABLE — fetches whatever account_id is given, no ownership check
@app.route('/account/<account_id>/statement')
def get_statement(account_id):
    if session.get('user_id'):  # only checks: is SOMEONE logged in?
        return db.get_statement(account_id)
```
A logged-in user views their own statement at `/account/1001/statement`. Nothing stops them from simply editing the URL to `/account/1002/statement` — the code never checks whether account `1002` actually belongs to the logged-in user, only that *a* valid session exists.

```python
# SECURE — verifies the requested resource actually belongs to the logged-in user
@app.route('/account/<account_id>/statement')
def get_statement(account_id):
    user_id = session.get('user_id')
    account = db.get_account(account_id)
    if not user_id or account.owner_id != user_id:
        abort(403)  # logged in, but not authorized for THIS resource
    return db.get_statement(account_id)
```
Here, even a fully valid, logged-in user is rejected if the specific `account_id` they're requesting doesn't actually belong to them.

## How an attacker actually does it, step by step
1. Log in normally with a real, legitimate account.
2. Find any URL or API call that references a record by a direct, visible ID — `/account/1001/statement`, `/invoice/5521`, `/user/302/profile`.
3. Change the ID to a nearby or guessed value (`1002`, `5522`, `303`) and resend the request.
4. If the response returns someone else's data instead of an access-denied error, access control is broken — this can then be automated to enumerate every ID in sequence, pulling every customer's data in bulk.

## Technical Impact
- Viewing, modifying, or deleting other users' data by simply changing an ID
- At scale, sequential IDs allow bulk automated enumeration of every record in the system, not just one victim

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | If the exposed resource is an account statement, balance, or transaction history, this is a direct, large-scale privacy breach — and if the endpoint allows modification (not just viewing), it can enable unauthorized transactions across many accounts at once, not just one |
| **Regulatory / compliance** | Mass, automatable exposure of customer financial records via a simple ID change is one of the most severe possible findings in a banking security review — it demonstrates a systemic access-control failure, not an isolated bug, which regulators and auditors treat far more seriously than a single-input flaw |
| **Reputational damage** | Because this is easily automatable, a single unpatched IDOR can affect thousands of customers' data simultaneously — breach disclosure at that scale is a major public event, not a quiet fix |
| **Legal liability** | Mass exposure of financial records is exactly the scenario class-action lawsuits and regulatory penalties are built around — the scale itself increases legal exposure disproportionately compared to a single-user vulnerability |
| **Operational cost** | Because so many records may be affected, the investigation has to determine exactly which accounts were actually accessed (not just theoretically exposed) — often requiring extensive log analysis, far more costly than a single-target incident |

**One-line interview answer:** *"Technically, IDOR means the server checks that you're logged in, but never checks that the specific record you're requesting actually belongs to you — so changing an ID in the URL can expose someone else's data. For a bank, this is especially severe because it's automatable at scale: an attacker isn't limited to one victim, they can potentially enumerate every customer's account by simply incrementing an ID, which is a mass-breach scenario, not an isolated one."*

## Mitigation — layered, not just one fix

1. **Server-side ownership/authorization checks on every request (the real fix)** — for every record accessed, verify the logged-in user is actually authorized for that specific record, not just that they're logged in at all. This check must happen on the server, every single time — never trust a hidden field or the frontend UI to enforce this.
2. **Use indirect references** — instead of exposing sequential database IDs directly in URLs, use non-guessable identifiers (UUIDs) or a per-user mapping table, so even attempting to guess a nearby ID doesn't work.
3. **Centralize authorization logic** — implement access checks in one shared, well-tested layer (middleware/decorator) applied consistently across all endpoints, rather than re-writing the check by hand in every single route — a missed check in just one endpoint is enough to reintroduce the vulnerability.
4. **Deny by default** — the default behavior for any unmatched or unchecked case should be to reject access, not allow it.

## Explaining it to a developer
*"This endpoint checks that a user is logged in, but it never checks that the account ID they're asking for actually belongs to them. Right now, any logged-in customer could view another customer's data just by changing a number in the URL. The fix isn't complicated — before returning any record, confirm the record's owner matches the logged-in user's ID, and reject the request if it doesn't. The important part is that this check has to happen on every single endpoint that touches user-specific data, not just the ones that seem obviously sensitive."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Server-side ownership check | Accessing another user's record via ID manipulation |
| Indirect references (UUIDs) | Even attempting to guess nearby valid IDs |
| Centralized authorization logic | A missed check on one endpoint reintroducing the flaw |
| Deny by default | Any unhandled edge case defaulting to safe, not exposed |
