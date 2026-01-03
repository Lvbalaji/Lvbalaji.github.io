---
layout: default
title: "Access Control Vulnerabilities: The Theory & Mechanics (Part 1)"
date: 2025-12-08
categories: [Web Security, Theory, Access Control]
toc: true
---

# Access Control Vulnerabilities: Understanding the Mechanics

Before we can exploit broken access controls, we must understand the "Gatekeeper" logic that *should* be protecting the application. Access Control flaws are among the most common and critical vulnerabilities because they allow users to act outside their intended permissions—viewing private data, modifying accounts, or gaining administrative access.

---

## 🔸 1. What is Access Control?

**Access Control** (or Authorization) is the process of verifying whether an *authenticated* user has permission to perform a specific action or access a specific resource.

It answers the question: **"I know who you are, but are you allowed to do this?"**

### The Three Pillars of Access Control

1.  **Vertical Access Control:** Restricting access based on roles (e.g., User vs. Admin).
     *Failure:* A standard user accessing the `/admin` panel.
2.  **Horizontal Access Control:** Restricting access to specific resources belonging to a user (e.g., User A vs. User B).
     *Failure:* User A viewing User B's private messages.
3.  **Context-Dependent Access Control:** Restricting access based on the state of the application or workflow.
     *Failure:* A user skipping "Step 1: Payment" and jumping straight to "Step 2: Shipping".

---

## 🧨 2. The Vulnerability: The Missing Check

Access control vulnerabilities occur when a user can act outside of their intended permissions.

### The Core Flaw
The vulnerability exists because developers often implement **Authentication** (logging you in) but fail to implement robust **Authorization** (checking your rights) on every single request.

1. **Developer Mistake:** Trusting client-side input. The server assumes that because a user is logged in, they are authorized to perform *any* action they request.
2. **Security by Obscurity:** Assuming that hiding a button or using a secret URL is the same as securing the function.



---

## 🧠 3. The Million Dollar Question: How does it even work?

*If I don't see the 'Delete User' button, how can I delete a user?*

This relies on the disconnect between the **Frontend** (what you see) and the **Backend** (what the API accepts).

### A. The "Frontend" Illusion
The developer hides the "Admin Panel" link from your navigation bar because you are a standard user. This is merely a cosmetic restriction.

### B. The "Backend" Reality
The API endpoint `POST /admin/deleteUser` often still exists and is listening for requests. If the backend code does not explicitly check `if (user.role == 'ADMIN')` before executing the logic, anyone who knows (or guesses) the URL can trigger it.

### C. The Attack
The attacker does not need the button. They simply send the raw HTTP request (using Burp Suite) directly to the server. The server receives a valid request from a logged-in user and executes the action, unaware that the user "shouldn't" have been able to send it.

---

## 🧩 4. Root Causes: Why does it happen?

The vulnerability typically exists due to one of three failures:

### A. Insecure Direct Object References (IDOR)
The application uses user-supplied input (like an ID number) to access objects directly.
 *Example:* `GET /invoice?id=123`
 *Flaw:* The server fetches Invoice #123 without checking if it belongs to the current user.

### B. Parameter Tampering
The application relies on client-side values (Cookies, hidden form fields, or JSON parameters) to determine privileges.
 *Example:* A cookie `role=user` that an attacker changes to `role=admin`.

### C. Missing Function Level Access Control
The application fails to verify permissions at the function level (the API endpoint).
 *Example:* An attacker forces browsing to `/admin/delete` and the server accepts the request because the URL was "hidden" but not secured.

---

## 📍 5. Where does it behave dangerously? (Attack Surface)

Access Control flaws are dangerous in these specific contexts:

| **Component** | **Dangerous Behavior** | **Impact** |
| :--- | :--- | :--- |
| **User Profiles** | Changing `id=1` to `id=2` in URL | **PII Leak / Identity Theft** |
| **Admin Panels** | Accessing `/admin` or `/robots.txt` | **Full System Compromise** |
| **Multi-step Flows** | Skipping payment steps | **Business Logic Abuse** |
| **API Endpoints** | PUT /users/123 with `"role":"admin"` | **Privilege Escalation** |

**Bug Bounty Tip:** Watch for parameters like `role`, `admin`, `is_admin`, or `privilege` in JSON bodies and Cookies.

---

## 🛠️ 6. Attack Mechanics: How we inject

We don't just "guess IDs." We use specific techniques to manipulate the "pointer" to a resource or the "proof" of authority.

### 1️⃣ Fuzzing & ID Enumeration
Iterating through numerical identifiers to find valid resources belonging to others.
```http
GET /my-account?id=1001
GET /my-account?id=1002
```

