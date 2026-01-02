---
layout: default
title: "Mastering JWT Attacks: A Deep Dive into PortSwigger Labs (Part 2)"
date: 2025-12-02
categories: [Web Security, Walkthroughs]
toc: true
---

# 🧠 Exploiting JWT Vulnerabilities

In Part 1, we explored the theory behind JSON Web Token (JWT) attacks—how blind trust in header parameters and weak verification logic can lead to total system compromise.

Now, we move from theory to practice. This guide breaks down **seven distinct exploitation scenarios** found in the PortSwigger Web Security Academy. We will cover everything from simple signature bypasses to complex header injections (`jwk`, `jku`, `kid`) and algorithm confusion attacks.

---

## 🧪 LAB 1: Authentication Bypass via Unverified Signature

### 🧐 How the Vulnerability Exists
The application reads the user identity (the `sub` claim) from the token payload to identify the user but **completely fails to verify the cryptographic signature**. It assumes that if a token is present, it must be valid.

**Root Cause:** The developer parses the JSON payload immediately without invoking the `verify()` function.

### ⚠️ Preconditions
* A valid user account (`wiener`) to generate a baseline token.
* The application does not enforce signature checks on the backend.

### 🚨 Exploitation Steps

1.  **Capture Request:**
    Log in as `wiener`. Find the `GET /my-account` request in Burp History and send it to **Repeater**.

2.  **Modify Payload:**
    In the Burp **Inspector** panel (or JSON Web Token tab), change the `sub` claim from `wiener` to `administrator`.
 ![image](/images/Pasted image 20251205132147.png)

3.  **Execute Bypass:**
    Click **Apply changes**. Do **not** change the signature (keep the original one).
    Send the request. The server accepts the token because it ignores the signature entirely.

4.  **Solve Lab:**
    Access the admin panel and delete the user `carlos`.
    
![image](/images/Pasted image 20251205132129.png)

**IMPACT:** Full Account Takeover via trivial signature bypass.

---

## 🧪 LAB 2: Bypass via Flawed Signature Verification (None Algorithm)

### 🧐 How the Vulnerability Exists
The server relies on the token's header `alg` parameter to decide which verification algorithm to use. Crucially, it supports the insecure **"none"** algorithm, which tells the server "this token is not signed".

**Root Cause:** The JWT library or implementation allows `alg: none` in a production environment.

### ⚠️ Preconditions
* The application does not filter or reject tokens with `alg: none`.

### 🚨 Exploitation Steps

1.  **Capture & Modify:**
    Capture the authenticated request in **Repeater**.
    Change the JWT payload `sub` claim to `administrator`.

2.  **Set Algorithm to None:**
    In the JWT Header, change the `alg` parameter to `none`.   
![image](/images/Pasted image 20251205134124.png)

3.  **Strip the Signature:**
    In the raw message editor, **delete the signature bytes** at the end of the token.
    **CRITICAL:** You must keep the final trailing dot (`.`) to maintain the structure: `header.payload.`.

4.  **Execute:**
    Send the request. The server sees `alg: none`, skips verification, and grants admin access.  
![image](/images/Pasted image 20251205134139.png)

**IMPACT:** Authentication Bypass using the "None" algorithm.

---

## 🧪 LAB 3: Cracking a Weak Signing Key

### 🧐 How the Vulnerability Exists
The server signs JWTs using a symmetric algorithm (`HS256`) but uses a **low-entropy secret key** (e.g., a simple dictionary word).

**Root Cause:** Configuration mistake using a weak secret (`secret1`) instead of a cryptographically secure random string.

### ⚠️ Preconditions
* The token is signed using HMAC (HS256).
* The secret exists in common wordlists (like `rockyou.txt`).

### 🚨 Exploitation Steps

1.  **Get Token:**
    Copy the JWT string from your valid session cookie.

2.  **Crack with Hashcat:**
    Save the token to a file (`jwt.txt`). Run Hashcat using mode `16500`:
    ```bash
    hashcat -a 0 -m 16500 jwt.txt /path/to/wordlist.txt
    ```
![image](/images/Pasted image 20251205135910.png)
    
    *Result:* The tool reveals the secret is `secret1`.

3.  **Forge Token:**
    Use **jwt.io** or the Burp JWT Editor.
    Paste your token, change `sub` to `administrator`.
    Enter `secret1` into the signature verifier/signer box to generate a valid signature.
    
![image](/images/Pasted image 20251205140455.png)

4.  **Execute:**
    Replace your session cookie with the forged token and access `/admin`.

![image](/images/Pasted image 20251205135823.png)

**IMPACT:** Compromise of the server's signing key, allowing generation of any token.

---

## 🧪 LAB 4: JWK Header Injection

### 🧐 How the Vulnerability Exists
The server allows the client to embed the **public verification key** directly in the token header using the `jwk` (JSON Web Key) parameter.

**Root Cause:** The server blindly trusts the `jwk` provided in the header to verify the signature, instead of checking it against a trusted list.

### ⚠️ Preconditions
* The server processes the `jwk` header parameter.
* No whitelist is enforced.

### 🚨 Exploitation Steps

1.  **Generate Malicious Key:**
    In Burp, go to the **JWT Editor Keys** tab. Click **New RSA Key** -> **Generate**.
    
![image](/images/Pasted image 20251205150938.png)

2.  **Modify Payload:**
    In **Repeater**, switch to the **JSON Web Token** tab. Change `sub` to `administrator`.
    
![image](/images/Pasted image 20251205151123.png)

3.  **Inject JWK:**
    Click **Attack** -> **Embedded JWK**. Select your newly generated RSA key.
    *Note:* This automatically adds the `jwk` parameter to the header and signs the token with your private key.

