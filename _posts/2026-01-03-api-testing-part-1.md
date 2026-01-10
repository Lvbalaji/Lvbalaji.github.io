---
layout: default
title: "API Testing: The Theory & Mechanics (Part 1)"
date: 2026-01-03
categories: [Web Security, Theory, API]
toc: true
---

# API Testing: Hacking the Backend Logic

Application Programming Interfaces (APIs) act as the communication bridge between client applications (web, mobile, IoT) and the backend server. Because APIs centralize business logic and security, they represent a critical attack surface.

Unlike standard web applications where the server renders HTML, APIs typically process raw data formats like JSON or XML. Vulnerabilities here often stem from the API processing inputs in ways the developer didn't anticipate, such as accepting unexpected content types or binding user input to internal objects.

---

## 🗺️ 1. Reconnaissance: Mapping the Attack Surface

Before you can attack an API, you must understand its structure. API documentation is the blueprint of the application.

### A. Discovering Documentation
Developers often leave machine-readable documentation accessible to facilitate integration.
**Common Endpoints:** Look for standard paths like `/api`, `/swagger/index.html`, or `/openapi.json`.
**Path Truncation:** If you find a deep endpoint like `/api/swagger/v1/users/123`, systematically remove path components to find the root documentation:
    Check `/api/swagger/v1`
    Check `/api/swagger`
    Check `/api`.

### B. Identifying Hidden Endpoints
**JavaScript Analysis:** Client-side JavaScript files often contain hardcoded references to API endpoints that aren't immediately triggered by browsing the UI. Tools like the **JS Link Finder** BApp or manual review can reveal these.
**HTTP Method Enumeration:** An endpoint like `/api/tasks` might be known for `GET` requests. However, you should test other verbs (`POST`, `DELETE`, `PUT`, `PATCH`) to see if unadvertised administrative functionality exists.


---

## 🎭 2. Content-Type Manipulation

APIs rely on the `Content-Type` header to determine how to parse the request body. Manipulating this header can bypass security controls or trigger parsing errors that leak information.

### A. Bypassing Security Controls (WAFs)
Security mechanisms often look for specific patterns within specific content types.
**The Bypass:** If a WAF blocks SQL injection in a JSON body (`application/json`), changing the Content-Type to `application/x-www-form-urlencoded` might allow the same payload to pass through because the WAF stops inspecting it, yet the backend might still process it.

### B. Information Disclosure
Sending an unexpected Content-Type (e.g., sending `application/xml` to an endpoint expecting `application/json`) may cause the application to throw a verbose error.
**Impact:** These errors can reveal stack traces, file paths, or specific library versions used by the backend.

### C. Injection Attacks (XXE)
If an API defaults to processing JSON but also supports XML (even if not advertised), changing the Content-Type to `application/xml` can open the door to **XML External Entity (XXE)** attacks.
**Scenario:** You change the header and inject an XML payload referencing `/etc/passwd`.
**Result:** The parser processes the entity and returns the system file contents.

---

## 📦 3. Mass Assignment (Auto-Binding)

**Mass Assignment** occurs when an API automatically binds (assigns) incoming request parameters to internal object fields without filtering. Modern frameworks (Rails, Laravel, etc.) often do this to simplify code.

### The Mechanism
1.  **Legitimate Request:** A user sends `{"username": "peter", "email": "peter@test.com"}` to update their profile.
2.  **Internal Object:** The backend code does something like `User.update(request.body)`.
3.  **The Attack:** The attacker guesses sensitive field names by inspecting API responses (e.g., seeing `"is_admin": false` in a GET response).
4.  **The Exploit:** The attacker sends `{"username": "peter", "is_admin": true}`.
5.  **The Result:** The framework binds the `is_admin` parameter to the internal User object, escalating privileges.

---

## 🔗 4. Server-Side Parameter Pollution (SSPP)

**Server-Side Parameter Pollution** happens when the application accepts user input and blindly inserts it into a request sent to a *backend* internal API. This allows an attacker to manipulate the internal request structure.