_Goal:_ Viewing another user's private data (IDOR).

### 2️⃣ Role Swapping

Capturing a request as a high-privileged user (if possible) and replaying it with a low-privileged session token.

Goal: Verifying if the endpoint checks the token's permission or just the token's validity.

### 3️⃣ Method Tampering (Verb Smuggling)

Changing the HTTP method to bypass filters that only block specific verbs.


```
POST /admin/delete HTTP/1.1  -->  403 Forbidden
GET /admin/delete HTTP/1.1   -->  200 OK
```

_Goal:_ Bypassing Access Control Lists (ACLs) that are misconfigured to only deny `POST` requests.

### 4️⃣ Header Manipulation

Injecting headers that backend frameworks might trust for routing or authorization.


```
X-Original-URL: /admin/delete
Referer: [https://site.com/admin](https://site.com/admin)
```

_Goal:_ Tricking the backend into believing the request is authorized or coming from a trusted internal path.

---

## 🛡️ 7. Remediation & Defense Strategies

Understanding the attack is only half the battle. Defense against Access Control flaws requires a "Deny by Default" approach.

### 1️⃣ Server-Side Authorization (The Golden Rule)

Never rely on client-side controls (hidden buttons, cookies, or disabled fields). Every single request that performs an action or accesses data must include a server-side check:

if (currentUser.hasPermission('DELETE_USER')) { ... }.

### 2️⃣ Use Indirect Object References

Instead of using sequential database IDs (`id=123`), use unpredictable pointers like UUIDs (`id=a1b2-c3d4`) or session-based lookups.

1. _Bad:_ `GET /invoice/123`
  
2. _Good:_ `GET /invoice/view` (Server looks up the invoice for the _current_ user from the session).
  

### 3️⃣ Deny by Default

Configure your application framework to deny access to all resources unless explicitly allowed.

1. _Config:_ "All routes starting with `/admin` require `ROLE_ADMIN`."

---

## ❓ 8. Interview Corner: Common FAQs (Pentest & AppSec)

If you are preparing for a role as a **Penetration Tester** or **Application Security Engineer**, expect to be tested on these concepts.

### Q1: What is the difference between IDOR and Missing Function Level Access Control?

**Answer:**

