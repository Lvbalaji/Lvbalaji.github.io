---
layout: default
title: "Host Header Injection Lab Walkthrough"
date: 2025-12-01
categories: [Web Security]
toc: true
---

# 🧠 Understanding Host Header Vulnerabilities

The HTTP `Host` header is a mandatory request header (HTTP/1.1) that specifies the domain name the client wants to access.  
Vulnerabilities arise when an application **blindly trusts** this header without validating it server-side.

This walkthrough covers **five real-world exploitation scenarios** from PortSwigger Web Security Academy, showing how Host header issues can lead to **Account Takeover, Authentication Bypass, Web Cache Poisoning, and SSRF**.

---

## 🧪 LAB 1: Basic Password Reset Poisoning

### 🧐 How the Vulnerability Exists

The application constructs password reset links using the value from the incoming HTTP `Host` header.

**Root cause:** The backend assumes the `Host` header always reflects the legitimate domain.

### ⚠️ Preconditions

- Ability to trigger a password reset for another user  
- Access to an external exploit server to capture requests  

### 🚨 Exploitation Steps

#### 1. Baseline Analysis
- Trigger a password reset for your own account  
- Inspect the email and note the reset link format  
  Example:  
  `https://YOUR-LAB-ID.web-security-academy.net/forgot-password?...`

#### 2. Test for Poisoning
- Send the `POST /forgot-password` request to **Burp Repeater**
- Change the `Host` header to `example.com`
- Send the request  
- If the email link points to `example.com`, the vulnerability exists

#### 3. Execute the Attack
- Set the `Host` header to your exploit server domain  
  Example: `Host: YOUR-EXPLOIT-SERVER-ID.exploit-server.net`
- Change the `username` parameter to the victim (`carlos`)
- Send the request

#### 4. Steal the Token
- Check the exploit server access log
- Look for a request like:  
  `GET /forgot-password?temp-forgot-password-token=STOLEN_TOKEN`
- Copy the token

#### 5. Account Takeover
- Use the **real lab domain** with the stolen token
- Reset the victim’s password
- Log in as the victim

**IMPACT:** Full account takeover.

---

## 🧪 LAB 2: Host Header Authentication Bypass

### 🧐 How the Vulnerability Exists

Access to `/admin` is restricted to “local” users, determined solely by checking if the `Host` header equals `localhost`.

**Root cause:** Authorization decisions rely on user-controlled headers.

### ⚠️ Preconditions

- No IP-based access control
- Host header is trusted directly

### 🚨 Exploitation Steps

#### 1. Identify the Restriction
- Request `/admin`
- Observe a **403 Forbidden** response

#### 2. Bypass Access Control
- Send the request to **Burp Repeater**
- Change the `Host` header to `localhost`
- Send the request
- Receive **200 OK** and admin panel HTML

#### 3. Perform Admin Action
- Modify the request path to an admin action  
  Example: `/admin/delete?username=carlos`
- Keep `Host: localhost`
- Send the request

**IMPACT:** Unauthorized admin access.

---

## 🧪 LAB 3: Web Cache Poisoning via Ambiguous Requests

### 🧐 How the Vulnerability Exists

- The caching layer uses the **first** Host header as the cache key
- The backend uses the **second** Host header to construct script URLs

**Root cause:** Disagreement between cache and backend about which Host header is authoritative.

### ⚠️ Preconditions

- Page is cacheable
- Host header is reflected into HTML output

### 🚨 Exploitation Steps

#### 1. Identify Reflection
- Send a request with two Host headers
- Observe the second Host value reflected in script imports

#### 2. Prepare Malicious Script
- Host the expected JavaScript path on your exploit server
- Insert malicious JavaScript payload

#### 3. Poison the Cache
- Send repeated requests until you observe `X-Cache: hit`

#### 4. Verify
- Victims visiting the page now load your injected script

**IMPACT:** Stored XSS via cache poisoning.

---

## 🧪 LAB 4: Routing-Based SSRF

### 🧐 How the Vulnerability Exists

An intermediate reverse proxy routes requests internally based on the `Host` header without validation.

**Root cause:** Internal routing logic trusts user input.

### ⚠️ Preconditions

- Internal network exists (e.g., `192.168.0.x`)
- Reverse proxy is misconfigured

### 🚨 Exploitation Steps

#### 1. Detect SSRF
- Replace the `Host` header with a Burp Collaborator payload
- Observe DNS or HTTP interactions

#### 2. Scan Internal Network
- Send request to **Burp Intruder**
- **Uncheck** “Update Host header to match target”
- Set `Host: 192.168.0.§0§`
- Payload type: Numbers (0–255)

#### 3. Identify Target
- Look for response differences (302 vs 404/500)
- Note the responsive internal IP

#### 4. Exploit
- Set `Host` to discovered IP
- Request `/admin`
- Extract session or CSRF tokens
- Perform privileged action

**IMPACT:** Internal interface access via SSRF.

---

## 🧪 LAB 5: SSRF via Flawed Request Parsing

### 🧐 How the Vulnerability Exists

- Security control validates the **absolute URL** in the request line
- Routing logic uses the `Host` header instead

**Root cause:** Inconsistent request parsing between components.

### 🚨 Exploitation Steps

#### 1. Confirm Parsing Inconsistency
- Absolute URL passes validation
- Arbitrary Host header routes request internally

#### 2. Scan Internal Network
- Keep absolute URL unchanged
- Inject internal IPs via Host header using Intruder

#### 3. Exploit
- Identify internal admin endpoint
- Use absolute URL + internal Host header
- Execute administrative action

**IMPACT:** SSRF bypassing domain restrictions.

---

## ⚡ Fast Triage Cheat Sheet

- **Password Reset Poisoning**  
  Signal: Reset email reflects Host value  
  Action: Set Host to exploit server

- **Authentication Bypass**  
  Signal: `/admin` exists but returns 403  
  Action: Use `Host: localhost`

- **Cache Poisoning**  
  Signal: Second Host header reflected  
  Action: Duplicate Host + wait for cache hit

- **Routing SSRF**  
  Signal: Collaborator interaction via Host  
  Action: Intruder scan + disable Host auto-update

- **Parsing SSRF**  
  Signal: Absolute URL + external Host works  
  Action: Absolute URL + internal Host header
