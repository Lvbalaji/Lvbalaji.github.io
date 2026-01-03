---
layout: default
title: "Mastering Access Control Attacks: A Deep Dive into PortSwigger Labs"
date: 2025-12-08
categories: [Web Security, Access Control]
toc: true
---

# 🧠 Understanding Access Control Vulnerabilities



Access control vulnerabilities occur when a user can act outside of their intended permissions. This manifests as IDOR (viewing another user's data), Privilege Escalation (becoming admin), or Missing Function Level Access Control (accessing hidden API endpoints).

This guide breaks down **thirteen** distinct exploitation scenarios found in the PortSwigger Web Security Academy.

---

## 🧪 LAB 1: Unprotected Admin Functionality

### 🧐 How the Vulnerability Exists
The application relies on "Security by Obscurity." The developer assumes that because the admin URL is not linked in the public UI, attackers cannot find it. However, the path is listed in `robots.txt` to hide it from search engines.

**Root Cause:** The server fails to check the user's role when the sensitive path is requested, relying solely on the secrecy of the URL.

### 🚨 Exploitation Steps

1.  **Check robots.txt:**
    Append `/robots.txt` to the lab's URL. Observe the `Disallow` entry (e.g., `Disallow: /administrator-panel`).

2.  **Access the Panel:**
    Navigate to that path (`/administrator-panel`) in your browser.

3.  **Solve:**
    Click the "Delete" button next to `carlos`.

    ![image](/images/Pasted%20image%2020251208213543.png)
    ![image](/images/Pasted%20image%2020251208213528.png)

---

## 🧪 LAB 2: Unprotected Admin Functionality with Unpredictable URL

### 🧐 How the Vulnerability Exists
The admin URL is randomized to prevent guessing, but it is embedded in the client-side JavaScript code so the legitimate admin's browser knows where to navigate.

**Root Cause:** Access control is based on "knowing the URL" rather than verifying session privileges.

### 🚨 Exploitation Steps

1.  **View Source:**
    Right-click the home page and select **View Page Source** (or `Ctrl+U`).

2.  **Identify URL:**
    Search for "admin" or "panel". Locate the variable revealing the hidden path (e.g., `var adminPanelTag = '<a href="/admin-s9x2...">`).

3.  **Solve:**
    Copy the path, visit it, and delete `carlos`.

    ![image](/images/Pasted%20image%2020251208213700.png)
    ![image](/images/Pasted%20image%2020251208213728.png)

---

## 🧪 LAB 3: User Role Controlled by Request Parameter

### 🧐 How the Vulnerability Exists
The application relies on a mutable client-side Cookie (`Admin=false`) to determine the user's privilege level without verifying it on the server.

**Root Cause:** Storing authorization state in the client (Cookie) instead of the server-side session.

### 🚨 Exploitation Steps

1.  **Intercept Login:**
    Log in as `wiener` and intercept the request in **Burp Proxy**.

2.  **Tamper:**
    Locate the cookie `Admin=false` and change it to `Admin=true`.

3.  **Solve:**
    Forward the request. Navigate to `/admin` and delete `carlos`.

    ![image](/images/Pasted%20image%2020251208214101.png)
    ![image](/images/Pasted%20image%2020251208214112.png)

---

## 🧪 LAB 4: User Role Can Be Modified in User Profile

### 🧐 How the Vulnerability Exists
The application suffers from **Mass Assignment**. The API endpoint for updating a profile accepts the internal `roleid` field if supplied in the JSON body.

**Root Cause:** The developer used a function that binds all incoming input to the user object without whitelisting allowed fields.

### 🚨 Exploitation Steps

1.  **Analyze JSON:**
    Update your email and intercept the request. Observe the JSON body structure.

2.  **Inject Parameter:**
    Add the role parameter to the JSON body:
    ```
    {
      "email": "wiener@normal-user.net",
      "roleid": 2
    }
    ```
   

3.  **Solve:**
    Send the request, verify the role update, and delete `carlos` from the admin panel.

    ![image](/images/Pasted%20image%2020251208215622.png)
    ![image](/images/Pasted%20image%2020251208215633.png)

---

## 🧪 LAB 5: User ID Controlled by Request Parameter (IDOR)

### 🧐 How the Vulnerability Exists
The application uses the `id` parameter in the URL (e.g., `?id=wiener`) to retrieve user details without checking if the ID belongs to the logged-in user.

**Root Cause:** Missing **Horizontal Access Control**. The system authenticates the user but fails to authorize access to the specific resource.

### 🚨 Exploitation Steps

1.  **Analyze URL:**
    Log in and note the URL: `/my-account?id=wiener`.

2.  **Modify:**
    Change the `id` parameter to `carlos`.

3.  **Solve:**
    The page loads Carlos's account. Copy his API Key to solve the lab.

    ![image](/images/Pasted%20image%2020251208215836.png)

---

## 🧪 LAB 6: IDOR with Unpredictable User IDs (UUIDs)

### 🧐 How the Vulnerability Exists
The application uses UUIDs to prevent ID guessing, but it leaks the UUIDs of other users in public areas, such as blog post author links.

**Root Cause:** **Information Leakage**. UUIDs only prevent enumeration; they do not replace access control checks.

### 🚨 Exploitation Steps

1.  **Find the Leak:**
    Find a blog post by `carlos`. Click his name and extract the UUID from the URL (`userId=xxxx...`).

2.  **Exploit:**
    Log in as `wiener`. Intercept the request to your account page.
    Replace your UUID with the one you copied.

3.  **Solve:**
    Retrieve Carlos's API Key.

    ![image](/images/Pasted%20image%2020251208220345.png)
    ![image](/images/Pasted%20image%2020251208220224.png)

---

## 🧪 LAB 7: IDOR with Data Leakage in Redirect

### 🧐 How the Vulnerability Exists
The application blocks unauthorized access by redirecting users (`302 Found`). However, it includes the sensitive account data in the response body *before* the redirect.

**Root Cause:** The server generates the full HTML response before checking permissions and appending the redirect header.

### 🚨 Exploitation Steps

1.  **Modify:**
    In **Repeater**, change the `id` parameter to `carlos`.

2.  **Inspect:**
    Observe the `302 Found` response. Do **not** follow the redirect.
    Look at the response **body** in the Repeater window.

3.  **Solve:**
    Extract the API Key from the raw HTML body.

    ![image](/images/Pasted%20image%2020251208221417.png)

---

## 🧪 LAB 8: IDOR with Password Disclosure

### 🧐 How the Vulnerability Exists
The "Edit Profile" form pre-fills the User object's data, including the password, into a hidden HTML input field.

**Root Cause:** Improper filtering of the database object before sending it to the frontend view.

### 🚨 Exploitation Steps

1.  **Modify:**
    Use IDOR to access the `administrator`'s profile page (`id=administrator`).

2.  **Inspect:**
    View the HTML source. Look for `<input type="hidden" name="password" value="...">`.

3.  **Solve:**
    Copy the password, log in as administrator, and delete `carlos`.

    ![image](/images/Pasted%20image%2020251208221702.png)
    ![image](/images/Pasted%20image%2020251208221641.png)

---

## 🧪 LAB 9: Insecure Direct Object References (Static Files)

### 🧐 How the Vulnerability Exists
Chat transcripts are stored as sequentially named static files (e.g., `2.txt`) and served directly by the web server without authentication checks.

**Root Cause:** Predictable naming conventions and lack of access control on static assets.

### 🚨 Exploitation Steps

1.  **Analyze URL:**
    Download a transcript and observe the URL: `/download-transcript/2.txt`.

2.  **Exploit:**
    Change the filename to `1.txt` to access the previous transcript.

3.  **Solve:**
    Read the chat log to find the user's password. Log in and solve.

    ![image](/images/Pasted%20image%2020251208222136.png)

---

## 🧪 LAB 10: URL-Based Access Control Circumvention

### 🧐 How the Vulnerability Exists
The frontend WAF blocks requests to `/admin`, but the backend framework supports the `X-Original-URL` header to override the path.

**Root Cause:** Architecture discrepancy. The frontend checks the request line, but the backend trusts the header.

### 🚨 Exploitation Steps

1.  **Inject Header:**
    Send a request to `GET /` (allowed) but add:
    `X-Original-URL: /admin/delete`

2.  **Craft Query:**
    Add the query parameters to the real URL:
    `GET /?username=carlos`

3.  **Solve:**
    The backend sees `/admin/delete` via the header and processes the request.

    ![image](/images/Pasted%20image%2020251208222410.png)
    ![image](/images/Pasted%20image%2020251208222844.png)

---

## 🧪 LAB 11: Method-Based Access Control Circumvention

### 🧐 How the Vulnerability Exists
The access control rule explicitly denies `POST` requests to the admin endpoint but fails to block `GET` requests.

**Root Cause:** Misconfiguration (`<limit POST>Deny</limit>`) and framework permissiveness (merging GET/POST parameters).

### 🚨 Exploitation Steps

1.  **Capture:**
    Capture the administrative `POST` request.

2.  **Bypass:**
    Right-click in Burp Repeater and select **Change request method** to convert it to a `GET` request.

3.  **Solve:**
    Send the request. The filter is bypassed, and the action executes.

    ![image](/images/Pasted%20image%2020251208224751.png)
    ![image](/images/Pasted%20image%2020251208224808.png)

---

## 🧪 LAB 12: Multi-Step Process Access Control Bypass

### 🧐 How the Vulnerability Exists
The administrative action is a multi-step process. The developer secured Step 1 (Selection) but failed to secure Step 2 (Confirmation).

**Root Cause:** Flawed logic assuming users cannot reach Step 2 without passing Step 1.

### 🚨 Exploitation Steps

1.  **Capture Step 2:**
    Identify the confirmation request (`POST /admin/promote-confirm`) using an admin account.

2.  **Switch User:**
    Log in as a standard user (`wiener`).

3.  **Replay:**
    Send the Step 2 request using `wiener`'s session cookie and parameters. The server executes it because it lacks a check on this specific step.

    ![image](/images/Pasted%20image%2020251208225712.png)
    ![image](/images/Pasted%20image%2020251208225722.png)

---

## 🧪 LAB 13: Referer-Based Access Control

### 🧐 How the Vulnerability Exists
The application verifies authorization by checking the `Referer` header to ensure the request came from the admin dashboard.

**Root Cause:** Using a client-controlled header for security decisions.

### 🚨 Exploitation Steps

1.  **Test Access:**
    Send an admin request as a standard user. It fails with 403.

2.  **Bypass:**
    Add or modify the header:
    `Referer: https://YOUR-LAB-ID.web-security-academy.net/admin`

3.  **Solve:**
    The server trusts the header and allows the request.

    ![image](/images/Pasted%20image%2020251208225148.png)
    ![image](/images/Pasted%20image%2020251208225206.png)

---

## ⚡ Fast Triage Cheat Sheet

| Attack Vector | 🚩 Immediate Signal | 🔧 The Critical Move |
| :--- | :--- | :--- |
| **Robots.txt** | `Disallow: /admin` entry. | Browse to the hidden path directly. |
| **Mass Assignment** | JSON body in profile update. | Add `"roleid": 2` or `"is_admin": true`. |
| **IDOR** | `?id=123` in URL. | Change ID to target victim or use array fuzzing. |
| **Redirect Leak** | `302` response with large body. | Inspect body in Repeater; do NOT follow redirect. |
| **Method Bypass** | `POST` is 403 Forbidden. | Change method to `GET` or `X-Method-Override`. |
| **URL Bypass** | URL blocked by WAF. | Use `X-Original-URL: /admin` on a valid path. |
