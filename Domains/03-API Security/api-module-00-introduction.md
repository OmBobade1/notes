# Module 00 - API Fundamentals & Types

## Why this comes before any vulnerability content
Every API vulnerability that follows depends on understanding what actually makes an API different from a normal website, and — critically — different API *types* have genuinely different attack surfaces. Testing a REST API and testing a SOAP API are not the same exercise, so knowing which one you're looking at first is not optional.

---

## What an API actually is, in plain terms
An API (Application Programming Interface) is a defined way for two pieces of software to talk to each other — not a human clicking buttons on a webpage, but one program directly requesting data or actions from another. A mobile banking app doesn't render its own account balance out of thin air — it calls an API, gets back raw data (usually JSON), and the app itself decides how to display it.

**Why this distinction matters for security specifically:** a normal website's HTML gives a browser (and a tester) a lot of visual, structural hints about what's happening. An API just returns data — there's no visual page to inspect, which means understanding the *specific technology and structure* of the API you're facing matters far more than it would for a standard webpage.

---

## REST (Representational State Transfer)

**What it is:** Not a strict protocol, but an *architectural style* — a set of conventions for structuring APIs around resources (things like "a user," "an order") accessed via standard HTTP methods and URLs.

**The core convention:** a resource is identified by a URL, and the *action* taken on it is expressed through the HTTP method used, not the URL itself.
```
GET    /users/101       → retrieve user 101
POST   /users           → create a new user
PUT    /users/101       → replace user 101's entire record
PATCH  /users/101       → partially update user 101
DELETE /users/101       → delete user 101
```

**Why REST is the dominant style today:** it's simple, maps naturally onto HTTP (which every web technology already understands), and is human-readable — you can often guess at a REST API's structure just by looking at its URLs, which is exactly why REST APIs are both easy to build and, from a security standpoint, relatively easy to explore and test.

**Data format:** almost universally JSON today (historically sometimes XML), which is why the sensitive data exposure and mass assignment-style vulnerabilities in later modules focus so heavily on JSON request/response bodies specifically.

---

## SOAP (Simple Object Access Protocol) & WSDL

**What SOAP actually is — this is the one you were trying to recall:** an older, much stricter, XML-based protocol for exchanging structured information between systems. Unlike REST's loose conventions, SOAP has a rigid, formally defined message format — every request and response is a specific XML structure, not flexible JSON.

**WSDL (Web Services Description Language)** — this is the actual term you were reaching for. A WSDL file is an XML document that formally, precisely describes exactly what a SOAP API can do — every available operation, what parameters each one expects, and what data type each response will contain. Think of it as a strict, machine-readable contract: unlike REST (where you often have to explore or guess at the structure), a SOAP API's WSDL file tells you, in full, exact detail, everything the API offers, if you can find/access it.

**A real SOAP request looks like this:**
```xml
<?xml version="1.0"?>
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Body>
    <GetUserDetails xmlns="http://example.com/api">
      <UserId>101</UserId>
    </GetUserDetails>
  </soap:Body>
</soap:Envelope>
```

**Why this matters for security specifically:** SOAP's XML-based nature means it connects directly to the **XXE (XML External Entity) content** already covered in this repo's web security series — a SOAP API is exactly the kind of target where an XXE vulnerability is realistically found, since the entire protocol is built on XML parsing by definition, not something bolted on as an occasional file upload feature.

**Where you'd actually still encounter SOAP today:** largely legacy enterprise systems, some banking/financial infrastructure, and certain government systems — SOAP has fallen out of favor for new development in favor of REST, but a huge amount of older, still-critical infrastructure runs on it, which is exactly why testers still need to recognize and understand it, not just the newer, trendier styles.

**Finding a target's WSDL file during testing:**
```
http://target.com/service?wsdl
```
Many SOAP services expose their WSDL directly at a predictable URL pattern like this — finding it hands you the API's complete, formal specification in one request.

---

## GraphQL

**What it is:** A query language for APIs (and a runtime for executing those queries) that flips REST's model — instead of the server deciding exactly what data each specific endpoint returns, the *client* specifies exactly what data it wants in a single request, potentially pulling from multiple resources at once.

