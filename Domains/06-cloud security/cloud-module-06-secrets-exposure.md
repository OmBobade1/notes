# Cloud Module 06 — Secrets & Credential Exposure

## Why this module comes after, not before, everything so far

Modules 01-05 assumed you already had a valid credential and walked through what to do with it. This module is about the step before that — **how attackers actually get that first credential in the real world.** The honest answer is almost never "sophisticated exploitation." It's almost always one of the handful of leak patterns below, found through boring, repeatable scanning.

---

## Source code and version control — the single biggest source

Developers hardcode credentials directly into code constantly, usually temporarily "just to test something," and then commit it. Once it's committed, deleting it in a later commit **does not remove it from git history** — this is the part beginners consistently miss.

```
git log -p | grep -i "AKIA"
```

`AKIA` is the fixed prefix every AWS access key ID starts with — searching for it across the full commit history (`-p` shows the actual diff content of every commit, not just messages) catches keys that were added and later "removed," because the old commit is still sitting in the repo's history, fully retrievable.

For scanning without manually grepping every repo, tools exist specifically for this:

```
trufflehog git https://github.com/target-org/target-repo
```

TruffleHog walks the entire commit history of a repo looking for high-entropy strings and known secret patterns (AWS keys, private keys, API tokens for dozens of services) — this is genuinely how a large share of real-world AWS compromises begin: automated bots running exactly this kind of scan continuously against every public GitHub push, not against specific targets. This is why the `.aws/credentials` leak scenario from Module 00 isn't a contrived example — it's the actual dominant pattern.

---

## Environment variables and `.env` files

The "proper" alternative to hardcoding is putting credentials in environment variables, loaded from a `.env` file. This is genuinely better practice than hardcoding — but `.env` files get committed to git by mistake constantly, specifically because `.gitignore` has to be configured correctly *before* the first commit, and it frequently isn't.

```
git log --all --full-history -- "**/.env"
```

This searches the entire history of every branch for any file ever named `.env` at any path, even if it was later deleted and gitignored — same underlying lesson as the source code case: deletion doesn't erase history.

---

## Container images — a source people forget entirely

Docker images can contain credentials baked in at build time — either hardcoded in a `Dockerfile` `ENV` instruction, or copied in via a `COPY .env .` line that seemed harmless. Once an image is pushed to a registry (Docker Hub, ECR, even a private one with weak access controls), every layer is inspectable, including layers from earlier build steps that later steps appear to "remove."

```
docker history --no-trunc target-image:latest
docker save target-image:latest -o image.tar && tar -xf image.tar
```

`docker history --no-trunc` shows the full, untruncated command run at each build layer — if a `RUN export AWS_SECRET_ACCESS_KEY=...` or similar ever happened, it's visible here even if a later layer unsets it. Extracting the image with `docker save` and untarring it lets you grep every filesystem layer directly, catching credentials in files that were deleted in a later layer but still physically exist in an earlier one — Docker layers are additive, not actually overwriting.

---

## Lambda environment variables — visible to more people than you'd expect

Lambda functions commonly store credentials as environment variables via the console or `serverless.yml`/Terraform. If you've gotten this far into an account (Module 01's enumeration would have surfaced Lambda functions via `aws lambda list-functions`), reading a function's configuration is a single low-privilege call:

```
aws lambda get-function-configuration --function-name target-function
```

The `Environment.Variables` field in the response is returned as **plaintext** unless the account has specifically configured KMS encryption for Lambda environment variables (`KMSKeyArn` in that same response — its absence is the tell). A huge number of real Lambda functions store third-party API keys, database passwords, and yes, AWS credentials for cross-account access, in plaintext env vars, readable by anyone with `lambda:GetFunctionConfiguration` — a permission that's far more commonly granted than people realize, because it looks read-only and harmless.

---

## Terraform state files — revisited from the credentials angle

Module 03 mentioned `.tfstate` files as a high-value target in S3 enumeration. Here's specifically why: Terraform state files record the **full resulting configuration** of every resource Terraform manages — and when a resource is created with a secret as an input (an RDS database password set via a Terraform variable, an IAM access key generated by the `aws_iam_access_key` resource), that secret is written into the state file **in plaintext**, by design, because Terraform needs to track the actual deployed state.

```
grep -i "secret\|password\|access_key" terraform.tfstate
```

This is not a misconfiguration of Terraform — it's documented, expected behavior that most teams don't account for when deciding where state files get stored and who can read them. Combined with the Module 03 finding of a public or loosely-permissioned S3 bucket being used as a Terraform backend, this is a direct, common path from "found a bucket" to "have valid, high-privilege credentials."

---

## Bringing it together: the actual attacker workflow, in order

1. Automated scanning (TruffleHog-style) against public GitHub, continuously, for pattern `AKIA` and similar — no specific target required
2. On a specific target: check public S3 buckets (Module 03) for `.tfstate`, `.env`, and config files
3. With any initial access: enumerate Lambda functions and pull their env var configs
4. With any initial access: pull container images from any accessible registry and inspect layer history

Every step here is passive-to-low-noise, matches the CloudTrail visibility gaps from Module 05 (reading a Lambda config or an S3 object may generate little to no useful logging), and requires zero custom exploit code — it's pattern recognition and knowing where people consistently make the same mistake.

---

## What's coming in Module 07

Module 07 closes out the technical attack-path content by covering **defense and remediation properly** — not a bullet list of "use least privilege," but the actual mechanisms: IAM Access Analyzer for finding the exact wildcard/public-access findings from this whole series automatically, Secrets Manager/Parameter Store as the correct alternative to everything in this module, and how to structure a client-grade remediation section so it reads as prioritized and actionable rather than a generic checklist — directly building on how you structured the Stay-or-Go panel report.