![image](/images/Pasted image 20251205151159.png)

4.  **Execute:**
    Send the request to gain admin access.

**IMPACT:** Bypassing verification by forcing the server to use an attacker-supplied key.

---

## 🧪 LAB 5: JKU Header Injection

### 🧐 How the Vulnerability Exists
The server trusts the `jku` (JWK Set URL) header, which specifies a **URL** from which to fetch the public key. It fails to restrict this URL to trusted domains.

**Root Cause:** Missing whitelist on the `jku` URL allows fetching keys from an external attacker-controlled server.

### ⚠️ Preconditions
* The server has outbound network access to reach the attacker's exploit server.

### 🚨 Exploitation Steps

1.  **Host the Key:**
    Generate a new RSA Key in Burp.
    Copy the **Public Key as JWK**.
    On your **Exploit Server**, create a body `{"keys": [ <PASTE_KEY> ]}` and store it.
    
![image](/images/Pasted image 20251205155217.png)

2.  **Modify Headers:**
    In **Repeater**, add the header `"jku": "YOUR_EXPLOIT_SERVER_URL"`.
    Update the `kid` header to match the `kid` of your generated key.
    
![image](/images/Pasted image 20251205155241.png)

3.  **Sign & Execute:**
    Change `sub` to `administrator`.
    Click **Sign** (select your RSA key) and ensure **"Don't modify header"** is checked.
    Send the request. The server fetches your key from the URL and verifies the token.

**IMPACT:** Authentication Bypass combined with potential SSRF.

---

## 🧪 LAB 6: KID Header Path Traversal

### 🧐 How the Vulnerability Exists
The application uses the `kid` (Key ID) header to retrieve a key file from the local filesystem but fails to sanitize it, allowing **Directory Traversal** (`../../`).

**Root Cause:** Predictable file locations. Attackers can point the `kid` to `/dev/null`, which is empty, forcing the server to verify using an "empty" secret.

### ⚠️ Preconditions
* The server runs on a Linux-like system where `/dev/null` exists.

### 🚨 Exploitation Steps

1.  **Generate "Null" Key:**
    In **JWT Editor Keys**, generate a new **Symmetric Key**.
    Set the `k` parameter to `"AA=="` (Base64 for a null byte/empty string).
   
OR USE THIS PYTHON CODE, IT WILL GENERATE THE TOKEN.
![image](/images/Pasted image 20251205161305.png)

JUST USE THIS TOKEN TO SEND REQUEST OR IF YOU WANT YOU CAN TRY THE METHOD WITH (`AA==`) BUT I DIDN'T TRY.

![image](/images/Pasted image 20251205161233.png)

2.  **Modify Header:**
    Change the `kid` parameter to: `../../../../../../../dev/null`.
    
    Ensure `alg` is set to `HS256`.
![image](/images/Pasted image 20251205162226.png)

3.  **Sign & Execute:**
    Change `sub` to `administrator`.
    Sign the token using your "Null" key.
    Send the request. The server reads `/dev/null` (empty), matches it with your empty signature, and validates the token.
    
![image](/images/Pasted image 20251205161321.png)

**IMPACT:** Forcing the use of a known static secret (empty string) to forge tokens.

---

## 🧪 LAB 7: Algorithm Confusion (RS256 → HS256)

### 🧐 How the Vulnerability Exists
The server supports both asymmetric (`RS256`) and symmetric (`HS256`) algorithms but fails to ensure the key type matches the algorithm.

**Root Cause:** When the attacker forces `HS256`, the server uses its own **Public Key** as the **HMAC Secret**. Since the public key is public, the attacker can use it to sign tokens.

### ⚠️ Preconditions
* The server exposes its public key (e.g., via `/jwks.json`).
* The implementation allows switching `alg` from RS256 to HS256.

### 🚨 Exploitation Steps

1.  **Get Public Key:**
    Navigate to `/jwks.json` or extract the public key from the server's standard endpoints.
    
![image](/images/Pasted image 20251205170219.png)

2.  **Save Key:**
    In Burp **JWT Editor Keys**, create a New RSA Key and paste the server's JWK.
    
![image](/images/Pasted image 20251205170237.png)

3.  **Modify & Attack:**
    In **Repeater**, change `sub` to `administrator` and `alg` to `HS256`.
    Click **Attack** -> **HMAC Key Confusion**. Select the server's public key.
    
![image](/images/Pasted image 20251205170306.png)

4.  **Execute:**
    The extension signs the token using the Public Key as an HMAC secret. Send the request to gain access.

**IMPACT:** High-severity bypass using the server's own public data against it.

---

## ⚡ Fast Triage Cheat Sheet

| Attack Vector | 🚩 Immediate Signal | 🔧 The Critical Move |
| :--- | :--- | :--- |
| **Unverified Signature** | Changing payload doesn't break session. | Change `sub` to `admin`. **Keep original signature.**|
| **None Algorithm** | App accepts `alg: none`. | Set `alg: none`. **Remove signature bytes.** Keep trailing dot (`.`). |
| **Weak Key** | Token uses HS256 (HMAC). | **Hashcat:** `hashcat -a 0 -m 16500 jwt.txt wordlist`. |
| **JWK Injection** | No `kid` or `jku` enforced. | Burp JWT Tab -> **Attack** -> **Embedded JWK**. |
| **JKU Injection** | `jku` header exists. | Host JWK on exploit server. Point `jku` to URL. |
| **KID Traversal** | `kid` looks like a filename. | Set `kid` to `../../dev/null`. Sign with **empty key** (`AA==`). |
| **Algo Confusion** | `RS256` used + Public Key found. | Burp JWT Tab -> **Attack** -> **HMAC Key Confusion**. |
