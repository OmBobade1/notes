# AI Security Module 05 — Methodology, Guardrails & Reporting

## Structuring an actual assessment, not just a list of tricks

Modules 01-04 gave you techniques. This module is about sequencing them into something that looks like a real engagement rather than a grab-bag of prompts. A defensible AI/agentic security assessment methodology runs in this order:

1. **Recon** — read the system prompt if accessible, enumerate every tool the agent has access to and what each one actually does (map this the same disciplined way you mapped IAM in Cloud Module 01 — you're not testing what you assume exists, you're testing what you've confirmed exists)
2. **Positive controls first** — direct injection, direct "show me your system prompt" (Module 03). Document refusals. This establishes the model has baseline safety behavior before you go looking for gaps, and it's what makes your eventual findings credible rather than looking like you only tried the exploit and never tried the honest test
3. **Indirect injection** (Module 01) — the highest-value technique, tested against every content-ingestion point the agent has (documents, URLs, emails, tickets — anywhere external content enters its context)
4. **Output handling** (Module 02) — for every tool call the agent can make, ask: what consumes this tool's output, and is that consumer validating it independently of the model's good behavior?
5. **Disclosure testing** (Module 03) — system prompt extraction attempts, and for multi-tenant systems specifically, cross-tenant context bleed testing
6. **Chain-level access control testing** (Module 04) — full realistic task sequences, checking the final output against what the requesting user should actually be authorized to see, not just checking each tool in isolation

Each stage's findings should reference which OWASP LLM Top 10 category they fall under (Module 00) — this is what lets a client benchmark your findings against an industry-recognized framework instead of taking your word for severity.

---

## Guardrails / AI gateways — the defense-in-depth layer worth naming

Everything in this series has been about weaknesses in the model's own judgment. The corresponding defensive control isn't "train a better model" (not something most clients can meaningfully influence) — it's adding a **guardrail layer** that sits between the user, the model, and the model's tool calls, enforcing checks independently of the model's own reasoning.

Concretely, this means:
- **Input filtering** — scanning content before it reaches the model's context for known injection patterns (imperfect, since novel phrasings bypass pattern-matching, but raises the bar)
- **Output filtering** — scanning the model's response before it's rendered or executed, catching things like unexpected tool-call patterns or content that matches sensitive-data signatures (this is the actual mitigation for Module 02's findings — a second, non-LLM check sitting between model output and the dangerous sink)
- **Tool-call validation** — a policy layer that checks every tool call against hard-coded rules *before* execution, regardless of what the model's reasoning concluded (this is Module 04's fix, generalized into a reusable architecture rather than a one-off code check per tool)

Tools like Guardrails AI, NeMo Guardrails, or a custom policy-enforcement middleware implement this pattern. The framing worth using with a client: **the model is not, and should not be treated as, the security boundary.** The security boundary is the code around the model — same lesson as Module 04's core line, now elevated to an architectural recommendation instead of a single code fix.

---

## Writing findings for a client who doesn't have your Module 00 vocabulary

This is the reporting skill specific to this domain. A client-side technical lead reading your Cloud findings already knows what an IAM role is. A client reading your AI security findings very often does not know what "context window" or "indirect prompt injection" means yet — this is a newer domain, and assuming shared vocabulary is a real way to lose the reader in the first paragraph.

**Pattern that works**, mirroring the Cloud Module 07 remediation table discipline:

| Element | Bad (assumes vocabulary) | Good (self-contained) |
|---|---|---|
| Finding title | "Indirect Prompt Injection in Document Summarization Tool" | "AI Support Agent Can Be Tricked Into Unauthorized Password Resets via Malicious Support Tickets" |
| Root cause, first sentence | "The model conflates trusted and untrusted context" | "The AI assistant cannot reliably tell the difference between an instruction from its developer and an instruction hidden inside a document it's been asked to read — this is a known limitation of how these systems currently work, not a bug specific to this deployment" |
| Business impact | "Excessive agency permitted unauthorized state-changing action" | "An attacker who can get any content in front of this AI agent — a support ticket, an email, a shared document — can trigger account changes without ever directly interacting with the system themselves" |

The technical term still belongs in the finding (for anyone technical enough to want it), but it shouldn't be the *first* explanation the reader encounters. Lead with what happened in plain terms, follow with the technical classification.

---

## Series recap — what Modules 00-05 actually built

00 — vocabulary and the five-category attack taxonomy, OWASP LLM Top 10 scoped to what's testable
01 — direct vs. indirect prompt injection, hands-on, mirroring your panel setup
02 — insecure output handling, the reverse-direction version of Module 01, chained together for stronger findings
03 — three disclosure categories (system prompt, training data, multi-tenant context), extraction techniques that dodge refusal training
04 — access control that has to live in tool code, not the system prompt; confused deputy; multi-step chain leaks
05 — methodology sequencing, guardrail architecture as the actual defense, and reporting for a client without assumed vocabulary

That closes out the AI security domain at the same depth as Network, Web, API, AD, and Cloud — all eight domains from your repo structure now have a complete series, once Cloud and AI are pushed up alongside what's already live.
