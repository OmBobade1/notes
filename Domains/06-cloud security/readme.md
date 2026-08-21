# Cloud Security

8 modules (`00`-`07`), covering AWS fundamentals through IAM enumeration, privilege escalation, S3 exposure, network-layer access, logging blind spots, secrets exposure, and remediation — built around the idea that cloud attacks are permission-system attacks, not exploit-based ones.

---

## 🧭 What to Reference, Based on What You're Doing

| If you're... | Reference these modules |
|---|---|
| **New to AWS entirely, or found a leaked access key and don't know what to do first** | `00` |
| **Mapping an account's users, roles, and policies with low-privilege creds** | `01` |
| **Checking if `iam:PassRole` or a similar combo leads to admin** | `02` |
| **Testing an S3 bucket's actual public exposure, or hunting for undisclosed buckets** | `03` |
| **Reading security group / NACL rules, or confirming a port is actually reachable** | `04` |
| **Checking what an account would actually have logged after the fact** | `05` |
| **Looking for leaked AWS credentials in code, containers, Lambda configs, or Terraform state** | `06` |
| **Writing the remediation section of a cloud findings report** | `07` |

## 🧭 A Practical Testing Order
1. `00` — confirm identity and access with any credential found (`aws sts get-caller-identity`) before anything else.
2. `01` — enumerate the full account: users, roles, trust policies, attached and inline policies.
3. `02` — check enumerated roles/policies for known privilege-escalation combinations.
4. `03` — check S3 bucket policies independently of IAM findings; run public-bucket enumeration.
5. `04` — check network-layer exposure (security groups, NACLs) for anything found reachable so far.
6. `05` — before concluding testing, check what of the above would actually be visible in the account's own logs.
7. `06` — separately, scan for leaked credentials across code/containers/serverless configs/IaC state.
8. `07` — map every finding to a specific, mechanism-level remediation rather than generic advice.

---

## 📖 Full Module Index

| # | File | Covers |
|---|---|---|
| 00 | `cloud-module-00-what-is-cloud.md` | AWS CLI vs. console, `aws sts get-caller-identity`, how creds live on disk and leak, reading a 403 for information |
| 01 | `cloud-module-01-iam-enumeration.md` | `list-users`/`list-roles`, trust policies, inline policies, group-inherited permissions |
| 02 | `cloud-module-02-privilege-escalation.md` | `iam:PassRole` + EC2 escalation end-to-end, plus three other common escalation paths |
| 03 | `cloud-module-03-s3-attack-surface.md` | Bucket policy vs. IAM policy, `--no-sign-request` enumeration, high-value file targets |
| 04 | `cloud-module-04-security-groups-nacls.md` | Stateful security groups vs. stateless NACLs, confirming live exposure |
| 05 | `cloud-module-05-logging-monitoring.md` | Management vs. data events, multi-region CloudTrail gaps, GuardDuty's blind spots |
| 06 | `cloud-module-06-secrets-exposure.md` | Git history, `.env` files, Docker layers, Lambda env vars, Terraform state |
| 07 | `cloud-module-07-defense-remediation.md` | Access Analyzer, Secrets Manager, finding-to-remediation mapping table |

## How to use this
Each module builds directly on the enumeration/findings from the one before it — read in order the first time through. Several modules reference the Stay-or-Go panel lab findings (wildcard trust policies, public buckets, open SSH, single-region CloudTrail) as concrete real-world matches for the technique being taught.
