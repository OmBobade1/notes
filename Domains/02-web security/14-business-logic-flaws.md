# Business Logic Flaws

## Why this comes last in this core set
Every vulnerability so far has been about a technical mistake — missing encoding, missing validation, an unsafe function call. Business logic flaws are different: **the code works exactly as written, with no technical bug at all** — the flaw is in the *design* of the workflow itself, which never anticipated a user doing things out of order, with unexpected values, or faster than intended. This is why automated scanners are genuinely bad at finding these — there's no malformed input or error message to detect, just a legitimate-looking sequence of requests used in an unintended way. This category is often the most relevant to banking specifically, since financial workflows (transfers, discounts, limits) are exactly where this shows up.

---

## Price / Amount Manipulation

**What it is:** Trusting a value (price, transfer amount, discount) that the client sends back, instead of re-verifying it server-side against the actual source of truth.

**Example scenario:**
```
POST /checkout
{
  "item_id": 42,
  "price": 999.00   <-- client claims this is the price
}
```
```python
# VULNERABLE — trusts the price the client sent
def checkout(item_id, price):
    charge_customer(price)  # uses whatever the client claims the price is
```
If the actual price of item 42 is meant to be looked up server-side, but the code trusts the `price` field sent in the request instead, an attacker can simply change that number to `0.01` before submitting.

```python
# SECURE — server looks up the real price itself, ignores any client-supplied price
def checkout(item_id, price):
    real_price = db.get_item_price(item_id)  # source of truth, never trust the client's number
    charge_customer(real_price)
```

**Business Impact:** Direct, immediate financial loss — every transaction processed at the manipulated price is real money lost, and this is trivially automatable once found (script the same request repeatedly).

---

## Race Conditions

**What it is:** A vulnerability that only appears when multiple requests hit the same code path at nearly the same time, and the application doesn't properly handle that simultaneity — most commonly seen in anything involving a balance check followed by a balance update as two separate steps.

**Example scenario:**
```python
# VULNERABLE — check and update are two separate steps, not atomic
def withdraw(account_id, amount):
    balance = db.get_balance(account_id)   # Step 1: check
    if balance >= amount:
        db.set_balance(account_id, balance - amount)  # Step 2: update
```
If an attacker fires off 10 withdrawal requests for the account's entire balance at the exact same moment, it's possible for multiple requests to all read the *same* starting balance during Step 1 (before any of them have finished Step 2), and all pass the `balance >= amount` check — resulting in far more money withdrawn than the account ever actually held.

```python
# SECURE — atomic operation, check and update happen as one indivisible database transaction
def withdraw(account_id, amount):
    with db.transaction():
        result = db.execute(
            "UPDATE accounts SET balance = balance - %s WHERE id = %s AND balance >= %s",
            (amount, account_id, amount)
        )
        if result.rows_affected == 0:
            raise InsufficientFundsError()
```
Here, the check and the update happen as a single atomic database operation — the database itself guarantees no two simultaneous requests can both succeed against the same starting balance.

**Business Impact:** This is one of the most directly financially damaging categories on this entire list for a bank specifically — race conditions in withdrawal, transfer, or balance-check logic can allow withdrawing more money than an account actually contains, multiplied across however many simultaneous requests an attacker can fire.

---

## Workflow / Step-Skipping

**What it is:** A multi-step process (e.g. checkout → payment → confirmation, or KYC verification → account activation) where later steps don't actually verify that earlier steps were genuinely completed — they just trust that if you're calling step 3, steps 1 and 2 must have happened.

**Example scenario:** A loan application process has three steps: `/apply/details`, `/apply/verify-income`, `/apply/approve`. If `/apply/approve` doesn't check that income verification was actually completed and passed, a user (or attacker) can call `/apply/approve` directly, skipping the verification step entirely.

**Business Impact:** For banking specifically, this could mean loan approval or account opening without required verification steps ever actually happening — a direct compliance violation (KYC/AML requirements exist precisely to prevent this), not just a workflow inconvenience.

---

## Negative Value / Quantity Abuse

**What it is:** Not validating that a numeric input is within a sane, expected range — particularly whether it's allowed to be negative when negative genuinely makes no sense for that field.

**Example scenario:**
```python
# VULNERABLE — no check that quantity is positive
def transfer_funds(from_account, to_account, amount):
    db.update_balance(from_account, -amount)
    db.update_balance(to_account, +amount)
```
If `amount` is allowed to be negative, submitting `amount = -1000` reverses the entire operation: it *adds* 1000 to the attacker's own account and *subtracts* 1000 from the victim's — turning a "transfer out" feature into a way to pull funds from someone else's account.

```python
# SECURE — explicitly validates amount is positive before proceeding
def transfer_funds(from_account, to_account, amount):
    if amount <= 0:
        raise InvalidAmountError()
    db.update_balance(from_account, -amount)
    db.update_balance(to_account, +amount)
```

**Business Impact:** Direct, unauthorized fund movement between accounts — arguably the clearest possible example of "the code works exactly as designed, but the design never considered a negative number," resulting in genuine financial loss.

---

## Quick-reference table

| Flaw | Root Cause | Business Impact |
|---|---|---|
| Price/Amount Manipulation | Trusting a client-supplied value instead of the server's own source of truth | Direct financial loss, easily automated |
| Race Conditions | Check-then-update logic that isn't atomic | Withdrawing more than an account actually holds |
| Workflow/Step-Skipping | Later steps don't verify earlier steps genuinely completed | Compliance violations (e.g. bypassed KYC/verification) |
| Negative Value Abuse | No validation that a quantity must be positive | Unauthorized fund transfer between accounts |

## Why this category matters most in an interview
Every prior vulnerability in this series has a clear technical signature — a payload, an error message, something a scanner can flag. Business logic flaws have none of that; the request looks completely legitimate at every layer. Being able to say *"I don't just look for broken input handling — I think about what a legitimate feature could be abused to do if used out of order, with unexpected values, or simultaneously"* demonstrates a level of thinking that goes beyond running a scanner, which is exactly the distinction between someone who can operate a tool and someone who actually understands security.

## Mitigation — the common thread across all four
1. **Never trust client-supplied values for anything that determines money movement** — always re-verify against the server's own source of truth.
2. **Make check-then-act sequences atomic** — use database transactions/locking so simultaneous requests can't both succeed against the same starting state.
3. **Verify prior steps were genuinely completed**, server-side, before allowing a later step to proceed — don't assume sequence based on which endpoint was called.
4. **Validate that numeric inputs make business sense** — not just "is this a number," but "is this number within the range that's actually valid for this specific field" (e.g. quantities and amounts should almost always be required to be positive).
