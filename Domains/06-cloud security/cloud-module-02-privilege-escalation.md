# Cloud Module 02 — IAM Privilege Escalation (Turning a Map Into Admin)

## Where you left off

Module 01 got you a full map: which roles exist, what's attached to them, and — critically — you were told to flag any role where the identity you're sitting in has `iam:PassRole` combined with EC2 or Lambda permissions. This module is that flag, executed.

Privilege escalation in AWS almost never means "finding a bug." It means finding a **combination of individually reasonable-looking permissions** that, together, let a low-privilege identity end up as an administrator. There are roughly 20 documented paths like this (this is public, well-studied research — start with Rhino Security Labs' AWS privesc methodology if you want the full list later). This module walks through the single most common one end-to-end, then names the rest so you know what to check for.

---

## The core mechanic: what `iam:PassRole` actually does

This is the permission almost every escalation path abuses, so understand it properly before touching a command.

Normally, when you launch an EC2 instance or a Lambda function, you can attach an IAM role to it — that's how the instance/function gets permissions to talk to other AWS services. `iam:PassRole` is the permission that lets *you* be the one who hands a role to a new resource.

The mistake that makes this exploitable: `iam:PassRole` doesn't check what the role you're passing can actually do. It just checks that you're allowed to pass *a* role. So if you have `iam:PassRole` plus permission to launch an EC2 instance, and there's an existing role in the account with `AdministratorAccess` sitting around (there almost always is — Lab 2 in your Stay-or-Go work was exactly this pattern), you can attach that admin role to a new instance you control, and then just... use it.

---

## The actual attack, step by step

**Step 1 — confirm you have the two permissions you need**

From Module 01's enumeration, you already know your attached policies. Confirm specifically:

```
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/your-current-user \
  --action-names iam:PassRole ec2:RunInstances
```

`simulate-principal-policy` is worth knowing on its own — it lets you test *hypothetically* whether an action would be allowed, without actually running it. Zero risk, zero logs of the actual action, just a policy-logic answer.

**Step 2 — identify a role worth stealing**

Back to your Module 01 role map. You're looking for a role whose trust policy allows `ec2.amazonaws.com` to assume it (this is what makes it usable by an EC2 instance in the first place):

```json
{
  "Effect": "Allow",
  "Principal": {"Service": "ec2.amazonaws.com"},
  "Action": "sts:AssumeRole"
}
```

Combined with a policy like `AdministratorAccess` attached to that role. This is the exact combination from your Lab 2 build — a role that trusts EC2 and carries admin.

**Step 3 — launch an instance, passing yourself that role**

```
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t2.micro \
  --iam-instance-profile Name=admin-instance-profile \
  --key-name your-keypair \
  --subnet-id subnet-0123456789abcdef0
```

The critical flag is `--iam-instance-profile` — this is where you pass the over-privileged role to the new instance. (Note: roles get attached to EC2 via an "instance profile," a thin wrapper around the role — if `list-instance-profiles` wasn't part of your Module 01 sweep, it should be next time; it's a separate object from the role itself.)

**Step 4 — get in and pull the credentials**

Once the instance is running, SSH in, and every EC2 instance with a role attached exposes that role's temporary credentials at a fixed, unauthenticated-from-localhost metadata endpoint:

```
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/admin-instance-profile
```

This returns a JSON blob with an `AccessKeyId`, `SecretAccessKey`, and `SessionToken` — a live, temporary admin credential set. Export those three into your CLI environment and you're now operating as an administrator, using nothing but a `t2.micro` instance you spun up with a low-privilege key.

```
aws sts get-caller-identity
```

Run it again. Compare the `Arn` to what you got in Module 00. That's the whole story of this attack in one before/after.

---

## The other common paths (know these exist, even without full walkthroughs yet)

- **`iam:CreatePolicyVersion`** — if you can create a new version of a policy already attached to your own user, you can set the new version's content to `AdministratorAccess`-equivalent JSON and mark it as default. You escalate yourself with no other resource involved.
- **`iam:AttachUserPolicy`** — if you can attach *any* policy to *any* user, attach `AdministratorAccess` to yourself directly. The most blunt version of this whole category.
- **`lambda:CreateFunction` + `iam:PassRole`** — same mechanic as the EC2 path above, but you pass the over-privileged role to a Lambda function instead, then invoke it to run arbitrary code (`boto3` calls) as that role — often stealthier than spinning up a visible EC2 instance.
- **`iam:UpdateAssumeRolePolicy`** — if you can modify a role's *trust* policy (not its permissions, its trust), you can rewrite the trust policy to allow your own user to assume it, regardless of what the role's actual permissions are.

Every one of these follows the same underlying lesson: individually, each permission looks like something a developer would reasonably need. The danger is never one permission — it's the combination.

---

## Why this matters more in cloud than it did in your on-prem/AD work

Compare this to the AD privilege escalation content in your Domain 05 series — there, escalation usually requires exploiting a *misconfiguration* or *vulnerability* (unconstrained delegation, ACL abuse, kerberoasting). Here, every single step you just did was a **documented, intended AWS API feature**, used exactly as designed. There's no exploit code, no payload, no injection. This is the "read a policy file and know this line is dangerous" judgment mentioned back at the very start of this series — cloud privesc is a permissions-design failure, not a software vulnerability, and that's exactly why tools like Prowler flag these combinations as findings rather than CVEs.

---

## What's coming in Module 03

Module 03 moves to **S3 as an attack surface** — public buckets, misconfigured bucket policies vs. IAM policies (a distinction that trips people up constantly), and the specific enumeration technique for finding buckets that were never meant to be found, before circling back to detection: how each technique in Modules 01-03 shows up (or fails to show up) in CloudTrail logs.
