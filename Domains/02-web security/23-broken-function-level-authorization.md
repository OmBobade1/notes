# Broken Function-Level Authorization

## Why this comes right after CRLF Injection, and how it differs from IDOR (file 07)
IDOR (file `07`) is about *records*: a logged-in regular user changing an ID to view someone else's data. Broken Function-Level Authorization is about *functions*: a logged-in regular user reaching an entire admin-only feature or endpoint that should never have been accessible to them at all — not someone else's data, but functionality meant for a different role entirely.

---

## What it is (in plain terms)
Applications often have role-based features — a regular customer role, and an admin/staff role with additional capabilities (viewing all users, issuing refunds, modifying system settings). Broken Function-Level Authorization happens when the server doesn't actually verify the caller's role before executing an admin-only action — it might hide the admin button in the regular user's interface, but if the underlying endpoint itself doesn't check the role server-side, a regular user can call it directly and it will simply work.

## Why it exists — the real-life cause

```python
# VULNERABLE — the admin panel button is just hidden in the UI for non-admins,
# but the actual endpoint never checks the role at all
@app.route('/admin/issue-refund', methods=['POST'])
def issue_refund():
    if session.get('user_id'):  # only checks: is SOMEONE logged in?
        process_refund(request.form['transaction_id'], request.form['amount'])
        return "Refund issued"
```
The frontend might only show an "Issue Refund" button to users with an `admin` role — but if the actual server-side endpoint only checks that *a* user is logged in (not *which kind* of user), any regular customer who simply knows or guesses the URL `/admin/issue-refund` can call it directly (e.g. via Burp Suite, or even a basic browser request), and it works exactly as if they were an admin.

```python
# SECURE — explicitly verifies the caller's role server-side, every time
@app.route('/admin/issue-refund', methods=['POST'])
def issue_refund():
    user = get_current_user()
    if not user or user.role != 'admin':
        abort(403)
    process_refund(request.form['transaction_id'], request.form['amount'])
    return "Refund issued"
```
Here, the server explicitly checks the actual role of the logged-in user before performing the action — hiding the button in the UI is now just a convenience for regular users, not the actual security control.

## How an attacker actually does it, step by step
1. Log in as a regular, low-privilege user.
2. Look for hints of admin-only functionality — URLs referenced in JavaScript source, API documentation, or predictable naming patterns (`/admin/...`, `/api/v1/manage/...`).
3. Call the suspected admin endpoint directly with the regular user's own session (using Burp Suite to craft the request manually, bypassing whatever UI restriction exists).
4. If the action succeeds despite the account not having admin privileges, Broken Function-Level Authorization is confirmed — this can then be used to perform any action the endpoint allows, using only a basic customer account.

## Technical Impact
- A regular user gaining access to entire categories of admin-level functionality — not just one record, but whole features (user management, refund processing, system configuration) meant for a different privilege level entirely

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | If the exposed function is something like issuing refunds, adjusting account balances, or waiving fees, a regular customer account gaining this capability is a direct, repeatable path to financial loss — and unlike a single-record IDOR, this can potentially be exploited against many accounts through one exposed function |
| **Regulatory / compliance** | Demonstrates a systemic gap between the frontend (which correctly hides the feature) and the backend (which never actually enforces it) — auditors treat this as an architectural authorization failure, similar in severity to IDOR but often broader in scope since it exposes entire functions rather than individual records |
| **Reputational damage** | Depends on what functionality is exposed — but any customer-accessible path to admin-level actions is a serious, headline-worthy finding if discovered by an actual attacker rather than an assessor |
| **Legal liability** | Same reasoning as IDOR — a clearly missing, straightforward server-side check is difficult to defend as reasonable security practice |
| **Operational cost** | Requires auditing every privileged endpoint across the entire application for the same missing check, not just the one discovered — this class of bug tends to appear in clusters if the underlying authorization framework itself is inconsistent |

**One-line interview answer:** *"Technically, Broken Function-Level Authorization means the server only hides an admin feature in the interface but never actually verifies the caller's role when the underlying endpoint is called directly. For a bank, if the exposed function is something like issuing refunds or adjusting balances, a basic customer account gaining that capability is a direct and potentially repeatable path to financial loss — and it tends to reveal a broader pattern across multiple endpoints, not just one."*

## Mitigation — layered, not just one fix

1. **Enforce role checks server-side on every privileged endpoint (the real fix)** — never rely on hiding a button or menu item in the frontend as the actual security control; the server must independently verify the caller's role every time, regardless of what the UI shows.
2. **Centralize authorization logic** — implement role checks in shared middleware/decorators applied consistently across all admin-level routes, rather than manually re-implementing the check in every individual endpoint (the same principle as centralizing ownership checks for IDOR in file `07`).
3. **Deny by default** — any endpoint without an explicit, defined authorization rule should reject the request, not allow it.
4. **Regularly audit the full list of privileged endpoints** against the authorization checks actually implemented, since new endpoints added over time can easily be missed if the process isn't systematic.

## Explaining it to a developer
*"The refund button might only be visible to admins in the interface, but that's just hiding it visually — it doesn't stop someone from calling the actual endpoint directly with their own account, since the button being hidden happens entirely in the browser, which the attacker fully controls. The server itself needs to check the caller's actual role every time this endpoint is hit, completely independent of whether the button was ever shown to them."*

## Quick-reference table

| Mitigation | What it stops |
|---|---|
| Server-side role check on every privileged endpoint | Direct endpoint calls bypassing hidden UI elements |
| Centralized authorization middleware | Inconsistent or missed checks across individual routes |
| Deny by default | Unhandled/new endpoints defaulting to safe, not exposed |
| Regular endpoint audits | Catches gaps introduced as the application grows over time |
