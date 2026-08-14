# Module 06 - Unrestricted Access to Sensitive Business Flows

## Why this is a genuinely distinct category, not just "more rate limiting"
Module 04 covered resource *consumption* — overwhelming server capacity. This is different: every individual request here is completely legitimate, correctly authenticated, and technically well-formed. The problem isn't that anything is broken — it's that a **legitimate business process was never designed with the assumption that it could be automated and run thousands of times per second by a script instead of performed occasionally by a human.**

---

## What it actually is
Certain API-exposed business flows are sensitive not because of a coding bug, but because of what happens when they're performed at a speed or scale no human could achieve manually — buying up an entire limited product stock in seconds, creating thousands of fake accounts to claim thousands of new-user promo bonuses, or posting thousands of fake reviews. The API works exactly as designed; the *business* was never designed to survive that action being automated.

## Real Example 1: Limited-Stock Purchasing (Scalping)

**The business flow:** a `POST /api/orders` endpoint lets an authenticated user purchase one unit of a limited-availability item.

**Why this becomes a problem at API scale:** a human clicking "buy" on a website can complete maybe one purchase every few seconds, limited by how fast they can physically click and fill out a form. A script calling the API directly has no such limit — it can attempt purchases as fast as the network allows, potentially completing hundreds of purchases before a single human customer's browser has even finished loading the product page.
```python
import requests
for i in range(500):
    requests.post("https://api.target.com/orders",
                   headers={"Authorization": f"Bearer {token}"},
                   json={"productId": "limited-item-001", "quantity": 1})
```
**Why this is a genuinely different category from BOLA or resource consumption:** nothing here is unauthorized (the token is real and valid), nothing here overwhelms server capacity in a way that constitutes a DoS (500 legitimate-looking purchase requests isn't a flood, from the server's perspective) — the *business* harm is entirely about fairness and availability to real customers, not a technical security boundary being crossed at all.

## Real Example 2: Promo Abuse via Mass Account Creation

**The business flow:** `POST /api/register` creates a new account, and new accounts automatically receive a signup bonus/discount code.

**Why this becomes exploitable at API scale:** a script can register thousands of accounts (using disposable email services, or minor variations of the same email address if the validation is weak) far faster than any human, claiming thousands of promo bonuses that were only ever intended for genuine new customers, one per real person.

## Real Example 3: Fake Review/Rating Manipulation

**The business flow:** an authenticated user can submit a product review via `POST /api/reviews`.

**Why this is a business-flow problem, not a technical one:** if account creation itself is automatable (Example 2) or a legitimate account can submit unlimited reviews with no reasonable cap, a script can flood a product with fake five-star (or, for a competitor's product, fake one-star) reviews — manipulating a business signal that real customers rely on, using entirely "valid" API calls the whole way through.

## How to actually think about identifying these during an assessment
The core question to ask about *every* sensitive business action an API exposes, independent of any specific technical vulnerability: **"What would happen if this exact same valid, authorized request were sent 10,000 times per second instead of once, by a human, occasionally?"** If the honest answer involves real business harm (unfair advantage, financial abuse, manipulated trust signals, exhausted limited resources), that flow needs specific protection — not because a technical rule was broken, but because the business process itself assumed human-scale usage.

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | Promo/bonus abuse at scale is a direct, measurable financial cost — thousands of fraudulently-claimed bonuses is real money paid out for zero genuine customer acquisition |
| **Regulatory / compliance** | For a banking context specifically, an unrestricted "open a new account and get a bonus" flow is a direct target for this kind of abuse, and regulators increasingly expect fraud controls around exactly this kind of promotional/incentive program |
| **Reputational damage** | A limited product/offer being instantly bought out by bots, leaving zero available to genuine customers, is a directly visible, frequently publicly complained-about customer experience failure |
| **Legal liability** | Weaker direct liability than a data breach, but promo abuse at scale can represent a material, quantifiable financial loss the business may seek to recover or must account for |
| **Operational cost** | Requires retroactively identifying and reversing/invalidating fraudulently obtained bonuses or accounts — a messy, manual cleanup process after the fact, much more costly than preventing it up front |

**One-line interview answer:** *"This category is different from a typical vulnerability because nothing is technically broken — every request is valid and authorized. The risk is that a legitimate business process, like claiming a signup bonus or buying limited stock, was only ever designed assuming human-scale usage, and an API makes that same valid action trivially automatable at a scale the business never accounted for."*

## Mitigation

1. **Behavioral rate limiting tied to the specific sensitive action**, not just general API rate limiting — a much stricter limit specifically on "create account" or "place order for limited item" than on general read-heavy endpoints.
2. **CAPTCHA or proof-of-work challenges specifically on sensitive business-flow endpoints** (registration, checkout for limited items) — genuinely raising the cost of automation for these specific flows, even if other endpoints don't need it.
3. **Device fingerprinting / anomaly detection** — flagging patterns humans don't produce (hundreds of account registrations from the same device/IP in minutes, purchase attempts with inhuman timing consistency).
4. **Explicit per-account, per-identity limits on promotional value** — one bonus per verified real identity (tied to something harder to fabricate at scale than an email address, like phone verification or payment method matching), not just "one per account."
5. **Design review at the business-logic level, not just the code level** — asking "what if this ran a thousand times a second" as a standard question during API design, before the endpoint ever ships.

## Quick-reference table

| Example | Business Harm | Key Mitigation |
|---|---|---|
| Limited-stock bulk purchasing | Unfair customer access, real customers locked out | Behavioral rate limiting, CAPTCHA on checkout |
| Mass account creation for promo abuse | Direct financial loss from fraudulent bonuses | Per-identity limits (not just per-account), stronger verification |
| Fake review/rating flooding | Manipulated trust signal, damaged customer confidence | Rate limiting reviews, anomaly detection on submission patterns |