### A. Query String Manipulation
If the frontend takes `name=peter` and creates an internal request `/internal_api?name=peter`, an attacker can inject syntax characters.
**Overriding Parameters:** Injecting a second parameter like `name=peter&name=admin`. Depending on the backend technology (PHP takes the last, Node.js takes the first, ASP.NET concatenates), this might force the internal API to return the admin's data.
**Truncation:** Using URL-encoded characters like `#` (`%23`) to truncate the rest of the internal query string (e.g., removing a hardcoded `&publicProfile=true` flag).

### B. REST Path Traversal
In RESTful APIs, parameters are often part of the URL path (e.g., `/api/users/{id}`).
**The Attack:** If the input is not URL-encoded/sanitized, an attacker can use path traversal sequences.
**Example:** Inputting `peter/../admin` might transform the internal call from `/api/users/peter` to `/api/users/peter/../admin`, effectively requesting the admin resource.

### C. Structured Data Injection (JSON/XML)
If the application places user input into a backend JSON request string without proper serialization.
**Input:** `peter","access_level":"admin`
**Resulting Internal JSON:** `{"name": "peter","access_level":"admin"}`.
**Impact:** The attacker has successfully injected a new JSON key-value pair, potentially altering access controls.


---

## 🛡️ 5. Prevention Strategies

Securing APIs requires a defense-in-depth approach.

1.  **Strict Allowlisting:** Only accept known, valid characters for input. Reject unexpected Content-Types.
2.  **Disable Mass Assignment:** Explicitly define which properties can be updated by the client (e.g., using Data Transfer Objects or `bind` lists). Never bind the entire request body to a model.
3.  **Validate Content-Types:** Enforce strict checks on the `Content-Type` header and ensure the body matches the header. Disable XML entity processing if XML is not needed.
4.  **Generic Error Messages:** Ensure the API returns generic errors ("Invalid Request") rather than stack traces or syntax errors.
5.  **Secure Documentation:** Do not expose Swagger/OpenAPI files on production environments without authentication.

---

## ❓ 6. Interview Corner: 10 Common Questions

If you are interviewing for an API Security or Pentest role, be prepared for these questions.

### Q1: What is Mass Assignment and how do you prevent it?
**Answer:**
Mass Assignment is a vulnerability where an API automatically binds client-provided data to internal object models. Attackers can exploit this to modify sensitive fields like `is_admin` or `balance`. To prevent it, developers should use **Allowlists** (Strong Parameters) to explicitly define which fields can be updated, or use Data Transfer Objects (DTOs) instead of binding directly to database entities.

### Q2: Why is the HTTP method important in API discovery?
**Answer:**
Developers might secure the standard methods (like `GET`) but forget to secure or remove unadvertised methods. For example, an endpoint might allow `GET /users` for everyone, but accidentally leave `DELETE /users` accessible. Testing `OPTIONS` or bruteforcing methods can reveal these hidden capabilities.

### Q3: How can changing the Content-Type header lead to a vulnerability?
**Answer:**
Changing the Content-Type (e.g., from `application/json` to `application/xml`) can trick the backend into using a different parser. This can lead to **XXE injection** if the XML parser is insecure, or **WAF bypass** if the firewall only inspects JSON payloads for malicious signatures.

### Q4: Explain Server-Side Parameter Pollution (SSPP).
**Answer:**
SSPP occurs when user input is unsafely embedded into a request made by the server to an internal backend API. An attacker can inject query syntax (like `&`, `#`, or `=`) to override parameters, inject new ones, or alter the logic of the internal request to access unauthorized data.

### Q5: What is the risk of exposing Swagger/OpenAPI files?
**Answer:**
Exposing these files provides attackers with a complete map of the API, including hidden endpoints, expected parameter types, and data structures. It significantly speeds up the reconnaissance phase and helps identify high-value targets like administrative endpoints.

