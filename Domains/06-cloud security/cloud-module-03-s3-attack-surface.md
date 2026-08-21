# Cloud Module 03 — S3 as an Attack Surface

## Why S3 gets its own module

S3 causes more real-world breaches than almost anything else in AWS — not because it's technically complex, but because it has **two separate, overlapping permission systems** that people configure inconsistently, and because buckets are globally-named and therefore guessable. This module covers both problems and the enumeration technique that exploits the second one.

---

## The permission confusion: bucket policy vs. IAM policy vs. ACL

This is the part that trips almost everyone up, so get it straight now. A single S3 bucket's access can be controlled by **three different mechanisms simultaneously**, and AWS evaluates all of them together — the most permissive combination that doesn't have an explicit `Deny` wins:

1. **IAM policy** — attached to a *user or role* (what you enumerated in Module 01). Says "this identity can do X to S3."
2. **Bucket policy** — attached to the *bucket itself*, not any identity. Says "this bucket can be accessed by X." This is where `"Principal": "*"` shows up — meaning literally anyone on the internet, no AWS account needed at all.
3. **ACLs (Access Control Lists)** — the oldest, least-used-correctly mechanism. Legacy, but still functional unless explicitly disabled at the account level (`S3 Block Public Access` settings, which is the modern account-wide override).

**Why this matters for you as an attacker:** a bucket can have a perfectly locked-down IAM setup — no user or role has S3 permissions — and still be world-readable, because the *bucket policy* independently grants public access. People check IAM and assume they're safe. They forget the bucket policy is a completely separate door.

---

## Reading a bucket policy for the danger signal

```
aws s3api get-bucket-policy --bucket target-bucket-name
```

You're scanning the returned JSON for exactly this shape:

```json
{
  "Effect": "Allow",
  "Principal": "*",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::target-bucket-name/*"
}
```

`"Principal": "*"` is the whole finding. It means no authentication required — the bucket is readable by anyone with the URL, full stop. This is precisely S3 Finding pattern from your Stay-or-Go Lab 4 (public read bucket), except now you're finding it on a real target through direct policy inspection instead of a lab you built and already know the answer to.

---

## The enumeration problem: finding buckets nobody told you about

Bucket *names* are global across all of AWS — no two AWS accounts anywhere can have a bucket with the same name. This global uniqueness is exactly what makes buckets guessable, because companies almost always name buckets predictably: `companyname-backups`, `companyname-prod-assets`, `companyname-logs-2025`.

**Manual check, one bucket at a time:**

```
aws s3 ls s3://guessed-bucket-name --no-sign-request
```

The `--no-sign-request` flag is doing real work here — it tells the CLI to make the request completely unauthenticated, no credentials attached at all. If this succeeds, the bucket is public to literally anonymous internet traffic, not just to someone with valid AWS credentials. That's a meaningfully worse finding than "public to any authenticated AWS user," and this flag is how you tell the difference.

**At scale, this becomes automated wordlist-based brute forcing** — tools like `S3Scanner` or a basic loop feeding a wordlist of company-name permutations (`companyname-dev`, `companyname-staging`, `companyname-uploads`, etc.) into the same `aws s3 ls --no-sign-request` check, hundreds or thousands of times, looking for the ones that don't 403. This is genuinely how a large share of real S3 breaches start — not sophisticated exploitation, just patient guessing against a naming convention.

---

## Once you're in: what you're actually looking for

Listing a bucket's contents:

```
aws s3 ls s3://target-bucket-name --no-sign-request --recursive
```

The `--recursive` flag matters — without it you only see the top-level "folders" (S3 doesn't actually have folders, just key prefixes that look like paths), and real damage usually sits nested a few levels deep. High-value targets in a listing: anything named `backup`, `.env`, `credentials`, `.git`, database dump extensions (`.sql`, `.bak`), or Terraform state files (`.tfstate` — these frequently contain plaintext secrets and full infrastructure details, and are catastrophically common to find sitting in an S3 bucket used as a Terraform backend with the wrong permissions).

Pulling a specific file down once you've spotted something:

```
aws s3 cp s3://target-bucket-name/path/to/file.sql . --no-sign-request
```

---

## CloudTrail: how Modules 01-03 actually show up in logs (or don't)

This is the detection side, tying the whole series together so far.

**IAM enumeration (Module 01)** — `list-users`, `list-roles`, `get-policy-version`, etc. all generate CloudTrail events, but they're `ReadOnly` event-type API calls, and most organizations don't alert on read-only IAM calls at all because legitimate tooling (Prowler, ScoutSuite, even the AWS Console itself) generates thousands of them constantly. This is exactly why enumeration is considered low-risk for an attacker — it blends into background noise.

**Privilege escalation (Module 02)** — `RunInstances`, `CreatePolicyVersion`, `AttachUserPolicy` are all logged as `WriteOnly` events and are genuinely detectable — a well-configured CloudTrail + GuardDuty setup should alert on `AttachUserPolicy` granting `AdministratorAccess` almost immediately. This is a real defensive control gap you can point to: if it didn't alert, the account either isn't running GuardDuty or isn't reviewing the alerts.

**S3 public access (this module)** — here's the important gap: reading objects from a bucket via `--no-sign-request` **may generate no CloudTrail event visible to the account owner at all**, if S3 server access logging or CloudTrail data-event logging for S3 isn't explicitly turned on — and by default in most accounts, it isn't. Unauthenticated, anonymous reads against a public bucket are, in many real configurations, essentially invisible. This is the single most important detection fact in this module: public S3 exposure isn't just a bigger blast radius than a locked-down IAM mistake — it's frequently also the quietest one.

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=GetObject
```

Run this against your own account after testing — if S3 data events aren't enabled, you'll see this come back empty even for objects you just pulled, which is the proof of the gap rather than just a claim about it.

---

## What's coming in Module 04

Module 04 moves to **network-layer cloud attack surface** — security groups and NACLs, the open-SSH-to-the-world pattern from your Lab 6, and how "it's a firewall" thinking from traditional network pentesting (Domain 01) maps onto — and differs from — AWS's security group model, which is stateful in a way that changes how you'd actually approach probing it.