1. **IDOR (Horizontal):** I can access the _same_ function (e.g., "View Invoice") but for a _different_ object (User B's invoice instead of mine). The failure is failing to check object ownership.
    
2. **Missing Function Level Access Control (Vertical):** I can access a _different_ function entirely (e.g., "Delete User") that I should not have access to. The failure is failing to check role permissions.
    

### Q2: Why is "Security by Obscurity" insufficient for Access Control?

Answer:

Security by Obscurity (like hiding the /admin URL or listing it in robots.txt) assumes the attacker cannot find the resource. However, attackers use tools like DirBuster, Burp Spider, and fuzzing to discover hidden paths. Once the path is found, if there is no actual access control check, the security is non-existent.

### Q3: How do you fix an IDOR vulnerability?

Answer:

The fix is to enforce an ownership check on the backend. When the server receives a request for id=123, it must query the database: "Does Resource 123 belong to the User currently logged in?". If not, return 403 Forbidden. Alternatively, remove the ID from the client-side request entirely and derive the target object solely from the user's session.

### Q4: Can changing the HTTP method (GET/POST) really bypass security?

Answer:

Yes. Some frameworks or web server configurations (like Apache .htaccess limits) might be configured to Deny POST requests to a specific URL but inadvertently allow GET or HEAD. If the application logic processes the input regardless of the method (e.g., using $_REQUEST in PHP which merges GET/POST), the attack will succeed.

### Q5: What is "Mass Assignment" and how does it relate to Access Control?

Answer:

Mass Assignment occurs when an application takes user input (like a JSON object) and binds it directly to an internal object model without filtering.

**Relation:** An attacker can inject restricted fields like `"is_admin": true` or `"role_id": 1` into their profile update request. If the model binds these fields blindly, the attacker effectively grants themselves administrative privileges.


---

### 🧠 Advanced Interview Questions

#### Q6: You found a UUID (e.g., `user_id=550e8400...`). Does this prevent IDOR?

Answer:

No. UUIDs prevent Enumeration (guessing the next ID), but they do not prevent Access Control attacks. If I can find another user's UUID (e.g., leaked in a public profile URL, a blog post, or an API response), I can still substitute my ID with theirs. If the server doesn't check ownership, the attack still works.

#### Q7: How would you test for Access Control flaws in a black-box assessment?

Answer:

I would use a matrix approach:

1. **Map:** Crawl the application to identify all endpoints and their intended privilege levels (Admin, Manager, User, Public).
    
2. **User Creation:** Create two users of each role (User A, User B, Admin).
    
3. **Horizontal Test:** Access User A's resources using User B's session.
    
4. **Vertical Test:** Access Admin endpoints using User A's session.
    
5. **Automate:** Use tools like Burp Suite's "Authz" or "AutoRepeater" extensions to replay requests with different session tokens automatically.
    

#### Q8: What is the risk of "Leaked Info in Redirects"?

Answer:

Sometimes an application will redirect unauthorized users to a login page (302 Found). However, due to sloppy coding, the server might generate the entire HTML body of the sensitive page (containing PII or API keys) before sending the redirect.

**Exploit:** Browsers automatically follow the redirect and discard the body. But an attacker using a proxy (Burp Suite) can intercept the response, ignore the redirect instructions, and read the sensitive data contained in the body.
    

#### Q9: How can `X-Original-URL` or `X-Rewrite-URL` headers bypass access controls?

Answer:

Some web frameworks (like Symfony or ASP.NET) allow these headers to override the requested path for routing purposes.

1. **Scenario:** A WAF sits in front and blocks `GET /admin`.
 
2. **Bypass:** The attacker sends `GET /` (which the WAF allows) but adds `X-Original-URL: /admin`.
  
3. **Result:** The backend application sees the header, trusts it, and serves the content for `/admin`, effectively bypassing the WAF's path-based restriction.
  

---

## 🎭 9. Scenario-Based Questions

### 🎭 Scenario 1: The "Unpredictable" URL

**Interviewer:** "We secured our admin panel by giving it a random, 32-character path that changes every deployment. We don't link it anywhere. Is this secure?"

Gold Standard Answer:

"No, this is Security by Obscurity.

1. **Leakage:** The URL might be leaked via `Referer` headers to external sites, browser history, or logs.
    
2. **Client-Side Code:** Often, the JavaScript for the main app contains references to this 'secret' path so the real admins can use it.
    
3. **The Fix:** You must implement an actual authentication and authorization check on the endpoint itself. Accessing the URL should require a valid session with `ROLE_ADMIN`."
    

### 🎭 Scenario 2: The Multi-Step Bypass

Context: A checkout process has 3 steps: 1. Review Cart -> 2. Enter Payment -> 3. Confirm Order.

Step 2 validates the credit card. Step 3 places the order.

**The Question:** How would you exploit this logic?

The "Hired" Answer:

"I would attempt Forced Browsing to skip Step 2.

1. I would complete a legitimate order to map the requests.
    
2. I would start a new order, add items to the cart (Step 1).
    
3. I would _skip_ the Payment request (Step 2) and send the request for Step 3 (`POST /order/confirm`) directly.
    
4. **Flaw:** If the application assumes 'User is at Step 3, so they must have passed Step 2', it will process the order without payment."
    

### 🎭 Scenario 3: The Static File

**Context:** Users upload sensitive medical documents. The download link is `/files/reports/report_101.pdf`. You check permissions when the user clicks "Download".

**The Question:** Is there a vulnerability here?

The "Hired" Answer:

"Likely, yes.

1. **Direct Access:** Even if the 'Download' button is protected, the _file itself_ might be served by the web server (Nginx/Apache) directly as a static asset.
    
2. **The Test:** I would copy the URL `/files/reports/report_101.pdf` and try to access it in a different browser (unauthenticated).
    
3. **The Fix:** Static files should not be public. You should store them outside the web root and serve them via a script that checks permissions (e.g., `GET /download?id=101`)."
 

### 🎭 Scenario 4: The "Hidden" ID

**Context:** A user profile update sends `{"username": "bob", "bio": "hi"}`. The ID is not in the JSON. The URL is `/api/profile`.

**The Question:** How do you test for IDOR if there is no ID parameter?

The "Hired" Answer:

"I would test for Mass Assignment and Blind Parameter Injection.

1. Even if the client doesn't send an ID, the backend model might accept one.
    
2. I would fuzz the JSON by adding `"id": 123`, `"user_id": 123`, or `"uid": 123` to the payload.
    
3. If the backend blindly binds this injected ID to the update logic, I might overwrite another user's profile despite the URL lacking an ID parameter."
 

---

## 🛑 Summary of Part 1

1. **Concept:** Access Control ensures users only act within their permissions.
    
2. **Flaw:** Trusting client-side input, security by obscurity, and failing to check ownership/roles on backend requests.
    
3. **Attack:** We use IDOR, Privilege Escalation, and Parameter Tampering to bypass these checks.
