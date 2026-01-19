---
layout: mission
title: "Mastering Race Condition Attacks: A Deep Dive into PortSwigger Labs (Part 2)"
date: 2026-01-10
categories: [Web Security, Exploitation, Race Conditions]
toc: true
---

# ⚔️ Exploiting Race Conditions: From Theory to Practice

In Part 1, we defined the "Race Window"—that critical split-second between a security check and a state update.

Now, we move to **exploitation**. This guide covers two specific attack vectors: exploiting **Limit Overruns** to stack coupons illegitimately, and utilizing **Single-Packet Attacks** to bypass rate limits for brute-forcing.

---

## 🧪 LAB 1: Limit Overrun Race Conditions

### 🧐 How the Vulnerability Exists
This lab demonstrates a business logic flaw caused by non-atomic database operations.

1.  **The Flaw:** The server enforces a "One coupon per order" limit by checking the database state. However, the **"Check"** (Is the coupon used?) and the **"Act"** (Apply the coupon) are separate steps.
2.  **The Mechanism:** By sending requests in parallel, we can force multiple requests to pass the "Check" phase simultaneously before the first "Act" phase writes the update to the database.
3.  **Session State:** The vulnerability persists because the cart state is stored server-side, allowing concurrent threads to modify the same session object.

### ⚠️ Preconditions
A state-changing endpoint (applying a coupon).
Ability to send parallel requests (using Burp Repeater Group or Turbo Intruder).

### 🚨 Exploitation Steps

1.  **Reconnaissance & Setup:**
    Log in and add the target item (Leather Jacket) to the cart.
    Capture the `POST /cart/coupon` request in **Burp Proxy** and send it to **Repeater**.
    **Verify Logic:** Apply the coupon once. Try again to confirm the error "Coupon already applied."
    **Reset:** Remove the coupon from the cart to clear the state for the attack.

2.  **Configuring the Attack Group:**
    In **Repeater**, right-click the coupon request tab and select **Add tab to group > Create tab group**. Name it "Race Test".
    Duplicate the tab (Right-click > Duplicate) until you have approximately **20 identical tabs** in the group.

    ![image](/images/Pasted%20image%2020260111200308.png)

3.  **Execution (Parallel Send):**
    Locate the **Send group** button (top left of Repeater).
    Select **Send group in parallel (last-byte sync)**.
    **Why this matters:** This uses Burp's synchronization technique to hold the last byte of every request and release them simultaneously, maximizing the chance of collision.
    Click **Send group (parallel)**.

4.  **Verification:**
    Analyze the responses. Instead of one `200 OK` and nineteen `400 Bad Request`, you should see multiple success messages.
    Refresh your browser cart. The discount should be stacked multiple times, allowing you to purchase the item.

    ![image](/images/Pasted%20image%2020260111200248.png)

**IMPACT:** Financial loss via unauthorized discount stacking.

---

## 🧪 LAB 2: Bypassing Rate Limits via Race Conditions

### 🧐 How the Vulnerability Exists
This lab exploits a race condition in the login failure counter to perform a high-speed brute force attack.

1.  **The Flaw:** The rate limiter works by reading the failure count from the database, incrementing it, and writing it back. This is not an atomic operation.
2.  **The Race Window:** If multiple login attempts arrive during the brief window between the read and the write, they all see the current count (e.g., "2 failures") and are allowed to proceed, even if the limit is 3.
3.  **Single-Packet Attack:** To exploit this over the internet, we use HTTP/2 to bundle multiple requests into a single TCP packet. This eliminates network jitter, ensuring all requests hit the application logic at the exact same microsecond.

### ⚠️ Preconditions
A rate-limited endpoint (Login).
The server supports HTTP/2.
Access to **Turbo Intruder** (Burp Extension).

### 🚨 Exploitation Steps

1.  **Benchmark:**
    Manually test the lock mechanism. Confirm the account locks after 3 failed attempts.
    *Optional Proof:* Use Repeater Group (Parallel) to send 20 bad logins. You will notice that more than 3 requests receive the "Invalid username" error before the "Account locked" error appears, proving the race window exists.

2.  **Configure Turbo Intruder:**
    Right-click the `POST /login` request and send to **Turbo Intruder**.
    Set the `username` to `carlos`.
    Place the `%s` marker in the `password` field.
    Select the **`race-single-packet-attack.py`** script template from the drop-down.

    **The Script Configuration:**
    `concurrentConnections=1`: Forces all requests into one TCP connection.
    `engine=Engine.BURP2`: Enables the HTTP/2 stack.
    `engine.openGate('1')`: The trigger that releases all requests simultaneously.

    ![image](/images/Pasted%20image%2020260111203458.png)

3.  **Execution:**
    Paste your password list into the clipboard.
    *Launch the attack. Turbo Intruder will queue the passwords and fire them in bursts.

4.  **Analysis:**
    Sort the results by **Status Code**.
    Look for a **302 Found**. This indicates a successful login, whereas `200 OK` indicates a failure (reloading the login page).
    Use the valid password to log in as Carlos and delete the user.

    ![image](/images/Pasted%20image%2020260111203359.png)

    ![image](/images/Pasted%20image%2020260111203347.png)

**IMPACT:** Authentication Bypass via Brute Force (evading lockouts).

---

## ⚡ Fast Triage Cheat Sheet

| Attack Vector | 🚩 Immediate Signal | 🔧 The Critical Move |
| :--- | :--- | :--- |
| **Limit Overrun** | "Coupon already applied" logic. | Use **Repeater Group (Parallel)** to send 20 requests. |
| **Rate Limit Bypass** | Account locks after N attempts. | Use **Turbo Intruder (Single-Packet)** to burst requests. |
| **Inventory/Stock** | "Only 1 item left" scenarios. | Attempt to purchase the same item ID in parallel sessions. |
| **Multi-Endpoint** | Payment logic is separate from Cart logic. | Send `POST /payment` and `POST /cart/add` simultaneously. |
