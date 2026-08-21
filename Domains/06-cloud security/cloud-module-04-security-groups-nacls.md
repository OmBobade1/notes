# Cloud Module 04 — Security Groups and NACLs (Network-Layer Cloud Attack Surface)

## Why this isn't "just a firewall" the way you already think about it

In your Network domain work, a firewall rule is something you probe from the outside and try to figure out through trial and error — closed ports, timeouts, inference. In AWS, security groups are **fully readable via the API by anyone with the right IAM permission**, the same way you read IAM policies in Module 01. You don't have to guess what's open. You can just ask.

```
aws ec2 describe-security-groups
```

This dumps every security group in the account, every inbound and outbound rule, as structured JSON. No port scanning required to know the rules exist — port scanning only becomes necessary if you don't have API access at all, which, if you're this deep into an account already, you usually do.

---

## Reading the output for the Lab 6 pattern

You're scanning each security group's `IpPermissions` (inbound rules) for this shape:

```json
{
  "IpProtocol": "tcp",
  "FromPort": 22,
  "ToPort": 22,
  "IpRanges": [{"CidrIp": "0.0.0.0/0"}]
}
```

Three things have to line up to be dangerous, and beginners often flag the wrong one:

- `FromPort`/`ToPort` = 22 (SSH) — or 3389 (RDP), or 3306 (MySQL), or 5432 (Postgres) — anything that shouldn't be internet-facing
- `CidrIp: 0.0.0.0/0` — this is the actual finding. It means "any IPv4 address on the internet," not "any address on this network." People sometimes misread this as a placeholder or internal-range notation. It isn't — it's the literal entire internet.
- The rule is **inbound**, not outbound. Outbound `0.0.0.0/0` on port 443 is completely normal (it's how the instance reaches the internet for updates, API calls, etc.) — don't flag outbound rules as findings by reflex; check direction every time.

This exact combination — SSH, port 22, source `0.0.0.0/0` — is your Lab 6 build, and it's consistently one of the most common individual findings across real-world cloud security audits, because it's the default a lot of tutorials and quick-start guides teach without explaining the risk.

---

## The part that's genuinely different from traditional network firewalls: security groups are stateful

This is the mental model shift for this module. A traditional network firewall (and NACLs, covered below) requires you to explicitly write rules for both directions of traffic — inbound and outbound are separate, independent rule sets, and a response to a request you let in doesn't automatically get let back out.

**Security groups don't work that way.** They're stateful: if you allow inbound traffic on a port, the *response* to that traffic is automatically allowed back out, regardless of what your outbound rules say. You never have to write a matching outbound rule for return traffic.

Why this matters practically: when you're reading a security group's rules, don't assume a lack of a specific outbound rule means the connection can't complete — a permissive inbound rule often is the whole story by itself. This trips people up who are used to reasoning about stateless traditional firewalls or NACLs.

---

## NACLs: the layer that actually is stateless (and where people misconfigure by assuming it isn't)

Network ACLs sit at the *subnet* level, not the instance level, and unlike security groups, **NACLs are stateless** — inbound and outbound are genuinely independent, exactly like a traditional firewall. If you allow inbound port 80 on a NACL, you must separately allow outbound on the ephemeral port range (typically 1024-65535) or return traffic gets silently dropped.

```
aws ec2 describe-network-acls
```

**Why this is worth checking as an attacker, even though NACLs are usually more permissive by default:** because NACLs are stateless and less commonly touched than security groups, they're where you sometimes find *leftover* rules — someone opened a wide range for a one-time task and never closed it, and because NACL misconfigurations don't cause the obvious "my app doesn't work" symptom that a broken security group does (traffic often still flows via whatever the security group already permits), a bad NACL rule can sit unnoticed far longer.

---

## Putting the two together: which one actually governs access

A request has to pass **both** the NACL (subnet-level) and the security group (instance-level) to reach an instance. Whichever is stricter wins for that request. This means:

- A wide-open security group with a genuinely restrictive NACL can still end up secure
- A locked-down security group with an accidentally-permissive NACL is usually *fine* in practice, specifically because the security group is what's evaluated per-instance and is the one people actually configure carefully — but it's still worth reading both, because you're mapping the account, not assuming based on one data point what the other looks like

When you're enumerating, read both and reason about the combination — don't stop at the first permissive-looking rule and call it a finding without checking whether the other layer would have blocked it anyway.

---

## Confirming a finding is actually exploitable, not just present in the rules

A security group rule existing doesn't guarantee the port is actually listening with a vulnerable service behind it. Once you've identified an instance with SSH open to `0.0.0.0/0`, confirm the instance is actually reachable and the service is live:

```
nc -zv <instance-public-ip> 22
```

If that connects, you've moved from "a permissive rule exists in the JSON" to "there is a live, internet-facing SSH service" — the difference between a config finding and a confirmed exposed attack surface, which matters when you're writing this up the way you did for the Stay-or-Go panel report.

---

## What's coming in Module 05

Module 05 covers **logging and monitoring blind spots** properly — going deeper than the CloudTrail basics touched on in Module 03, including the difference between management events and data events, why CloudTrail being "enabled" doesn't mean what most people assume it means, and the specific multi-region logging gap your own Lab 8 was built around.
