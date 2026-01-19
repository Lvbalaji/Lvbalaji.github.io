---
layout: mission
title: "Business Logic Vulnerabilities: The Theory & Mechanics (Part 1)"
date: 2025-12-29
categories: [Web Security, Theory, Business Logic]
toc: true
---

# Business Logic Vulnerabilities: The "Human" Flaws

Business Logic Vulnerabilities (BLVs) are arguably the most unique and elusive class of web security issues. Unlike SQL Injection or XSS, which exploit technical syntax errors or unsafe handling of characters, BLVs exploit the **design and rules** of the application itself.

They occur when an attacker interacts with a legitimate workflow in an unanticipated way, causing the application to behave incorrectly. Because these flaws are specific to the unique logic of each application, automated scanners rarely find them—making manual testing critical.

---

## 🔸 1. What is "Business Logic"?

**Business Logic** defines the rules your application follows to decide *what can be done*, *by whom*, *when*, and *how*.

### The Amusement Park Analogy
Imagine an amusement park:
1.  You buy a ticket at the gate.
2.  You enter the park.
3.  You show the ticket to get on a ride.

The **Business Logic** dictates that you must pay *before* you ride.
A **Logic Flaw** exists if you can find a side entrance, jump a fence, or convince the ride operator that you already paid when you haven't.


---

## ⚙️ 2. How Do These Vulnerabilities Arise?

BLVs usually stem from developers making faulty assumptions about user behavior.

### 1️⃣ Excessive Trust in Client-Side Controls
Developers assume that if they hide a field or disable a button in the HTML/JavaScript, the user cannot change it.

**The Assumption:** "The price is set to $100 in the HTML `hidden` field, so the user will pay $100."

**The Reality:** Attackers use proxies (Burp Suite) to intercept the request and change the price to $0.01 before it reaches the server.

### 2️⃣ Flawed Assumptions About Behavior
Developers design workflows assuming users will strictly follow the intended path (A -> B -> C).

**The Assumption:** "A user will always go to the Checkout page before the Payment page."

**The Reality:** An attacker forces their browser to go directly to the Payment Confirmation page (Step C), skipping the actual Payment (Step B).

### 3️⃣ Complex Application Logic
In large applications, different developers work on different components. A lack of holistic understanding leads to "blind spots" where the security controls of one component fail to cover the inputs of another.

---

## 🧨 3. Common Categories of Logic Flaws

### 1. Excessive Trust in Client-Side Controls
This occurs when the server relies on the browser to validate input.
**Scenario:** An e-commerce site calculates the total price in JavaScript.
**Attack:** Intercept the `POST /checkout` request and change `price=1000` to `price=1`.

**Impact:** Theft of goods or services.

### 2. High-Level Logic Flaws
These are failures to enforce rules at the aggregate level.

**Scenario:** A bulk discount gives 10% off if you buy 10 items.
**Attack:** Add 10 items to get the discount, then remove 9 items before paying. The discount persists on the single item.
**Root Cause:** Failure to re-validate criteria at every state change.

### 3. Low-Level Logic Flaws (Unconventional Input)
These involve edge cases like negative numbers or integer overflows.
**Scenario:** A bank transfer feature checks if `amount < balance`.
**Attack:** Transfer `-1000` to a friend.
**Logic:** `(-1000 < 100)` is True. The system subtracts -1000 from your account (adding money) and adds -1000 to the victim (removing money).
**Impact:** Infinite money generation.

### 4. Weak Isolation on Dual-Use Endpoints
When one API endpoint handles multiple functions based on the presence of parameters.
**Scenario:** A password endpoint handles "Reset" (token required) and "Recovery" (email required).
**Attack:** Remove the `token` parameter entirely.
**Result:** The code falls back to the "Recovery" path but grants access intended for the "Reset" path, potentially bypassing authentication.

