# Cloud Module 05 — Logging & Monitoring Blind Spots

## "CloudTrail is enabled" is a much weaker statement than it sounds

Every AWS account has CloudTrail on by default, in a form called the **90-day event history** — visible in the console, covering the last 90 days of *management events only*, in *that region only*. Most people see "CloudTrail: enabled," assume they have full logging, and move on. As someone assessing an account, this default is your starting assumption about what's actually being missed, not what's being covered.

```
aws cloudtrail describe-trails
```

An empty result, or a result with no trails at all, means the account is relying purely on that 90-day default — no persistent log storage, no multi-region coverage, no data events. That single command tells you more about an account's actual detection capability than almost anything else you'll check.

---

## Management events vs. data events — the distinction that matters most

This is the core concept for this module, and it directly explains the S3 detection gap from Module 03.

**Management events** are control-plane operations — creating a user, launching an instance, changing a policy, deleting a bucket. Things that change the *configuration* of the account. These are logged by CloudTrail by default whenever a trail exists.

**Data events** are operations on the actual *data* inside a resource — reading an S3 object (`GetObject`), invoking a Lambda function. These are **not logged by default**, even with a trail configured, because the volume is enormous (every single object read, at scale) and AWS charges separately for data event logging.

```
aws cloudtrail get-event-selectors --trail-name <trail-name>
```

This tells you whether data events are actually being captured for a trail, and for which resources specifically. If it comes back showing only management events, you've just confirmed: every `GetObject` call from Module 03's S3 enumeration — even against a private bucket, by someone with legitimate but compromised credentials — generated **no log entry the account owner would ever see**. Not "hard to find." Not logged at all.

This is worth sitting with, because it reframes something from Module 03: the danger of public/exposed S3 access isn't just that the data is exposed — it's that the *investigation* after the fact may have literally nothing to work with.

---

## The multi-region gap — your Lab 8, explained properly

CloudTrail trails are, by default, **single-region**. A trail created in `us-east-1` only logs API activity that happens to be processed in `us-east-1`. AWS has many regions; an attacker with valid credentials can simply operate in a region the trail doesn't cover.

```
aws cloudtrail describe-trails --query 'trailList[*].{Name:Name,IsMultiRegion:IsMultiRegionTrail,HomeRegion:HomeRegionRegion}'
```

`IsMultiRegionTrail: false` is the finding — it means the account has *some* logging, which can create false confidence, while entire regions are completely dark. This is exactly the gap your Lab 8 was built around, and it's a genuinely common real-world misconfiguration: someone sets up CloudTrail once, in whatever region they happened to be working in, and never revisits it as the account's footprint grows into other regions.

Practical attacker behavior this enables: enumerate and act in a region far from the account's normal operating region (check `ec2 describe-instances` region-by-region to see where the account is actually active), specifically because a security team's alerting and review habits — even without a technical logging gap — tend to focus on the regions they know they use.

---

## Recovering (or confirming the absence of) evidence, tying back to Module 03

Given a bucket and a time window, this is how you'd check what's actually recoverable:

```
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=target-bucket-name \
  --start-time 2026-08-01T00:00:00Z \
  --end-time 2026-08-20T00:00:00Z
```

If S3 data events were never enabled (Module 03's finding), this returns nothing for `GetObject` calls, no matter how much unauthorized reading happened in that window — while still potentially showing `PutBucketPolicy` or other management events if the bucket's configuration was changed. This split result — some activity visible, the actual data access invisible — is what a real incident investigation against an under-logged account looks like, and it's worth being able to demonstrate exactly this gap in a findings report the way you did for the Stay-or-Go panel (documenting "logging insufficient to determine data access" is a legitimate, common finding on its own).

---

## GuardDuty: detection built on top of these same logs

Worth naming even briefly: GuardDuty is AWS's managed threat-detection service, and it works by analyzing CloudTrail, VPC Flow Logs, and DNS logs for known-bad patterns (credential compromise indicators, communication with known malicious IPs, anomalous API call sequences). The direct consequence of everything above: **GuardDuty inherits the same blind spots as the logs it's built on**. If data events aren't logged, GuardDuty can't detect anomalous data access, no matter how sophisticated its detection models are. It's not a separate, independent safety net — it's downstream of the exact gaps this module covers.

```
aws guardduty list-detectors
```

An empty result means GuardDuty isn't even running in that region — worth checking region-by-region for the same reason the CloudTrail multi-region check matters.

---

## What's coming in Module 06

Module 06 shifts from "finding misconfigurations" to **secrets and credential exposure specifically** — where AWS credentials actually leak from in practice (source code, environment variables, container images, Lambda environment configs, even CloudFormation/Terraform state files revisited from a credentials angle), and the specific detection/scanning approach for finding them before an attacker does.
