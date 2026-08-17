# API Security

10 modules (`00`-`09`), covering API fundamentals through the full OWASP API Security Top 10 — built to connect back to the web and network security series rather than repeat them.

---

## 🧭 What to Reference, Based on What You're Doing

| If you're... | Reference these modules |
|---|---|
| **Not sure what type of API you're even looking at (REST/SOAP/GraphQL/gRPC)** | `00` |
| **Changing an ID in a URL/request to see if you can reach someone else's object** | `01` (BOLA) |
| **Testing login, API keys, or tokens specifically** | `02` |
| **Reading a full API response for extra fields, or trying to write extra fields on update** | `03` |
| **Looking for an admin-only function reachable without a UI, or testing pagination/batch limits** | `04` |
| **Testing webhooks, URL-based file processing, or GraphQL resolvers** | `05` |
| **Found a legitimate action worth automating — checking if it should be rate-limited** | `06` |
| **Checking CORS, headers, or error verbosity on a JSON API specifically** | `07` |
| **Trying to find old API versions, shadow endpoints, or forgotten debug routes** | `08` |
| **Reviewing how your own app consumes a third-party API** | `09` |

## 🧭 A Practical Testing Order
1. `00` — identify the API type first; it changes which later modules even apply (SOAP won't have GraphQL-specific issues, for instance).
2. `08` — map the full API surface before testing anything, including old/undocumented versions — you can't test what you haven't found.
3. `01`-`04` — the core authorization/authentication/resource-limit checks, run against every discovered endpoint.
4. `05` — injection and SSRF, especially through webhooks and any GraphQL resolvers found.
5. `06` — step back and think about business-flow abuse at automation scale, independent of any single technical bug.
6. `07` — configuration-level checks (CORS, headers, error verbosity) across the whole surface.
7. `09` — if the app itself calls third-party APIs, review that consumption separately.

---

## 📖 Full Module Index

| # | File | Covers |
|---|---|---|
| 00 | `api-module-00-introduction.md` | REST, SOAP/WSDL, GraphQL, gRPC, HTTP methods, status codes, auth overview |
| 01 | `api-module-01-bola.md` | Broken Object Level Authorization — the API version of IDOR |
| 02 | `api-module-02-broken-authentication.md` | JWT in API context, API keys vs. user identity, credential stuffing on raw endpoints |
| 03 | `api-module-03-property-level-authorization.md` | Excessive Data Exposure + Mass Assignment |
| 04 | `api-module-04-function-level-auth-resource-consumption.md` | Function-level auth without a UI to hide behind, pagination/batch/GraphQL depth limits |
| 05 | `api-module-05-ssrf-injection.md` | SSRF via webhooks/file processing, GraphQL resolver injection, JSON-native NoSQLi |
| 06 | `api-module-06-sensitive-business-flows.md` | Legitimate actions abused at automation scale — scalping, promo abuse, fake reviews |
| 07 | `api-module-07-security-misconfiguration.md` | API-specific CORS/header/error-verbosity risks |
| 08 | `api-module-08-improper-inventory-management.md` | Shadow APIs, deprecated versions, forgotten debug endpoints |
| 09 | `api-module-09-unsafe-consumption.md` | Trusting third-party API responses without validation |

## How to use this
Each module explicitly references the specific web/network security file where the underlying concept was first covered in full depth — read those alongside this series rather than treating API security as a standalone topic; almost every module here is "the same core vulnerability, reached through an API-specific path."