### Q6: How does REST path traversal work?
**Answer:**
If an API uses user input to construct a URL path for an internal request (e.g., `/api/{user_input}`), and that input is not sanitized, an attacker can use `../` sequences. Inputting `user/../admin` could change the internal request to access the `admin` resource instead of the intended user resource.

### Q7: What is the difference between `PUT` and `PATCH`?
**Answer:**
Technically, `PUT` is idempotent and is used to replace an entire resource. `PATCH` is used to apply partial updates to a resource. In terms of security, `PATCH` is often more susceptible to Mass Assignment because it is specifically designed to update subset fields.

### Q8: How can error messages be a security vulnerability in APIs?
**Answer:**
Verbose error messages can leak implementation details. For example, a database error might reveal the SQL syntax (aiding SQL injection), or a stack trace might reveal file paths and library versions, helping attackers search for known CVEs.

### Q9: What is "Structured Data Injection" in SSPP?
**Answer:**
This happens when user input is inserted into a backend JSON or XML structure without proper encoding. An attacker can use format-specific characters (like `"` and `,` for JSON) to break out of the data field and inject new keys, such as `access_level: admin`.

### Q10: Why should you test old API versions (e.g., `/v1/`)?
**Answer:**
Organizations often improve security in newer versions (`/v2/`) but leave older versions running for backward compatibility. These older endpoints might lack newer security controls (like rate limiting or input validation), making them easier targets for exploitation.

---

## 🎭 7. Scenario-Based Questions

### 🎭 Scenario 1: The Hidden Admin Field
**Context:** You are testing a profile update endpoint `POST /api/profile`. The documentation only lists `name` and `email` as parameters.
**The Question:** How would you check for Mass Assignment?
**The "Hired" Answer:**
"I would first check the `GET /api/profile` response to see the full user object structure. If I see fields like `role`, `is_admin`, or `tier` in the response, I would try adding them to the `POST` body (e.g., `"role": "admin"`). If the application automatically binds this input to the object, I will have escalated my privileges."

### 🎭 Scenario 2: The Logic Bypass
**Context:** A WAF is blocking your SQL injection payload `' OR 1=1--` in a JSON request.
**The Question:** What is a simple bypass attempt involving headers?
**The "Hired" Answer:**
"I would try changing the `Content-Type` header to `application/x-www-form-urlencoded` or `text/xml`. If the WAF rules are only applied to JSON traffic, but the backend application framework automatically parses other formats, the payload might slip through the WAF and execute on the database."

### 🎭 Scenario 3: The Internal Query
**Context:** You notice your username `peter` is reflected in a query string for an internal API call: `GET /internal/search?user=peter`.
**The Question:** How do you test for SSPP to access `admin` data?
**The "Hired" Answer:**
"I would test for parameter overriding. I'd try injecting `peter&user=admin`. If the backend prioritizes the last parameter (like PHP), it might return the admin's data. I would also try truncation using hash characters, like `admin#`, to cut off any trailing parameters enforced by the server."

### 🎭 Scenario 4: The Deep Path
**Context:** You are scanning a host and find no obvious API documentation. You see a request to `/api/v3/mobile/config` in the proxy history.
**The Question:** How do you find the docs?
**The "Hired" Answer:**
"I would perform path truncation. I would send requests to `/api/v3/mobile`, `/api/v3`, and `/api`. I would also append common documentation paths to these bases, such as `/api/swagger.json`, `/api/openapi.yaml`, or `/api/docs`, to see if the schema definition is exposed at the root level."

### 🎭 Scenario 5: The XML Surprise
**Context:** An API explicitly sets `Content-Type: application/json`. You send XML data and get a "Syntax Error".
**The Question:** Is this interesting?
**The "Hired" Answer:**
"Yes. A syntax error indicates the parser *attempted* to read the XML but failed on the structure. This means XML parsing is enabled. I would immediately attempt an **XXE injection** by sending valid XML with a DOCTYPE definition to try and extract `/etc/passwd` or perform SSRF."
