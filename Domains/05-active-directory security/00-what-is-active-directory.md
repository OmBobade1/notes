# 00 — What is Active Directory? (Explained for absolute beginners)

> Read this first. Everything else in this folder builds on these ideas.

## The simple version

Imagine a school. The school has:
- A big **register** with every student's and teacher's name, ID, and class.
- **Classrooms** (groups) that organize people — "Class 10-A", "Teachers", "Sports Club".
- A **principal's office** that decides who is allowed into which room, who can see which files, and who is allowed to change the rules.

**Active Directory (AD)** is that system, but for computers instead of a school.

It is a service made by Microsoft that runs on Windows Server. Its whole job is to keep track of:
- **Users** (people who log in — like `john`, `admin`, `ethel.carla`)
- **Computers** (laptops, servers, printers on the network)
- **Groups** (like "IT Admins", "HR Team", "Domain Admins")
- **Permissions** (who is allowed to do what)

Almost every company you've heard of — banks, hospitals, government offices — uses Active Directory to manage their office computers. If you can log in to your company Windows laptop with one username/password and it magically gives you access to shared folders, printers, and internal websites, that's AD doing its job in the background.

## Why does AD matter for hacking (white-hat / pentesting)?

Because AD is basically **the keys to the whole kingdom**. If an attacker (or a penetration tester simulating an attacker) can take over Active Directory, they usually control every computer in that company's network. This is why "AD attacks" are one of the most important skills in real-world corporate penetration testing — almost every internal pentest engagement eventually turns into "can we become Domain Admin?"

## The building blocks (learn these words — you'll see them everywhere)

Think of it like a filing cabinet with folders inside folders inside folders:

| Term | Simple meaning | School analogy |
|---|---|---|
| **Object** | Anything stored in AD — a user, computer, printer, group | A student's record card |
| **OU (Organizational Unit)** | A folder used to organize objects | A classroom that holds students, desks, a whiteboard |
| **Domain** | A group of computers/users managed together, sharing one directory database | The whole school |
| **Domain Controller (DC)** | The actual server that runs AD and holds all this data | The principal's office / server room |
| **Tree** | A group of domains that share a naming pattern (e.g. `company.com`, `sales.company.com`) | A school with branch campuses that all use the same name |
| **Forest** | A group of trees that don't share a naming pattern but still trust each other | Two different schools under the same school board |
| **Trust** | A relationship where one domain agrees to accept logins/permissions from another domain | Two schools agreeing "if you're a student at School A, you can use School B's library too" |

### Trust relationships, simplified

- **One-way trust**: Domain A trusts Domain B, but not the other way around. (You can visit their library, they can't visit yours.)
- **Two-way trust**: Both domains trust each other.
- **Transitive trust**: If A trusts B, and B trusts C, then A automatically trusts C too. (Default between a parent domain and a child domain in the same forest.)
- **Non-transitive trust**: The trust does NOT pass along automatically — it only applies to the two domains directly involved.

Five common trust types you'll hear mentioned: **Parent-Child**, **Tree-Root**, **Forest**, **Shortcut**, and **External** trusts. You don't need to memorize all of these on day one — just know they exist and that trusts are basically "bridges" between domains that an attacker can sometimes walk across.

## What's next?

1. `01-lab-setup.md` — how to build your own practice Active Directory environment (safely, on your own machine — never on a real company's network without written permission).
2. `02-enumeration.md` — how to look around an AD environment once you have basic access.
3. `03-attack-kerberoasting.md`, `04-attack-asrep-roasting.md`, `05-attack-pass-the-hash.md`, `06-attack-golden-silver-ticket.md` — the core attacks.
4. `07-mitigations-and-defenses.md` — how defenders stop these attacks (equally important to know).

## ⚠️ Ground rules before you touch anything

- Only run these techniques against **your own lab VMs** or environments you have **explicit written permission** to test (e.g. HackTheBox, TryHackMe, a work engagement with signed authorization).
- Attacking a network without permission is illegal, regardless of intent.
- Keep write-ups from public platforms (HTB/THM) compliant with their publishing rules — usually you can only publish full write-ups for **retired** machines.
