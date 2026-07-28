# Cloud Security Assessment: AWS Deliberate Misconfiguration Lab

**Author:** Om Bobade
**Type:** Self-directed research — Cloud & AI Security training program
**Date:** July 2026

> 📌 *Screenshots referenced below (`media/*.png`) are from the original assessment — add them here after checking each image for real account IDs, ARNs, or other identifying details.*

---

## Executive Summary

This assessment evaluated a controlled AWS environment purpose-built to demonstrate eight distinct, deliberate security misconfigurations spanning Identity and Access Management (IAM), storage (S3), network configuration (EC2 security groups), and audit logging (CloudTrail). Each misconfiguration was independently designed, deployed as Infrastructure-as-Code (Terraform), detected using automated tooling (Prowler, mapped to the CIS AWS Foundations Benchmark), and — where applicable — validated through manual exploitation to confirm real-world impact rather than relying solely on tool output.

Of the eight findings, two are rated Critical, five are rated High, and one is rated Medium. The most severe findings (Findings 3 and 4) each independently permit either full account compromise or unauthenticated public data exposure, requiring no prior foothold in the environment. Two findings (2 and 8) compound risk by removing the safeguards — MFA and audit logging — that would otherwise limit or help investigate an incident stemming from the other findings.

All misconfigurations were deployed, evidenced, and then fully destroyed within the assessment window; no resources remain live in the target account.

---

## Scope & Methodology

**Scope:** A single AWS account, region `us-east-1`, containing eight independently deployed and torn-down resource groups, each representing one deliberate misconfiguration. No production systems or data were involved; all resources were purpose-built lab artifacts.

**Methodology:**
- Infrastructure defined and deployed as code using Terraform, providing a reproducible, auditable configuration artifact for every finding.
- Automated detection performed using Prowler (v5.33.2), an open-source AWS security scanner, with findings mapped to the CIS AWS Foundations Benchmark where a corresponding control exists.
- Where automated tooling had no matching check (see Finding 3), the misconfiguration was identified and confirmed through manual configuration review of the relevant AWS Console resource.
- Where feasible, findings were validated through manual proof-of-impact — using the vulnerable identity's own permissions or a fully unauthenticated request — to confirm exploitability rather than relying on tool output alone.
- Each finding documents Severity, mapped CIS Control, Description, Blast Radius, Evidence, and Remediation.

---

## Finding 1: IAM Policy Allows Privilege Escalation
**Severity: High** | **CIS Control 1.16** — Ensure IAM policies that allow full administrative privileges are not created

A custom IAM policy (`lab1-wildcard-admin-policy`) was attached to an IAM user (`lab1-vulnerable-user`) with `"Action": "*"` / `"Resource": "*"` — granting the identity unrestricted permissions across every AWS service and resource in the account.

**Blast Radius:** The identity could escalate its own permissions further (`iam:AttachUserPolicy`), create new users/access keys for persistence (`iam:CreateUser`, `iam:CreateAccessKey`), disable audit logging to erase evidence (`cloudtrail:StopLogging`), or delete/modify any resource including data stores and backups.

**Evidence:** Terraform policy definition; live policy confirmed in IAM console; Prowler flagged `iam_policy_allows_privilege_escalation` (FAIL, High); manual exploitation confirmed — using only `lab1-vulnerable-user`'s own credentials, successfully attached the AWS-managed `AdministratorAccess` policy to itself via `aws iam attach-user-policy`.

**Remediation:** Remove the wildcard policy immediately and replace with least-privilege scoping; enforce via AWS Organizations SCPs to block wildcard Action/Resource combinations account-wide; enable AWS Config rule `iam-policy-no-statements-with-admin-access` for continuous detection.

---

## Finding 2: IAM User with Full AdministratorAccess Directly Attached, No MFA Enforced
**Severity: High** | **CIS Controls 1.16 + 1.10** — Administrative privileges without controls; MFA for console access

An IAM user (`lab2-admin-no-mfa-user`) had `AdministratorAccess` attached directly rather than via a group, with a static long-lived access key and no enforced MFA.

**Blast Radius:** A single identity holds full administrative control with no additional authentication barrier — a leaked key or password grants immediate, unrestricted account access. Direct policy attachment (bypassing groups) also makes permissions harder to audit and revoke at scale, and long-lived keys with no rotation extend the exposure window if compromised.

**Evidence:** Terraform showing direct `aws_iam_user_policy_attachment`; IAM console confirming direct attachment; Prowler confirming both the direct-admin and no-MFA findings.

**Remediation:** Detach `AdministratorAccess` from the individual user; assign admin access via a dedicated IAM Group instead. Enforce MFA via an IAM policy condition (`aws:MultiFactorAuthPresent`). Implement a key-rotation policy (e.g. 90-day max age) and monitor stale keys via IAM Access Analyzer.

