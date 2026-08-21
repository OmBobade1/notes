# Cloud Module 00 — What Cloud Actually Is (From the Seat of Someone Attacking It)

## Forget the brochure definition

"Cloud computing" is not a concept you need to meditate on. It is this:

**AWS, Azure, and GCP are companies that rent you a computer, over the internet, that you control through an API instead of a power button.**

That's it. Instead of buying a physical server, racking it in a room, and plugging in a keyboard, you send AWS a request over HTTPS that says "give me a computer," and 30 seconds later you have one. You never touch it physically. Everything — turning it on, turning it off, deleting it, reading its files — happens through commands sent over the network.

This one fact is the entire reason cloud security is a different discipline from traditional pentesting. In a normal on-prem pentest, you're attacking a network, then a machine, then maybe an app running on it — three separate layers you have to fight through one at a time. In cloud, the **control plane is the target**, not the machine. If you get valid credentials, you don't need to "hack in" at all — you just ask the API for what you want, and it gives it to you, because that's literally what the API is for.

That's the mindset shift for this whole module series: you're not attacking a computer. You're attacking a **permission system**.

---

## The two ways anyone (attacker or admin) touches AWS

There is no "logging into a server." There are exactly two doors:

1. **The Console** — the website, `console.aws.amazon.com`. Point-and-click. This is what beginners use and what attackers almost never rely on, because it can't be scripted.
2. **The API** — every single thing the console button does, it does by calling an API endpoint behind the scenes. You can call those same endpoints directly, from a terminal, without ever opening a browser.

You will use the API directly, via a tool called the **AWS CLI** (Command Line Interface). This is non-negotiable to understand, because almost every real-world cloud attack — and every tool you'll use later in this series (Prowler, pacu, ScoutSuite) — is just software that calls the same API you're about to call by hand.

---

## Your first real command, broken down completely

Assume you have zero terminal experience. Here is the single most important command in cloud security, explained character by character.

```
aws sts get-caller-identity
```

Type this into a terminal where the AWS CLI is installed and configured. Here's what each piece means:

| Piece | What it actually is |
|---|---|
| `aws` | The program you're running. This is the AWS CLI tool itself, installed on your machine. |
| `sts` | The **service** you're talking to. AWS is not one thing — it's hundreds of separate services (S3 for storage, EC2 for servers, IAM for permissions, etc). `sts` = Security Token Service, the service that tells you who a set of credentials belongs to. |
| `get-caller-identity` | The **action** you're asking that service to perform. Every AWS CLI command follows this pattern: `aws <service> <action>`. |

Run it, and you get back something like:

```json
{
    "UserId": "AIDAABCD1234EFGH5678I",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/dev-test-user"
}
```

**Why an attacker runs this literally first, every single time:** if you find AWS credentials anywhere — leaked in a GitHub repo, a `.env` file, an S3 bucket, a Jenkins config — this is the command you run before anything else. It costs nothing, it's not a destructive action, and it instantly tells you three things: which AWS account you've landed in, which specific IAM identity these credentials belong to, and (from the `Arn` format) whether it's a low-value user or something juicier like a role with broader trust. Everything you do next depends on this one answer.

---

## How credentials actually get onto your machine (so the command above works at all)

The CLI doesn't magically know who you are. It reads credentials from a plain text file at:

```
~/.aws/credentials
```

Which looks like this:

```ini
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

Two values. That's the entire "key" to an AWS account:

- **Access Key ID** (`AKIA...`) — think of this as the username. Not secret. Shows up in logs, error messages, sometimes even git history.
- **Secret Access Key** — this is the actual password. If this leaks, whoever has it can run `aws sts get-caller-identity` and every other command as that identity, from anywhere in the world, with no MFA prompt, no login page, nothing stopping them — unless the account owner has explicitly restricted it.

This is *the* number one real-world cloud breach pattern: a developer commits `.aws/credentials` or hardcodes these two strings into a script, pushes to a public GitHub repo, and within minutes automated bots (this really happens — GitGuardian and similar scanners exist specifically because of this) have scraped the key and are running exactly the command above against your account.

---

## Why "403 Forbidden" from the API is actually useful to you

When you run a command you're not allowed to do, you don't get a vague error — you get a specific, information-leaking one:

```
An error occurred (AccessDenied) when calling the CreateUser operation: 
User: arn:aws:iam::123456789012:user/dev-test-user is not authorized 
to perform: iam:CreateUser on resource: *
```

Read that again. AWS just told you, for free:
- The exact IAM action name (`iam:CreateUser`) — useful later for IAM enumeration
- Confirmation the identity exists and is valid (bad creds give a different error — `InvalidClientTokenId`)
- That this specific permission is denied — which, if you're probing systematically, lets you build a map of what you *can* do by process of elimination

This is different from web app pentesting, where a 403 usually just means "go away." In AWS, a 403 is a data leak about the permission structure itself.

---

## What's coming in Module 01

Now that you can talk to the API and read what it tells you, Module 01 covers **IAM enumeration** — using nothing but low-privilege, "boring" permissions (the kind developers leave attached to service accounts by accident) to map out an entire AWS account's users, roles, and policies without ever needing admin access. This is the actual first move in almost every real cloud compromise, and it's 100% API calls, no console, no guessing.
