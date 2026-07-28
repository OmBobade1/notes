# Insufficient Logging & Monitoring

## Why this comes last, and why it's a different kind of category entirely
Every one of the previous 29 files was about **prevention** — stopping a vulnerability from being exploitable in the first place. This one is different: it's about **detection** — assuming, realistically, that something will eventually get through despite every prevention layer, how quickly would anyone actually notice? For a bank specifically, this is arguably as important as prevention itself, since real-world fraud detection depends entirely on this category.

---

## What it is (in plain terms)
Insufficient logging and monitoring means security-relevant events either aren't being logged at all, are logged but never actually reviewed or alerted on, or take so long to be noticed that an attacker has ample time to complete their objective and cover their tracks before anyone responds. This isn't a single code-level flaw like the previous 29 — it's an organizational and architectural gap.

## What "insufficient" actually looks like in practice

**Missing security-relevant events entirely:**
```python
# VULNERABLE — login failures aren't logged at all
@app.route('/login', methods=['POST'])
def login():
    if not authenticate(request.form['username'], request.form['password']):
        return "Invalid credentials", 401  # failure happens silently, no record kept
```
Without logging failed login attempts, there's no way to detect a brute-force or credential-stuffing attack (file `03`) in progress — the very first signal of that attack is simply never captured.

```python
# SECURE — security-relevant events explicitly logged with enough context to investigate later
@app.route('/login', methods=['POST'])
def login():
    if not authenticate(request.form['username'], request.form['password']):
        security_logger.warning(
            f"Failed login attempt for username={request.form['username']}, "
            f"ip={request.remote_addr}, timestamp={datetime.utcnow()}"
        )
        return "Invalid credentials", 401
```

**Events logged, but nobody's watching:** Many organizations do log extensively — but if those logs simply accumulate in storage with no automated alerting and no one regularly reviewing them, the practical detection value is close to zero. A log entry that's never read might as well not exist for detection purposes, even though it exists for later forensic investigation.

**No correlation across systems:** A single failed login might mean nothing. A thousand failed logins across a thousand different accounts, all from the same source IP within a short window (a hallmark of credential stuffing), is a clear attack pattern — but only if something is actually correlating events across many individual log entries rather than looking at each in isolation.

## What real-world attacks depend on this gap
Many of the most damaging breaches in the real world weren't discovered by the victim organization's own monitoring at all — they were discovered by a third party (a security researcher, a payment processor noticing unusual chargeback patterns, or law enforcement) weeks or months after the actual intrusion. The gap between "attacker got in" and "organization noticed" is often the single biggest factor determining how much damage a breach ultimately causes — every day of unnoticed access is another day of potential data exfiltration, fraud, or lateral movement.

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | The single biggest lever on the ultimate cost of any breach is how quickly it's detected — an intrusion caught in hours costs vastly less than the same intrusion running undetected for months, since fraud and data exfiltration continue the entire time detection is delayed |
| **Regulatory / compliance** | Banking regulators specifically expect fraud-detection and security-monitoring capability as a baseline control — insufficient logging isn't just a technical gap, it's frequently an explicit, named compliance requirement (e.g. maintaining audit trails for a minimum retention period, real-time transaction monitoring for fraud patterns) |
| **Reputational damage** | "We discovered this ourselves within hours and contained it immediately" is a fundamentally different public narrative than "a security researcher informed us of a breach that had been ongoing for four months" — detection speed directly shapes how a breach is perceived publicly, independent of the original vulnerability's severity |
| **Legal liability** | Extended undetected access, when logging that would have caught it earlier was genuinely feasible and simply wasn't implemented, strengthens claims that the organization failed a basic, reasonably expected security control |
| **Operational cost** | Late detection means a much larger forensic investigation scope — reconstructing what happened over months, across every system potentially touched, is dramatically more expensive than investigating a contained, quickly-caught incident |

**One-line interview answer:** *"Technically, insufficient logging and monitoring means either security events aren't captured at all, or they are but nobody's actually watching or correlating them. For a bank, this is uniquely important because the biggest single factor in how damaging a breach turns out to be is how fast it's detected — an intrusion caught in hours costs a fraction of the same intrusion running undetected for months, and regulators specifically expect real-time fraud and security monitoring as a baseline control, not an optional extra."*

## What should actually be logged (security-relevant events)
- All authentication events — successes *and* failures, with source IP and timestamp
- Access control failures (attempted access to unauthorized resources/functions — connects to files `07` and `23`)
- High-value transactions and administrative actions (funds transfers above a threshold, account modifications, admin panel access)
- Input validation failures that suggest attack attempts (repeated malformed requests matching known attack patterns)

## Mitigation

1. **Log all security-relevant events with sufficient context** (who, what, when, from where) — not just errors, but successes too, since a pattern of successful-but-unusual actions can itself be the signal (e.g. one account suddenly transferring funds at 3am from a new location).
2. **Centralize logs from all systems** into a single, searchable platform (a SIEM — Security Information and Event Management system) rather than leaving logs scattered across dozens of individual servers/services with no unified view.
3. **Implement automated alerting on known attack patterns** — repeated failed logins, access-control failures, unusual transaction volume/timing — rather than relying solely on manual log review, which doesn't scale.
4. **Protect log integrity** — logs themselves should be tamper-resistant (write-once storage, or shipped immediately to a separate system an attacker with a foothold can't easily reach) so an attacker who gains access can't simply delete the evidence of their own activity.
5. **Define and practice an incident response process** — logging and alerting are only useful if there's a clear, practiced process for what happens once an alert fires; detection without response is only half the solution.

## Explaining it to a developer
*"Right now, a failed login just returns an error message with nothing recorded about it — which means if someone's actively trying to brute-force accounts, there's no record anywhere that would let anyone notice it's happening. The fix isn't complicated on the code side — log the attempt with enough context (username attempted, source IP, timestamp) to actually investigate later — but the bigger point is that logging by itself isn't enough. Someone or something needs to actually be watching for patterns in those logs, or we're just writing a diary nobody reads."*

## Quick-reference table

| Practice | Why it matters |
|---|---|
| Log all auth events (success + failure) | First signal for brute-force/credential-stuffing detection |
| Centralize logs (SIEM) | Enables correlation across systems, not isolated per-server visibility |
| Automated alerting on known patterns | Doesn't rely on manual review, which doesn't scale |
| Tamper-resistant log storage | Prevents an attacker from erasing evidence of their own activity |
| Practiced incident response process | Detection without a response plan doesn't actually limit damage |

---

## This completes the expanded web vulnerabilities set (30 files)
Files `01` through `30` now form a genuinely comprehensive walkthrough: passive inspection (headers, cookies) → authentication/session mechanics → the full injection family (SQLi through NoSQL/SSTI/CRLF) → access control (IDOR and function-level) → client-side risks (postMessage, clickjacking) → infrastructure hygiene (TLS, misconfiguration, secrets, third-party scripts) → availability (DoS) → and finally detection itself, closing the loop on what happens if every prior layer is somehow bypassed.