---

## Finding 3: Cross-Account Trust Policy with Wildcard Principal
**Severity: Critical** | Mapped to IAM least-privilege principles + MITRE ATT&CK T1078.004

*No dedicated CIS control exists for this exact pattern — rating assigned independently, since Prowler returned no built-in finding for it (see tooling-gap note below). Practical impact is equivalent to, or worse than, Finding 1.*

An IAM Role (`lab3-cross-account-wildcard-role`) had a trust policy with `"Principal": {"AWS": "*"}` — permitting **any** AWS account, not just the owner's, to call `sts:AssumeRole`. The role has `ReadOnlyAccess` attached.

**Blast Radius:** Any AWS account holder globally can assume the role with zero prior relationship or invitation. Even with only read access, an attacker could fully enumerate the environment — S3 contents, IAM configuration, EC2 metadata, security group rules — as reconnaissance for a follow-up attack. Wildcard trust policies are actively scanned for by opportunistic attackers precisely because they require no prior foothold.

**Evidence:** Terraform showing `Principal = {AWS = "*"}`; AWS Console confirming the live trust relationship. **Tooling gap:** no Prowler check (v5.33.2) specifically targets wildcard principals in role trust policies — the closest check (`iam_role_cross_service_confused_deputy_prevention`) targets a different, narrower pattern and returned no findings. This gap was identified through manual configuration review, worth flagging explicitly as a limitation of automated tooling.

**Remediation:** Replace the wildcard Principal with specific trusted account ID(s): `{"AWS": "arn:aws:iam::<trusted-account-id>:root"}`. Add an `sts:ExternalId` condition as defense-in-depth even when scoped correctly. Enable IAM Access Analyzer, which flags roles/resources shared with external entities.

---

## Finding 4: S3 Bucket — Public Read Access via Bucket Policy
**Severity: Critical** | **CIS Control 2.1.1** — S3 Buckets configured with Block Public Access

An S3 bucket (`lab4-public-bucket-<suffix>`) had Block Public Access explicitly disabled, plus a bucket policy granting `s3:GetObject` to `Principal: "*"` — allowing unauthenticated access to every object from anywhere on the internet.

**Blast Radius:** Anyone with no AWS account and no credentials can download every object simply by knowing or guessing the bucket URL. If real data were stored here, this is a silent data breach — AWS logs it as "working as configured," not an attack. Publicly readable buckets are continuously scanned by automated bots; exposure time is typically minutes to hours before discovery.

**Evidence:** Terraform showing `block_public_acls = false` / `restrict_public_buckets = false` plus the public bucket policy; direct proof-of-impact opening the object URL unauthenticated in a browser; Prowler flagged `s3_bucket_public_access` and `s3_bucket_level_public_access_block` — both FAIL.

**Remediation:** Enable S3 Block Public Access at the bucket level (all four settings) as the default. Enable it as an account-level default to prevent recurrence on future buckets. Remove the public policy statement; use time-limited pre-signed URLs if external sharing is genuinely needed.

---

## Finding 5: S3 Bucket — No Default Encryption, No Versioning
**Severity: Medium** | **CIS Control 2.1.2** — Server-side encryption + versioning best practice

An S3 bucket (`lab5-unencrypted-bucket-<suffix>`) holding simulated backup data had no default server-side encryption and no versioning enabled — both deliberately omitted.

**Blast Radius:** Without default encryption, objects are stored in plaintext at rest — any other access path exposes raw content with no decryption barrier. Without versioning, accidental deletion, overwrite, or tampering permanently destroys prior data with no recovery path — particularly severe for a bucket explicitly meant to hold backups.

**Evidence:** Terraform showing no accompanying encryption/versioning resource block; AWS Console confirming both settings disabled; Prowler flagged `s3_bucket_default_encryption` and `s3_bucket_object_versioning` — both FAIL.

**Remediation:** Enable default server-side encryption (SSE-S3 minimum, SSE-KMS preferred for sensitive data). Enable versioning on all buckets storing backup/critical data, paired with a lifecycle policy for cost management. Enforce both account-wide via AWS Config conformance pack or SCP.

---

## Finding 6: Security Group — SSH Open to the Entire Internet
**Severity: High** | **CIS Control 5.2** — No security group ingress from 0.0.0.0/0 to port 22

A security group (`lab6-open-ssh-sg`) permitted inbound TCP/22 from `0.0.0.0/0` — every IPv4 address on the internet, no source restriction.