### 5. Infinite Money Logic Flaws
Domain-specific flaws involving coupons, store credit, or loyalty points.
**Scenario:** "Cancel Order" refunds $10 store credit.
**Attack:** Order an item, but don't pay. Cancel the order immediately. Receive $10 credit. Repeat 1000 times.
**Root Cause:** Crediting the user before verifying that the original payment was finalized.

### 6. Authentication Bypass via Encryption Oracle
This occurs when an app uses the same encryption mechanism for user input and internal tokens.
**Scenario:** A "Remember Me" cookie uses AES encryption. A "Encrypt Profile" feature also uses AES with the same key.
**Attack:** Encrypt the string "admin" using the Profile feature. Take the ciphertext and place it in your "Remember Me" cookie.
**Result:** The server decrypts it, sees "admin", and logs you in.

---

## 🛠️ 4. Detection Strategies

Since scanners fail here, you must rely on **Manual Logic Testing**.

### The "What If" Methodology
Ask yourself these questions while testing:
1.  **Skip Steps:** Can I go straight to the "Success" page?
2.  **Repetition:** Can I use this coupon twice? Can I transfer money 100 times in a second?
3.  **Data Manipulation:** What happens if I send a negative number? A huge number? A string instead of an int?
4.  **Limits:** What happens if I register a username that is 10,000 characters long? Does it truncate and create a collision? (e.g., `admin...` truncating to `admin`).

---

## 🛡️ 5. Remediation & Defense

1.  **Server-Side Validation:** Never trust the client. Always recalculate prices and verify permissions on the server.
2.  **Enforce Workflow State:** Use server-side state machines to track user progress. Ensure a user cannot reach Step 3 without completing Step 2.
3.  **Atomic Transactions:** Ensure that critical actions (like transfers) either fully complete or fully fail. Don't credit accounts before debiting them.
4.  **Logical Constraints:** Enforce positive integers for quantities and prices. Validate maximum lengths for strings.

---

## ❓ 6. Interview Corner: Common FAQs (Pentest & AppSec)

If you are interviewing for a security role, expect these questions on Business Logic.

### Q1: Why are Business Logic vulnerabilities hard to detect with automated scanners?
**Answer:**
Scanners look for technical signatures (like `' OR 1=1` for SQLi). Business logic flaws involve legitimate traffic that violates the *intent* of the application. A scanner doesn't know that "buying a TV for $1.00" is wrong; it just sees a valid HTTP request.

### Q2: Give an example of a "Race Condition" in business logic.
**Answer:**
A classic example is using a single-use coupon code in multiple parallel threads. If I send 20 requests simultaneously to apply the coupon, the server might read the "coupon unused" state for all 20 threads before it has time to write the "coupon used" state, applying the discount 20 times.

### Q3: What is "Mass Assignment" and how does it relate to business logic?
**Answer:**
Mass Assignment occurs when an application binds client-side input directly to internal objects. If an attacker adds `"isAdmin": true` to their profile update request, and the backend framework automatically binds this to the User object without filtering, the attacker creates a logic flaw allowing privilege escalation.

### Q4: How would you test for "Information Disclosure" via logic flaws?
**Answer:**
I would look for IDORs (Insecure Direct Object References). If the URL is `/orders?id=100`, I would test `/orders?id=101` to see if I can view someone else's order. This violates the logic rule that "Users should only see their own data".

### Q5: Explain "Cookie Tampering" in the context of business logic.
**Answer:**
If an application stores logic flags in cookies (e.g., `discount_applied=false` or `role=user`), an attacker can simply modify these values in the browser. The server should never store stateful logic in client-side storage without integrity checks (signatures).

### Q6: What is a "Truncation Attack"?
**Answer:**
Databases often have fixed limits (e.g., 255 chars). If I register with `admin[250 spaces]x`, the database might truncate the `x`, storing just `admin` (with trailing spaces trimmed). If the logic treats this as the existing `admin` account, I could take it over.