**A real GraphQL query:**
```graphql
query {
  user(id: 101) {
    name
    email
    orders {
      id
      total
    }
  }
}
```
This single request pulls a user's name, email, and all their orders (with order ID and total) in one round trip — a REST equivalent might require three separate requests (`/users/101`, then `/users/101/orders`, etc.).

**Why this matters for security specifically:** GraphQL's flexibility is exactly its biggest security risk category — because the client controls the query shape, a poorly-secured GraphQL API can be asked for far more data, or far more deeply nested data, than the developers ever anticipated a single request needing, which is the basis of GraphQL-specific attacks covered in later API modules (introspection abuse, query depth/complexity attacks).

**Almost always a single endpoint**, unlike REST's many distinct URLs:
```
POST /graphql
```
Every single request — regardless of what data it's asking for — goes to this one URL, with the actual "what do you want" logic entirely inside the request body.

---

## gRPC (gRPC Remote Procedure Calls)

**What it is:** A modern, high-performance framework (built by Google) where a client directly calls functions/methods that appear to run locally, but actually execute on a remote server — using **Protocol Buffers (protobuf)**, a compact binary format, instead of JSON or XML.

**Why it's meaningfully different for testing:** because gRPC uses a compact *binary* format rather than human-readable JSON/XML, you generally can't just read the raw traffic the way you can with REST or SOAP — testing gRPC typically requires the API's `.proto` file (the formal definition of its available methods and data structures, conceptually similar to what a WSDL provides for SOAP) or specialized tooling capable of decoding the binary format.

**Where you'd encounter it:** increasingly common in modern microservices architectures, particularly for internal service-to-service communication (less commonly exposed directly to external/public clients) — mobile apps and internal backend systems are common real-world users of gRPC specifically for its performance advantages.

---

## HTTP Methods, Status Codes, and Authentication — the shared foundation across all types

**HTTP Methods (most relevant to REST/GraphQL specifically, since SOAP/gRPC largely ignore this distinction):**
| Method | Purpose | Security relevance |
|---|---|---|
| GET | Retrieve data | Should never change data — a GET request that modifies something is itself a design flaw |
| POST | Create new data | Common target for mass assignment (later module) |
| PUT | Replace an entire resource | Overwrites everything, not just specified fields |
| PATCH | Partially update a resource | Often where field-level authorization flaws show up |
| DELETE | Remove a resource | High-impact if authorization isn't properly checked |

**Status Codes worth knowing precisely, since they leak information:**
| Code | Meaning | What it can reveal to a tester |
|---|---|---|
| 200 | Success | The request worked |
| 401 | Unauthorized | You're not authenticated at all |
| 403 | Forbidden | You ARE authenticated, but not permitted — confirms the resource exists |
| 404 | Not Found | Resource genuinely doesn't exist (or is deliberately hidden this way) |
| 500 | Server Error | Something broke — often leaks stack traces (connects to the security misconfiguration content in the web series) |

**Why the 401-vs-403 distinction specifically matters for testing:** getting a 403 instead of a 404 when guessing at resource IDs (`/users/102` when you're only authorized for `/users/101`) confirms the resource *exists*, even though you can't access it — valuable reconnaissance in itself, and this exact behavior is what IDOR/broken access control testing (covered later) specifically probes for.

**Authentication approaches, at the overview level (detailed in later modules):**
- **API Keys** — a static, long-lived secret sent with each request, simple but risky if leaked (connects to the hardcoded secrets content in the web series).
- **OAuth 2.0** — a delegation framework letting a user grant one application limited access to their data on another service, without sharing their actual password.
- **JWT (JSON Web Tokens)** — already covered in full depth in the web security series (file `17`) — extremely common for API authentication specifically, not just traditional web sessions.

## Quick-reference table

| Type | Data Format | Single or multiple endpoints | Where you'll actually find it |
|---|---|---|---|
| REST | JSON (mostly) | Multiple, resource-based URLs | Everywhere — the current default |
| SOAP | XML (strict, WSDL-defined) | Often a single endpoint, action defined in the XML body | Legacy enterprise, banking, government |
| GraphQL | JSON | Almost always one single endpoint | Modern apps needing flexible, client-driven queries |
| gRPC | Protocol Buffers (binary) | Defined by `.proto` files | Internal microservices, performance-critical systems |
