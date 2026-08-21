# Cloud Module 07 — Defense, Remediation & Reporting It Properly

## Why "use least privilege" is not a finding

Every single technical thing in Modules 01-06 traces back to one root cause: a permission or a config that was broader than it needed to be. So the temptation, when writing this up, is to close every finding with "apply least privilege" and move on. Don't. A client (or your panel) reading that gets nothing actionable — it's true, but it's true of almost every cloud finding that has ever existed, which makes it meaningless as specific guidance. This module is about the actual mechanisms that turn "least privilege" from a slogan into a concrete fix, mapped to each thing you found.

---

## IAM Access Analyzer — the tool that finds Module 01-04's findings automatically

Access Analyzer is a native AWS service that continuously scans resource-based policies (S3 bucket policies, IAM role trust policies, KMS key policies, and others) and flags any that grant access to an external entity — anyone outside the account, or optionally outside your organization.

```
aws accessanalyzer create-analyzer --analyzer-name main-analyzer --type ACCOUNT
aws accessanalyzer list-findings --analyzer-arn <analyzer-arn>
```

This is directly the automated version of what you did by hand in Module 01 (reading trust policies) and Module 03 (reading bucket policies) — the exact `"Principal": "*"` and wildcard-trust patterns you were manually scanning JSON for. **This is the correct remediation to recommend for those findings**: not "review policies," but "enable Access Analyzer, which continuously catches this class of misconfiguration going forward, rather than relying on periodic manual audits."

For each finding it returns, the fix is either to explicitly archive it (if the external access is genuinely intentional — a cross-account role meant to be shared, for example) or to tighten the policy's `Principal`/`Condition` block, which Access Analyzer's finding detail will point you toward directly.

---

## Secrets Manager / Parameter Store — the actual fix for Module 06

Every leak pattern in Module 06 (hardcoded creds, plaintext `.env`, plaintext Lambda env vars) has the same root cause: secrets stored as static, human-readable values sitting in something (code, a file, a config) that outlives the secret's intended lifecycle. Secrets Manager fixes this at the mechanism level, not just the "don't do that" level:

```
aws secretsmanager create-secret --name db-password --secret-string "actual-password-here"
```

The application then retrieves it at runtime instead of reading it from an env var or file:

```
aws secretsmanager get-secret-value --secret-id db-password
```

The concrete advantages worth stating explicitly in a report, because they're what make this a real fix rather than moving the problem: **automatic rotation** (Secrets Manager can rotate database credentials on a schedule without code changes, so a leaked credential has a shelf life instead of being valid forever), and **IAM-governed access** (reading a secret is itself an IAM-permissioned API call, so you get the same access logging and control you'd apply to anything else in Modules 01-02, rather than "anyone who can read this file has the secret" with no audit trail).

Parameter Store (part of Systems Manager) is the lighter-weight sibling — same idea, no automatic rotation, free for standard-tier parameters — appropriate to recommend when the client's actual need is config values rather than credentials requiring rotation. Knowing which one to recommend, not just naming both, is what makes a remediation section read as informed rather than copy-pasted.

---

## Mapping every module to its specific fix — the structure a real report needs

This is the actual reporting skill: each finding needs a remediation that maps to the *specific mechanism* that caused it, not a generic control. Structure it like this:

| Module | Root cause | Specific remediation | Not this |
|---|---|---|---|
| 01 — IAM enumeration | Read-heavy policies (`IAMReadOnlyAccess`) over-attached | Scope read access to only what a role's function requires; enable Access Analyzer | "Review IAM permissions" |
| 02 — Privilege escalation | `iam:PassRole` + broad EC2/Lambda permissions on same identity | Split `PassRole` into a separate, tightly-scoped policy limited to specific role ARNs via `Resource` and `iam:PassedToService` condition | "Apply least privilege" |
| 03 — S3 exposure | Bucket policy independently granting public access, unrelated to IAM | Enable S3 Block Public Access at account level; require bucket policies to pass through Access Analyzer before deployment | "Secure your S3 buckets" |
| 04 — Security groups | Inbound `0.0.0.0/0` on sensitive ports | Restrict to specific CIDR ranges or use AWS Systems Manager Session Manager instead of SSH entirely (removes the need for the port to be open at all) | "Restrict security groups" |
| 05 — Logging gaps | Single-region trail, no data events enabled | Enable multi-region trail + S3/Lambda data events explicitly; deploy GuardDuty in all active regions | "Enable more logging" |
| 06 — Secrets exposure | Static plaintext secrets in code/config/state | Migrate to Secrets Manager with rotation; add pre-commit hooks (`git-secrets` or TruffleHog in CI) to block commits containing key patterns | "Don't hardcode secrets" |

The right column is what your Stay-or-Go panel report already did well (the mitigation specificity you built into those 8 findings) — this table is that same discipline, applied across this whole learning series, and is the version worth reusing as a template structure in future reports.

---

## The one remediation that resolves the most findings at once

Worth naming as the closing point, because it's the kind of observation that reads as senior-level judgment rather than a checklist item: **most of Modules 01-06 individually are symptoms of the same underlying gap — no continuous, automated policy review.** A one-time audit (exactly what this whole series has been simulating) finds the findings that exist *today*. Access Analyzer, GuardDuty, and CI-integrated secret scanning are the difference between that and catching the *next* misconfiguration the moment it's introduced, which is the actual argument for why a client should invest in these tools rather than just fixing the current list and moving on.

---

## Series recap — what Modules 00-07 actually built

00 — real API interaction, credentials, why cloud attacks are permission-system attacks, not exploit-based
01 — enumerating an account's full IAM structure using only read permissions
02 — turning specific IAM combinations into full privilege escalation
03 — S3's dual permission system and the enumeration technique for finding exposed buckets
04 — network-layer access (stateful security groups vs. stateless NACLs) and confirming findings are live
05 — logging blind spots that determine what's even investigable after the fact
06 — where credentials actually leak from in practice, across code/containers/serverless/IaC
07 — mapping every finding above to a specific, mechanism-level fix instead of generic advice

That's the full attack-to-defense arc for the cloud domain — matching the depth level your network, web, API, and AD series already have in the repo.
