---
layout: mission
title: "GraphQL Injection: The Theory & Mechanics (Part 1)"
date: 2026-01-05
categories: [Web Security, Theory, API]
toc: true
---

# 🧠 GraphQL Injection: Understanding the Mechanics

Before we can exploit GraphQL, we must unlearn how we treat traditional REST APIs. GraphQL is not just a different URL structure; it is a fundamentally different **architectural style** that shifts control from the server to the client.

This guide explores the architecture, discovery techniques, and the unique interaction models that make GraphQL a playground for hackers.

---

## 🔸 1. What is GraphQL?

GraphQL is a query language for APIs that acts as a strict **contract** between a client (frontend) and a server (backend). Unlike traditional REST APIs, where the server defines what data is returned, GraphQL allows the client to ask for **exactly what they need**—nothing more, nothing less.


### The "Single Endpoint" Paradigm

In a REST architecture, you might hit three different endpoints to load a profile page:
1. `GET /users/123` (User details)
2. `GET /users/123/posts` (User's posts)
3. `GET /users/123/followers` (Follower count)

In **GraphQL**, there is typically **one single endpoint** (e.g., `/graphql`). The client sends a complex query to this single URL, and the server stitches the data together in one response.

**Common Endpoint Names:**
1. `/graphql`
2. `/api/graphql`
3. `/v1/graphql`


---

## 🧨 2. The Vocabulary: How to Speak GraphQL

To hack it, you must speak it. GraphQL operations fall into three main categories:

### A. Queries (Read)
Equivalent to a `GET` request in REST. Used to fetch data.
```
query {
  user(id: 1) {
    name
    email
  }
}
```

### B. Mutations (Write)

Equivalent to `POST`, `PUT`, or `DELETE`. Used to create, update, or delete data.

```
mutation {
  updateUser(id: 1, email: "hacked@example.com") {
    success
  }
}
```

### C. Subscriptions (Listen)

WebSockets for real-time data updates.

---

## 🕵️‍♂️ 3. Discovery: Finding the Endpoint

Since GraphQL often lives on a single URL, finding it is step one.

### A. Manual Fingerprinting using `__typename`

If you suspect an endpoint (e.g., `/api`) might be GraphQL, send a universal probe. Every GraphQL schema has a reserved field called `__typename`.

**The Probe:**

```
curl -X POST [https://example.com/api](https://example.com/api) \
     -H "Content-Type: application/json" \
     --data '{"query":"{ __typename }"}'
```

The Signal:

If the response contains {"data": {"__typename": "Query"}}, you have confirmed it is a GraphQL API.

### B. Heuristic Scanning

Developers often expose interactive consoles like **GraphiQL**, **GraphQL Playground**, or **Voyager**.

**Fuzzing Targets:**

1.`/graphiql`
    
2. `/playground`
    
3. `/explorer`
    
4. `/altair`
    

---


## 🛠️ 4. Interaction Models: The Protocol

While GraphQL is "transport agnostic," it almost always rides on HTTP. However, _how_ it uses HTTP determines the attack surface.

### Method 1: POST (JSON) — The Standard

The client sends a `POST` request with `Content-Type: application/json`.

**Security:** This is generally secure against **CSRF** because browsers cannot send JSON with custom content types cross-origin without a CORS preflight check.
    

### Method 2: GET — The Weak Link

Some implementations allow queries via URL parameters:

GET /graphql?query={user{name}}

**Vulnerability:** If enabled, this opens the door to **CSRF**. An attacker can embed this URL in an `<img>` tag on a malicious site. If the victim visits it, their browser executes the query using their cookies.
    

### Method 3: POST (Form-Encoded) — The Bypass

Some APIs accept application/x-www-form-urlencoded.

query=mutation{...}&variables={...}

**Vulnerability:** Like GET requests, HTML forms can submit this content type cross-origin. If the API accepts this, it is vulnerable to CSRF.
    
---


## 🗺️ 5. Introspection: Mapping the Attack Surface

### What is Introspection?

Introspection is a built-in feature of GraphQL that allows you to ask the GraphQL server for information about its own schema. It is essentially an API querying itself.

In REST, you rely on external documentation (Swagger/OpenAPI). In GraphQL, the API **is** its own documentation. By querying the `__schema` field, you can retrieve:

1. **Queries:** How to fetch data.
    
2. **Mutations:** How to modify data.
    
3. **Types:** The structure of objects (User, Product, etc.).
    
4. **Directives:** Special instructions for the server.
    

### The Mechanics

When a developer sets up a GraphQL server, Introspection is usually enabled by default to help frontend teams build their queries.

**The Basic Probe:**

```
{
  __schema {
    types {
      name
    }
  }
}
```

If the server responds with a list of types (e.g., `Query`, `Mutation`, `User`, `InternalAdmin`), Introspection is active.

The "Full" Query:

Attackers use a massive, pre-built query (often called the "Kitchen Sink" query) to extract everything. This JSON response can be fed into tools like GraphQL Voyager or InQL to visualize the entire database structure, effectively reverse-engineering the backend in seconds.

### The Security Risk

Introspection turns a "Black Box" test into a "White Box" test.

1. **Hidden Fields:** You might see fields like `is_admin`, `reset_token`, or `legacy_password_hash` that are never used in the UI but exist in the backend.

2. **Shadow APIs:** You might find mutations like `deleteUser` or `debug_reset` that were meant for internal tools only.
    

### Bypassing Introspection Defenses

If a WAF blocks `__schema`, try:

1. **Whitespace Obfuscation:** `query { __schema { ... } }` (Add spaces/newlines).
    
2. **Method Flipping:** Try sending the introspection query via `GET` if `POST` is blocked.
    

---

## 📍 6. The Attack Surface (Theory)

Where do things go wrong?

|**Vector**|**Theory**|**Mechanism**|
|---|---|---|
|**BOLA / IDOR**|Arguments (`id: 1`) are trusted blindly.|Changing `product(id: 1)` to `product(id: 2)` accesses unowned data.|
|**DOS (Denial of Service)**|Nested queries consume server resources.|`user { posts { user { posts ... } } }` creates an infinite loop.|
|**Batching / Brute Force**|Sending multiple queries in one request.|Using **Aliases** (`login1: login(...)`, `login2: login(...)`) allows brute-forcing 100 passwords in a single HTTP request.|
|**Information Disclosure**|Verbose error messages.|"Did you mean 'password'?" suggestions leak hidden fields.|

---

### ⚡ Comparison Table: SOAP vs. REST vs. GraphQL

|**Feature**|**SOAP (Simple Object Access Protocol)**|**REST (Representational State Transfer)**|**GraphQL**|
|---|---|---|---|
|**Architecture**|Protocol (Strict Standard)|Architectural Style (Flexible)|Query Language (Strict Contract)|
|**Data Format**|XML only|Typically JSON (can use XML, HTML, etc.)|JSON|
|**Endpoints**|Exposes operations (e.g., `/getUser`)|Exposes resources (e.g., `/users/123`)|Single Endpoint (e.g., `/graphql`)|
|**Data Fetching**|Fixed structure defined by the server (WSDL).|Fixed structure defined by the endpoint. Risk of **Over/Under-fetching**.|Client specifies exact structure. No Over/Under-fetching.|
|**Transport**|HTTP, SMTP, TCP, UDP|HTTP (HTTPS)|HTTP (HTTPS)|
|**Operations**|Defined in WSDL (standardized).|Uses HTTP methods: GET, POST, PUT, DELETE.|Uses Queries (Read) and Mutations (Write) typically via POST.|
|**Caching**|Difficult (POST requests mostly).|Excellent (uses HTTP caching standard).|Complex (requires client-side libraries like Apollo).|
|**Security**|WS-Security (Built-in, Enterprise-grade).|Relies on HTTPS & OAuth/JWT.|Relies on HTTPS & OAuth/JWT.|
|**Performance**|Slower (Verbose XML parsing).|Fast (Lightweight JSON).|Fast (Fetches precise data in one trip).|

---

### 🚀 When to Use Which?

#### **1. SOAP (The Enterprise Standard)**

**What is it?** A highly structured, XML-based protocol known for its robustness and security standards.
    
**When to use:**
    
    1. **Financial & Telecommunication Services:** When you need strict security standards (WS-Security) and ACID compliance (Atomicity, Consistency, Isolation, Durability) for transactions.
        
    2. **Legacy Systems:** Many older enterprise environments are built entirely on SOAP.
        
    3. **Stateful Operations:** When the application state needs to be maintained across multiple requests (though REST can do this, SOAP was designed for it).
        

#### **2. REST (The Web Standard)**

**What is it?** An architectural style that treats server objects as "resources" that can be created, read, updated, or deleted using standard HTTP methods.
    
**When to use:**
    
    1. **Public APIs:** It is the standard for the web. If you are building an API for third-party developers, REST is expected.
        
    2. **Microservices:** Services are decoupled and can communicate easily over HTTP.
        
    3. **Simple Data Requirements:** When the data structure is simple and caching is important (e.g., a news site or blog).
        

#### **3. GraphQL (The Modern Flexible)**

**What is it?** A query language that acts as a contract between the frontend and backend, allowing the client to ask for exactly what it needs.
    
**When to use:**
    
    1. **Mobile Applications:** Bandwidth is expensive. GraphQL allows fetching complex data (e.g., User + Posts + Comments) in a single request, saving battery and data.
        
    2. **Complex Systems:** When you have many related entities (graph-like data) and REST would require too many round trips.
        
    3. **Rapid Iteration:** Frontend teams can change data requirements without asking backend teams to create new endpoints.

---

Here are example structures for both SOAP (XML) and REST (JSON) requests and responses.

### SOAP (Simple Object Access Protocol)

SOAP messages are strictly formatted in XML and must be wrapped in a specific "Envelope".

**SOAP Request**

This example requests the details of a user with ID `123`.

```
POST /UserService HTTP/1.1
Host: www.example.com
Content-Type: text/xml; charset=utf-8
Content-Length: length
SOAPAction: "http://www.example.com/GetUser"

<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Header>
    <Authentication>
      <Token>SecretToken123</Token>
    </Authentication>
  </soap:Header>
  <soap:Body>
    <GetUser xmlns="http://www.example.com/">
      <UserId>123</UserId>
    </GetUser>
  </soap:Body>
</soap:Envelope>
```

1. **`<soap:Envelope>`**: The root element that defines the XML document as a SOAP message.
    
2. **`<soap:Header>`**: Contains meta-data like authentication tokens (Optional).
    
3. **`<soap:Body>`**: Contains the actual request data (Function call `GetUser` and arguments `UserId`).
    

SOAP Response

The server responds with the user's data wrapped in the same envelope structure.
```
HTTP/1.1 200 OK
Content-Type: text/xml; charset=utf-8
Content-Length: length

<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <GetUserResponse xmlns="http://www.example.com/">
      <GetUserResult>
        <Id>123</Id>
        <Name>John Doe</Name>
        <Email>john.doe@example.com</Email>
      </GetUserResult>
    </GetUserResponse>
  </soap:Body>
</soap:Envelope>
```

**`<GetUserResult>`**: The actual return value of the function called in the request.
    

---

### REST (Representational State Transfer)

REST messages are typically JSON (though they can be XML) and rely on standard HTTP methods (GET, POST, etc.) rather than a specific envelope.

**REST Request**

This example creates a new user.
```
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer SecretToken123

{
  "name": "Jane Doe",
  "email": "jane.doe@example.com",
  "role": "admin"
}
```

1. **`POST`**: The HTTP method defining the action (Create).
    
2. **`/api/users`**: The endpoint (Resource) being accessed.
    
3. **Body**: A simple JSON object with the data payload.
    

REST Response

The server returns a standard HTTP status code and the created resource in JSON.

```
HTTP/1.1 201 Created
Content-Type: application/json

{
  "success": true,
  "data": {
    "id": 456,
    "name": "Jane Doe",
    "email": "jane.doe@example.com",
    "role": "admin",
    "created_at": "2023-10-27T10:00:00Z"
  }
}
```

1. **`201 Created`**: Standard HTTP status code indicating success.
    
2. **JSON Body**: The data returned is direct and lightweight, often without extensive wrappers.
  

###  GraphQL (The Query Language)

GraphQL requests are almost always sent as `POST` requests to a single endpoint. The "magic" happens inside the JSON body, where the client defines the exact structure of the data it wants using a `query` string.

**GraphQL Request**

This example fetches a user's details **and** the titles of their posts in a single request (something that might take two calls in REST).

```
POST /graphql HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer SecretToken123

{
  "query": "query getUser($id: ID!) { user(id: $id) { name email role posts { title } } }",
  "variables": {
    "id": "123"
  }
}
```

1. **`/graphql`**: The **Single Endpoint** that handles all requests, regardless of the data type.
    
2. **`"query"`**: This string defines the **contract**. The client is asking specifically for `name`, `email`, `role`, and `posts` (with just the `title`).
    
3. **`"variables"`**: Separating the data (`123`) from the query structure is a security best practice to prevent injection.
    

GraphQL Response

The server returns a JSON object where the structure exactly matches the query requested by the client.

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": {
    "user": {
      "name": "Jane Doe",
      "email": "jane.doe@example.com",
      "role": "admin",
      "posts": [
        { "title": "Why I love GraphQL" },
        { "title": "REST vs SOAP" }
      ]
    }
  }
}
```

1. **`"data"`**: The root key for a successful response.
    
2. **Mirrored Structure**: Notice that the response JSON hierarchy (`user` -> `posts`) mirrors the request exactly. No extra fields were sent (solving "Over-fetching"), and no fields were missing (solving "Under-fetching").

---

## ❓ 7. Interview Corner: Common FAQs

Q1: Why is GraphQL often considered more vulnerable to IDOR than REST?

Answer: In REST, authorization is often handled at the endpoint level (e.g., /admin/users). In GraphQL, there is only one endpoint. Authorization must be handled at the field level or resolver level. Developers often forget to check permissions for every single field in the graph, leading to IDOR (e.g., accessing a private_email field on a public User type).

Q2: How do you prevent Denial of Service in GraphQL?

Answer:

1. **Max Depth Limiting:** Reject queries deeper than _n_ levels (e.g., 5).
    
2. **Query Cost Analysis:** Assign a "score" to expensive fields and reject requests that exceed a budget.
    
3. **Rate Limiting:** Not just by HTTP request, but by the complexity of the query.
    

Q3: Is turning off Introspection enough to secure the API?

Answer: No. While it makes reconnaissance harder (security by obscurity), attackers can still guess fields using Suggestions (e.g., requesting productInfo and seeing the error "Did you mean 'productDetails'?") or by fuzzing common field names using tools like Clairvoyance.

Q4: Can Introspection be performed via a GET request?

Answer: Yes. If the API supports GET (often for caching purposes), you can pass the query as a URL parameter: /graphql?query={__schema{types{name}}}. WAFs configured to inspect POST bodies often miss this.

Q5: What is the difference between __schema and __type?

Answer: __schema retrieves the entire schema definition. __type(name: "User") is used to introspect a specific named type to see its fields and arguments.

Q6: Name a tool you would use to visualize an Introspection response.

Answer: GraphQL Voyager (for a visual node graph) or GraphiQL (for an interactive query IDE).

Q7: Why do developers leave Introspection enabled in production?

Answer: Often by accident or for the convenience of using tools like GraphiQL/Playground during debugging. They may assume that if they don't publish the URL, no one will find it.

Q8: What is a "Shadow API" in the context of GraphQL?

Answer: These are endpoints or mutations (like adminDeleteUser) that exist in the schema but are not connected to the frontend application. Introspection reveals them immediately.

Q9: How does the "Suggestions" feature aid an attacker?

Answer: It acts as a side-channel for schema discovery. Even if __schema is blocked, the verbose error messages ("Did you mean...") allow an attacker to brute-force valid field names.

Q10: What is the primary remediation for Introspection risks?

Answer: Disable Introspection in the production environment. Additionally, disable "Field Suggestions" to prevent fuzzing, and ensure the schema itself does not contain sensitive internal fields.


Q11: What is the specific GraphQL field used to trigger introspection?

Answer: The __schema field. It resides at the root of the query type and allows access to all types and directives in the API.

Q12: If Introspection is disabled, is the API safe from schema discovery?

Answer: No. Attackers can still use Field Suggestions (Clairvoyance). By sending an invalid field like user { nmae }, the server might reply "Did you mean 'name'?". Tools can automate this to fuzz and reconstruct the schema without Introspection.

Q13: How would you bypass a WAF rule that blocks the string __schema?

Answer: I would use whitespace, newlines, or encoding to break the signature.

1. `query { __schema { ... } }` (Standard)
    
2. `query { __schema\n { ... } }` (Newline injection)
    
3. `query { __schema, { ... } }` (Comma injection, if supported).
   
 
---


# 🎭 Scenario-Based Questions (Bar Raiser)

These scenarios test your ability to apply knowledge in real-world situations.

### 🎭 Scenario 1: The "Private" API

Context: You are testing an API. The developer claims, "We disabled Introspection, so you can't map our API." You send __schema and get a "400 Bad Request: Introspection disabled."

Question: How do you proceed to map the API?

The "Hired" Answer:

"I would pivot to Suggestion-Based Fuzzing (using a tool like Clairvoyance). I would send queries with common field names (e.g., user, account, email) but intentionally misspell them slightly. If the server responds with 'Did you mean...', I can recursively confirm which fields exist. I would also check if the API exposes a development console like /graphiql or /playground which might have its own internal schema access.".

### 🎭 Scenario 2: The WAF Block

Context: You send a POST request with __schema and get a 403 Forbidden from Cloudflare. You suspect it's a simple string-matching rule.

Question: Demonstrate three distinct ways to bypass this WAF rule.

The "Hired" Answer:

1. Method Flip: Send the query via GET instead of POST.
    
    GET /graphql?query={__schema{types{name}}}.
    
2. Padding/Obfuscation: Add non-printing characters or newlines that GraphQL parsers ignore but WAFs might fail to normalize.
    
    query { __schema%0a{ types { name } } }
    
3. Operation Name: Sometimes WAFs only look for __schema at the start. I would wrap it in a named query:
    
    query IgnoreMe { __schema { types { name } } }.
    

### 🎭 Scenario 3: The "Public" Schema

Context: A company has a public API (like Shopify) that must have Introspection enabled for third-party developers.

Question: How do they secure sensitive internal admin mutations while keeping the public schema open?

The "Hired" Answer:

"They should implement Schema Filtering (or Views). The public API endpoint should serve a restricted version of the schema that only includes public-facing types. Administrative mutations and internal fields should be separated into a different internal schema or endpoint that is not accessible publicly, or filtered out at the gateway level before the introspection response is sent.".

### 🎭 Scenario 4: The Hidden ID

Context: You successfully introspected the API and found a query user(id: ID!). You can query your own ID (100), but you want to find the Admin.

Question: You don't know the Admin's ID. How does Introspection help you find it?

The "Hired" Answer:

"Introspection helps me understand the Type System. I would check the User type definition to see if there are fields that might leak privilege levels, such as role, isAdmin, or permissions. Once I know the field names, I can query my own user to see the format (e.g., 'ROLE_USER'). Then, I would look for mutations that allow filtering users, like searchUsers(role: "ROLE_ADMIN"), which I can discover via the introspection of the Mutation or Query types.".

### 🎭 Scenario 5: The "Secure" POST

Context: The API only accepts JSON POST requests. The developer says, "We are safe from CSRF because we don't support GET or Form-Data."

Question: Is this true? How would you verify it during a pentest?

The "Hired" Answer:

"It is likely true, but I must verify the Content-Type Validation. I would attempt to send a POST request using application/x-www-form-urlencoded (query=mutation...) or text/plain. If the server accepts the request and processes the GraphQL query despite the wrong Content-Type, I can still perform a CSRF attack using a simple HTML form. The protection only holds if the server strictly rejects non-JSON content types.".

---

## 🛑 Summary of Part 1

1. **Concept:** Single endpoint (`/graphql`) replaces multiple REST endpoints.
    
2. **Protocol:** Clients specify exact data needs; Server responds with JSON.
    
3. **Risk:** Introspection leaks the entire API map; GET support enables CSRF; flexible queries enable DOS and Brute Force.

---
