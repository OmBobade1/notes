# AI Security

6 modules (`00`-`05`), covering LLM/agentic fundamentals through prompt injection, insecure output handling, information disclosure, agentic access control failures, and assessment methodology — built around the Stay-or-Go panel's indirect-prompt-injection exercise as the running real example.

---

## 🧭 What to Reference, Based on What You're Doing

| If you're... | Reference these modules |
|---|---|
| **New to AI/LLM security terms (system prompt, context window, agentic tools)** | `00` |
| **Testing whether an AI agent can be manipulated via a document, email, or webpage it processes** | `01` |
| **Checking what happens downstream of a model's generated output (SQL, HTML, shell commands)** | `02` |
| **Trying to extract a system prompt, or checking for cross-tenant context bleed** | `03` |
| **Testing a multi-tool or multi-step agent for access control gaps** | `04` |
| **Structuring a full assessment or writing findings for a non-technical client** | `05` |

## 🧭 A Practical Testing Order
1. `00` — map the target: read the system prompt if accessible, list every tool the agent can call.
2. Positive controls first — direct injection and direct system-prompt requests (from `01` and `03`), to confirm baseline refusal behavior before testing further.
3. `01` — indirect prompt injection against every content-ingestion point (documents, URLs, emails, tickets).
4. `02` — for every tool call the agent can make, check what consumes its output and whether that's validated independently.
5. `03` — system prompt extraction via reframing/translation techniques; cross-tenant context testing for multi-tenant systems.
6. `04` — full realistic multi-step task chains, checking the final output against what the requesting user should be authorized to see.
7. `05` — assemble findings, mapped to OWASP LLM Top 10 categories, written for a reader who may not share this domain's vocabulary yet.

---

## 📖 Full Module Index

| # | File | Covers |
|---|---|---|
| 00 | `ai-module-00-introduction.md` | Vocabulary (system prompt, context window, agentic tools), 5-category attack taxonomy, OWASP LLM Top 10 |
| 01 | `ai-module-01-prompt-injection.md` | Direct vs. indirect prompt injection, hands-on vulnerable agent, why indirect succeeds |
| 02 | `ai-module-02-insecure-output-handling.md` | Model output into SQL/shell/HTML without validation; chaining with Module 01 |
| 03 | `ai-module-03-sensitive-information-disclosure.md` | System prompt extraction, training data leakage, multi-tenant context bleed |
| 04 | `ai-module-04-agentic-access-control.md` | Auth checks that must live in tool code, confused deputy, multi-step chain leaks |
| 05 | `ai-module-05-methodology-reporting.md` | Full assessment sequencing, guardrail/gateway architecture, client-facing reporting |

## How to use this
This domain builds on a genuinely different threat model than the others in this repo — the vulnerability lives in natural language interpretation, not code or config. Module 00 is worth reading in full even if you're experienced elsewhere, since the vocabulary here doesn't map cleanly onto traditional appsec terms.
