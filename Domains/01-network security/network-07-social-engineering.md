# 07 - Social Engineering (Explained Simply)

## Why this comes right after malware
Everything from file `01` through `06` has been technical — scanning ports, exploiting software, running malware. Social engineering is the reminder that **the easiest way into a system is often not a technical exploit at all — it's convincing a person to just let you in.**

---

## The core idea, in one sentence
Social engineering is **tricking a person, not a computer** — using psychology (trust, urgency, fear, curiosity) to get someone to hand over access, information, or money voluntarily.

## Why attackers bother with this at all
Every technical vulnerability covered so far requires the target to have a specific, exploitable flaw. A person, on the other hand, can be tricked regardless of how well-patched or well-configured their systems are — this is exactly why social engineering remains one of the most common starting points for real-world breaches, even against organizations with strong technical defenses.

## Phishing — the most common form
**Phishing** is sending a fraudulent message (usually email) designed to look like it's from a trustworthy source, tricking the recipient into clicking a malicious link, downloading a malicious attachment, or entering their credentials on a fake login page.

**Spear Phishing** — a *targeted* version, aimed at one specific person or organization, using real, researched details about them (their actual boss's name, a real project they're working on) to make the message far more convincing than a generic, mass-sent phishing email.

**Whaling** — spear phishing specifically targeting senior executives or high-value individuals ("big fish"), since compromising one executive's account or authority can be worth far more than a regular employee's.

**Vishing** — phishing done over voice calls instead of email/text — e.g. someone calling and pretending to be from the bank's fraud department, creating urgency to extract information over the phone.

**Smishing** — phishing done via SMS text message.

## Pretexting — building a believable fake story
**Pretexting** is inventing a fabricated scenario (a "pretext") to convincingly justify why the attacker needs certain information or access. For example: calling an employee pretending to be from IT support, saying "we're doing a security audit and need to verify your login" — the fake scenario itself is what makes the request feel legitimate.

## Other common techniques

**Baiting** — leaving something tempting (a USB drive labeled "Salaries 2026" in a parking lot) for someone to find and plug in out of curiosity, which then runs malware the moment it's connected.

**Tailgating / Piggybacking** — physically following an authorized person through a secured door (badge-access entrance) without having your own access, relying on politeness ("could you hold the door?") to bypass a physical security control entirely.

**Quid Pro Quo** — offering something in exchange for information or access ("I'm from IT, I can fix that slow computer issue — just let me remote in real quick").

## Why social engineering works — the psychological levers being pulled
- **Urgency** — "your account will be locked in 10 minutes" pressures people to act before thinking carefully.
- **Authority** — pretending to be a boss, IT department, or law enforcement makes people less likely to question the request.
- **Fear** — "there's been suspicious activity on your account" triggers a panic response that overrides normal caution.
- **Trust/Familiarity** — using real names, real project details, or mimicking a colleague's writing style makes the request feel legitimate.
- **Curiosity** — an unexpected attachment or a found USB drive triggers "what is this?" before "should I open this?"

## Defense — why this category is genuinely different to defend against
Every other file in this series has a technical fix — a config change, a code fix, a patch. Social engineering's primary defense is **awareness and process**, not a technical control:
- Security awareness training, repeated regularly (a one-time training doesn't stick)
- Verification procedures for sensitive requests (e.g. "IT will never ask for your password over the phone, full stop, no exceptions")
- A clear, blame-free reporting process so employees who suspect they clicked something malicious report it immediately instead of hiding it out of embarrassment — the delay caused by fear of getting in trouble is often more damaging than the original mistake

## Quick-reference table

| Technique | Delivery method | Target |
|---|---|---|
| Phishing | Email | Broad/general |
| Spear Phishing | Email | One specific person/org |
| Whaling | Email | Senior executives specifically |
| Vishing | Phone call | Varies |
| Smishing | SMS text | Varies |
| Pretexting | Any (in-person, phone, email) | Varies |
| Baiting | Physical (USB, media) | Curiosity-driven |
| Tailgating | Physical presence | Physical access control |