### Q7: What is an "Encryption Oracle"?
**Answer:**
It's a feature where the application encrypts user-supplied plaintext and returns the ciphertext using the same key/algorithm used for internal security tokens. Attackers use this to "forge" valid encrypted tokens by feeding the oracle the data they want to encrypt (like `role=admin`).

### Q8: How can "Rounding Errors" be exploited?
**Answer:**
In financial apps, if interest or fees are calculated with floating-point math and then rounded, an attacker might make thousands of small transactions where the rounding favors them (e.g., collecting fractions of a penny), leading to significant theft over time (salami slicing).

### Q9: What is the difference between client-side and server-side validation?
**Answer:**
* **Client-side:** JavaScript running in the browser. Useful for UX (feedback), but useless for security because it can be bypassed.
* **Server-side:** Logic running on the backend. This is the **only** trusted place to enforce security rules and business logic.

### Q10: How do you fix a "Workflow Bypass" vulnerability?
**Answer:**
Implement a server-side state machine. When a request comes in for "Order Confirmation," the server should check the user's session state. If the session state is not "Payment_Successful," the request should be rejected, regardless of the URL visited.

---

## 🎭 7. Scenario-Based Questions

### 🎭 Scenario 1: The Negative Transfer
**Context:** A banking app checks `if (transferAmount < currentBalance)`. You transfer `-1000`.
**The Question:** What happens and how do you fix it?
**The "Hired" Answer:**
"The logic `-1000 < 500` returns True. The transfer proceeds. The math `Balance - (-1000)` results in adding funds to the sender.
To fix this, the application must enforce `abs()` or check `if (transferAmount > 0)` before processing any transaction."

### 🎭 Scenario 2: The Coupon Stack
**Context:** You have a `20%` coupon and a `$5` coupon. The system blocks reusing the *same* coupon, but you can apply them alternately `A -> B -> A -> B` to reduce the price to zero.
**The Question:** What is the flaw?
**The "Hired" Answer:**
"The application likely uses a temporary variable for 'Current Coupon' rather than a persistent list of 'Applied Coupons'. When I switch coupons, it overwrites the check for the previous one. The fix is to maintain a list of all unique coupon codes applied to the session/cart and validate against that list."

### 🎭 Scenario 3: The 2FA Skip
**Context:** After login, you are redirected to `/2fa-verify`. You simply change the URL to `/dashboard` and get access.
**The Question:** Why did this work?
**The "Hired" Answer:**
"The application assumed that because you passed the password check, you are 'authenticated'. It treated the 2FA page as a UI step, not a security gate. The `/dashboard` endpoint failed to check if the session had the specific `2fa_completed` flag set."

### 🎭 Scenario 4: The Dual-Use Reset
**Context:** A password reset page takes `email` and `token`. If you delete the `token` parameter, it lets you reset the password without it.
**The Question:** What happened?
**The "Hired" Answer:**
"This is a weak isolation flaw. The code likely checks `if (token) { validate(token) } else { // Assume internal admin override }`. By removing the parameter, I fell into the logic path intended for administrators or a different function (like Account Recovery initiation) that doesn't require the token at that stage."

### 🎭 Scenario 5: Infinite Store Credit
**Context:** You buy an item, don't pay, cancel the order, and get $10 credit.
**The Question:** How do you exploit and fix this?
**The "Hired" Answer:**
"I exploit it by repeating the cycle to generate unlimited funds. The flaw is that the 'Credit Refund' logic triggered before the 'Payment Confirmed' state was verified. The fix is to ensure refunds/credits are only issued for transactions that have a confirmed 'Settled' or 'Paid' status in the database."

---

## 🛑 Summary of Part 1
1.  **Concept:** Exploiting valid application workflows to achieve unintended results.
2.  **Impact:** Financial loss, privilege escalation, data theft.
3.  **Attack:** Manipulation of price, quantity, workflow steps, and state logic.
4.  **Defense:** Server-side validation of all input and strict enforcement of workflow states.