**Blast Radius:** Any resource using this group is directly reachable for SSH attempts from anywhere. This is one of the most actively exploited misconfigurations in real-world environments — mass-scanning botnets probe the entire IPv4 space for port 22 exposure, typically within minutes to hours of the rule going live. Combined with weak/default credentials, this alone is enough for brute-force or exploit-based access.

**Evidence:** Terraform showing `cidr_blocks = ["0.0.0.0/0"]` on the port-22 ingress rule; AWS Console confirming the live rule; Prowler confirmed unrestricted SSH ingress.

**Remediation:** Restrict source CIDR to known, specific ranges — never `0.0.0.0/0`. Where possible, eliminate direct SSH exposure using AWS Systems Manager Session Manager (shell access with no inbound port open). Enable an AWS Config rule for continuous detection.

---

## Finding 7: Publicly-Reachable EC2 Instance, No Network Segmentation
**Severity: High** | Maps to CIS Section 5 (Networking) + defense-in-depth/segmentation best practice

An EC2 instance (`lab7-public-instance`) was launched into a public subnet with `associate_public_ip_address = true`, attached to a security group allowing `0.0.0.0/0` inbound on both port 22 (SSH) and port 80 (HTTP), with no network-level isolation from other account resources.

**Blast Radius:** The instance is reachable on two independently exploitable ports (SSH brute-force, HTTP-layer attacks). With a public IP in the default VPC and no segmentation, a successful compromise offers a foothold with potential lateral-movement paths to other resources on the same flat network. This is a compounding architectural risk, not a single-control failure.

**Evidence:** AWS Console confirming the instance's `running` state with public IPv4 assigned; Prowler flagged `ec2_instance_public_ip` — FAIL.

**Remediation:** Launch instances that don't require direct internet exposure into private subnets with outbound-only access via NAT Gateway. Where public-facing compute is genuinely required, front it with a load balancer or bastion rather than exposing the instance directly. Implement network segmentation (subnets/VPCs by sensitivity tier) so compromise of one public resource can't reach more sensitive internal ones.

---

## Finding 8: CloudTrail Misconfigured — Single-Region, No Log File Validation, Missing Global Service Events
**Severity: High** | **CIS Controls 3.1 + 3.2** — CloudTrail enabled in all regions; log file validation enabled

A CloudTrail trail (`lab8-weak-trail`) had three deliberate weaknesses: `is_multi_region_trail = false`, `enable_log_file_validation = false`, `include_global_service_events = false` — logging only `us-east-1`, without cryptographic log integrity, and excluding global service events like IAM activity.

**Blast Radius:** Any API activity in a region other than `us-east-1` goes completely unlogged — a known real-world attacker technique is deliberately operating in an unlogged region to evade detection. Without log file validation, an attacker with access to the log bucket could alter or delete entries undetected. Since IAM is global and global events are disabled, activity like the privilege escalation in Finding 1 would leave **no audit trail at all**. Combined, incident responders would have little to no forensic evidence to reconstruct a real breach.

**Evidence:** AWS Console confirming "Multi-region trail: No" and "Log file validation: Disabled"; Prowler flagged `cloudtrail_multi_region_enabled` and `cloudtrail_log_file_validation_enabled` — both FAIL.

**Remediation:** Reconfigure with `is_multi_region_trail = true` to capture activity across all regions. Enable `enable_log_file_validation = true` for cryptographic tamper detection. Enable `include_global_service_events = true` to capture IAM and other global-service activity. Route logs to a centralized, access-restricted logging account so even a fully compromised primary account can't tamper with its own audit trail.

---

## Overall Risk Summary

| # | Finding | Severity |
|---|---------|----------|
| 1 | IAM Policy Allows Privilege Escalation | High |
| 2 | IAM User — Direct AdministratorAccess, No MFA | High |
| 3 | Cross-Account Trust Policy, Wildcard Principal | Critical |
| 4 | S3 Bucket — Public Read Access | Critical |
| 5 | S3 Bucket — No Encryption, No Versioning | Medium |
| 6 | Security Group — SSH Open to Internet | High |
| 7 | Public EC2 Instance, No Segmentation | High |
| 8 | CloudTrail — Single-Region, No Validation | High |

### Priority Recommendations
1. **Remediate Findings 3 and 4 immediately** — each independently enables full account compromise or unauthenticated data exposure with no prior foothold required.
2. **Address Findings 2 and 8 as a paired priority** — together they remove both the authentication barrier (MFA) and the audit trail that would otherwise limit or help investigate exploitation of the other findings.
3. **Adopt account-wide preventive controls** (Service Control Policies, S3 Block Public Access at the account level, AWS Config conformance packs) rather than remediating findings individually, to prevent recurrence of these patterns on future resources.
4. **Where automated tooling coverage gaps exist** (see Finding 3), supplement scanning with periodic manual configuration review of IAM trust relationships.
