# Cloud Module 01 — IAM Enumeration (Mapping an Account with Low-Privilege Creds)

## The situation you're actually in

You did what Module 00 covered — you found a leaked access key, ran `aws sts get-caller-identity`, and confirmed it's real. But the identity attached to it is boring. Not an admin. Just some low-privilege service account or dev user. Most people stop here and think "useless key, move on."

That's the wrong instinct. Before you touch anything destructive or noisy, your job is to **map the account** — find out what users, roles, and policies exist, without needing permission to actually use most of them. This is IAM enumeration, and it's almost always possible even from a near-zero-privilege identity, because AWS's default IAM permission model is more permissive for *reading* than most people realize.

---

## Why "read" permissions on IAM are dangerous by default

Here's the thing nobody tells beginners: a huge number of real AWS accounts have policies like `IAMReadOnlyAccess` or custom policies with `iam:List*` / `iam:Get*` attached to low-privilege users — because developers need to see their own permissions, or debug something once, and the access never gets removed.

If your leaked identity has even a sliver of this, you can build a full map of the account's IAM structure: every user, every role, every group, every attached policy, and critically — **what each of those roles can actually do**, without ever needing to assume them.

Run this first:

```
aws iam list-users
```

If it doesn't error with `AccessDenied`, you get back every IAM user in the account:

```json
{
    "Users": [
        {
            "UserName": "dev-test-user",
            "UserId": "AIDAABCD1234EFGH5678I",
            "Arn": "arn:aws:iam::123456789012:user/dev-test-user",
            "CreateDate": "2025-11-02T14:22:00Z"
        },
        {
            "UserName": "backup-service-account",
            "Arn": "arn:aws:iam::123456789012:user/backup-service-account",
            "CreateDate": "2024-06-14T09:10:00Z"
        }
    ]
}
```

Immediately worth noticing: `backup-service-account` — a name like this is a signal. Service accounts tend to be over-permissioned because nobody expects them to be "used" the way a human would use console access, so they often get broad policies attached and then forgotten.

---

## Mapping roles — the more valuable target

Users are interesting, but **roles** are usually the real prize, because roles are what applications, EC2 instances, and Lambda functions assume to get their permissions. Compromise the right role, and you inherit whatever that application can do.

```
aws iam list-roles
```

This returns every role, including its **trust policy** — literally the rules for who's allowed to assume it. Look specifically at the `AssumeRolePolicyDocument` field in the output for each role. This is where you'd catch something like the wildcard trust policy pattern from the Stay-or-Go panel work (Finding 3: `Principal: {"AWS": "*"}`) — except now you're finding it through enumeration on a real target instead of a lab you built yourself.

---

## The command that tells you exactly what a role or user can actually do

Listing roles tells you they exist. It doesn't tell you what they're capable of. For that:

```
aws iam list-attached-role-policies --role-name <role-name>
```

This returns the AWS-managed policies attached to that role (e.g. `AmazonS3FullAccess`, `AdministratorAccess`). Then, for custom (non-AWS-managed) policies, you need one more step to see the actual permission JSON:

```
aws iam get-policy --policy-arn <policy-arn>
aws iam get-policy-version --policy-arn <policy-arn> --version-id <version-id>
```

Two calls, not one — AWS versions policies, so `get-policy` only tells you which version is currently active; `get-policy-version` is what actually returns the permission JSON (the `Action`/`Resource`/`Effect` statements) you need to read.

**What you're scanning for while reading that JSON:** any `"Action": "*"` or `"Resource": "*"`, any `iam:PassRole` combined with EC2 or Lambda permissions (a classic privilege-escalation combo — you can hand a role to a new resource and inherit its permissions), and any `sts:AssumeRole` targeting a more privileged role than the one you're looking at.

---

## Doing this with almost nothing — inline policies and groups too

Don't stop at attached managed/custom policies. Two more places permissions hide:

```
aws iam list-user-policies --user-name <username>          # inline policies on a user
aws iam get-user-policy --user-name <username> --policy-name <policy-name>

aws iam list-groups-for-user --user-name <username>         # groups the user belongs to
aws iam list-attached-group-policies --group-name <group-name>
```

Inline policies specifically are worth checking every time — they're policies written directly onto a single user or role rather than attached as a standalone reusable policy, and because they don't show up in a general policy list, they're easy for the account owner to forget existed at all.

---

## Putting it together: the actual enumeration sequence, in order

1. `aws sts get-caller-identity` — confirm who you are (Module 00)
2. `aws iam list-users` — map every user
3. `aws iam list-roles` — map every role, inspect trust policies for wildcard/overly broad `Principal` entries
4. For each interesting role/user: `list-attached-*-policies` → `get-policy` → `get-policy-version` to read the actual permission JSON
5. `list-user-policies` / `get-user-policy` and `list-groups-for-user` — catch inline and group-inherited permissions the previous steps missed

At the end of this, you haven't exploited anything yet — you've built a complete permission map of the account using nothing but read-only calls that, on most misconfigured accounts, generate very little suspicion because they're the kind of calls legitimate automation makes constantly.

---

## What's coming in Module 02

With the map built, Module 02 covers turning specific findings from this enumeration into actual privilege escalation — the concrete `iam:PassRole` + EC2/Lambda technique flagged above, walked through end-to-end with real commands, plus the handful of other well-known IAM privilege-escalation paths (there are about 20 documented ones — you'll learn the mechanism, not just memorize the list).
