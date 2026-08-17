# Module 08 - Improper Inventory Management

## Why this is a genuinely different kind of risk than everything before it
Every module so far assumed you're testing a *known* API endpoint. This module is about the risk that comes from **not knowing what API surface actually exists at all** — old versions, forgotten environments, undocumented endpoints. You can't secure, patch, or even test something the organization itself has lost track of.

---

## What it actually is
Modern organizations often run many versions and copies of the same API simultaneously — production, staging, development, and old versions kept around for backward compatibility with older mobile app releases still in use. Improper Inventory Management is the risk that accumulates when this sprawl isn't tracked, documented, or decommissioned properly — meaning some of these copies quietly become the weakest, least-monitored, least-patched entry point into the exact same underlying systems the well-maintained production API protects carefully.

---

## Shadow APIs — the ones nobody officially knows about

**What it is:** an API endpoint or entire API that exists and is reachable, but isn't part of the organization's official inventory or documentation — often created for a temporary purpose, a quick internal tool, or a feature that was later abandoned but never actually removed.

**Why this happens in practice, not just in theory:** development moves fast, and a developer spinning up a quick internal API for a specific task rarely goes back later to formally register or document it once its original purpose is done — it just keeps running, forgotten, still fully reachable.

**How this gets discovered — both by legitimate security teams and by attackers, using the same techniques:**
```
# Certificate Transparency logs frequently reveal subdomains never intended to be publicly known
crt.sh?q=%.target.com

# Subdomain enumeration (connects to Module 03's reconnaissance content)
subfinder -d target.com

# Historical API endpoint discovery via web archives
waybackurls target.com | grep -i "api\|v1\|v2"
```
The same reconnaissance techniques already covered for general subdomain/asset discovery apply directly here — a shadow API is, at its core, just an undocumented asset, discoverable the exact same way any other forgotten asset gets found.

---

## Deprecated API Versions Left Running

**What it is:** organizations frequently need to support older mobile app versions that haven't updated yet, so `api.target.com/v1/` might still be fully live and functional years after `v2` or `v3` became the current, actively-maintained version.

**Why old versions specifically become the weakest point, not just an equally-risky duplicate:** security fixes, rate limiting improvements, and updated authentication requirements are almost always applied to the *current* version first, and sometimes only to the current version — an old `v1` endpoint may still be running with security practices from years ago, even while the exact same underlying data and business logic is being protected far more carefully on `v2`/`v3`.

**A concrete, realistic scenario:** BOLA (Module 01) gets discovered and properly fixed on `v2` of an orders API — but nobody thinks to check whether `v1`, still fully live for backward compatibility, has the exact same underlying vulnerability, since it was never part of the fix's testing or deployment scope at all.

```
GET /api/v1/orders/5002    ← the old, unpatched, forgotten version
GET /api/v2/orders/5002    ← the current, properly-secured version
```

**Testing implication worth stating directly:** a complete API assessment has to explicitly enumerate and test *every discoverable version*, not just whatever version is currently linked from official documentation — an attacker doesn't respect "please only use the current version," and neither should a thorough test.

---

## Undocumented/Excessive Data Exposure Through Old or Debug Endpoints

**A related pattern:** endpoints created for internal debugging or early development ("`/api/debug/dump-all-users`") that were meant to be temporary but never got removed before or after the API went to production — often with far weaker or entirely absent authentication compared to the properly-built production endpoints, precisely because they were never intended to be a real, permanent part of the API surface at all.

---

## Missing or Outdated API Documentation as a Security Problem, Not Just an Inconvenience

**Why this connects back to the earlier authorization modules:** an organization can't properly review its own access controls (Module 01, Module 04) across an API surface it doesn't have an accurate, complete map of. Improper inventory isn't just "attackers might find something extra" — it directly undermines the organization's own ability to apply its own security fixes comprehensively, since a fix applied to "the API" often actually only means "the specific parts of the API someone remembered still exist."

## Business Impact

| Angle | What it actually means |
|---|---|
| **Financial loss** | A forgotten `v1` API still exposing the same BOLA vulnerability the current `v2` API already fixed means the actual, real-world risk to customer financial data never actually went away — it just moved to a location nobody was checking |
| **Regulatory / compliance** | "We fixed that vulnerability" being only partially true — fixed on the version anyone remembered to test — is a serious finding in any audit that specifically checks for comprehensive remediation, not just remediation on the primary/documented surface |
| **Reputational damage** | Shadow APIs and forgotten debug endpoints are exactly the kind of finding that shows up in bug bounty reports and security research publications, since they're specifically the parts of an API surface official security reviews are most likely to have missed |
| **Legal liability** | An organization is generally still fully responsible for a forgotten, undocumented endpoint exposing customer data, regardless of whether anyone internally remembered it existed — "we didn't know about it" is not a meaningful legal defense for data it was still actively serving |
| **Operational cost** | Building a genuinely accurate inventory retroactively, after years of undocumented sprawl, is a significant, ongoing effort — far more expensive than maintaining accurate documentation and a deprecation process from the start |

**One-line interview answer:** *"Improper inventory management means an organization doesn't have a complete, accurate picture of its own API surface — old versions kept alive for backward compatibility, forgotten debug endpoints, or shadow APIs nobody documented. The real danger is that a fix applied to 'the API' often only reaches the parts someone remembered still exist — a vulnerability properly patched on the current version can remain fully live and exploitable on a forgotten older version indefinitely."*

## Mitigation

1. **Maintain a genuinely complete, actively-updated API inventory** — every version, every environment, tied to an owner responsible for its lifecycle, not a document created once and never revisited.
2. **Implement a formal deprecation and sunset process for old API versions** — a defined timeline for when an old version actually gets shut down, not "kept alive indefinitely just in case."
3. **Apply security fixes and reviews across every live version, not just the current one** — if `v1` is still reachable, it's still in scope for every security review, full stop.
4. **Regularly run external discovery scans against your own organization** (certificate transparency, subdomain enumeration, web archive checks) — the same techniques covered above — specifically to catch what internal records may have missed.
5. **Strictly separate debug/development endpoints from production deployments** at the infrastructure level, not just by convention — a debug endpoint should be architecturally incapable of being reachable in production, not just "not supposed to be."

## Quick-reference table

| Issue | Root Cause | Fix |
|---|---|---|
| Shadow APIs | Quick/temporary endpoints never formally tracked | Complete, actively-maintained inventory |
| Deprecated versions left running | Backward compatibility need, no sunset process | Formal deprecation timeline, security fixes applied to all live versions |
| Forgotten debug endpoints | Never removed after development | Architectural separation of debug/production, not just convention |
| Inaccurate documentation | Security reviews only cover what's remembered | External self-discovery scans (cert transparency, subdomain enum) |
