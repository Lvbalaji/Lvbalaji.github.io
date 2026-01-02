---
layout: default
title: "JWT Attacks: The Theory & Mechanics (Part 1)"
date: 2025-12-05
categories: [Web Security, Theory, Authentication]
toc: true
---

# JWT Attacks: Understanding the Mechanics

Before we dive into cracking secrets and injecting headers, we must understand the fundamental architecture of Token-Based Authentication. The vulnerability here isn't just about weak passwords; it is about **trusting the wrapper**. It is about the disconnect between *reading* a token and *verifying* its integrity.

---

## 🔸 1. What is a JSON Web Token (JWT)?

A JWT is a stateless, compact, and URL-safe way of representing claims to be transferred between two parties. In modern web applications, it is the standard for **Stateless Authentication**.

### The Shift: Cookies vs. Tokens

* **Traditional (Stateful):** The server remembers the user (Session ID in a database). The client just holds the ID cookie.
* **JWT (Stateless):** The server forgets the user. The *client* holds all the data (User ID, Role, Permissions) in the token.

Because the server stores nothing, it relies entirely on the **cryptographic signature** to ensure the user hasn't tampered with that data.

### The Structure

A JWT consists of three parts separated by dots (`.`):

1.  **Header:** Metadata about the token (e.g., `{"alg": "HS256", "typ": "JWT"}`).
2.  **Payload:** The actual data or "claims" (e.g., `{"sub": "user", "admin": false}`).
3.  **Signature:** The cryptographic proof of integrity.

---

## 🧨 2. The Vulnerability: Blind Trust

The vulnerability exists when an application **trusts the token's contents** before or without properly verifying the cryptographic signature.

It typically appears when:
1.  **Client-Controlled Verification:** The application relies on headers sent by the client (like `alg`, `kid`, `jku`) to determine *how* to verify the token.
2.  **Flawed Logic:** The verification step is skipped entirely, or insecure algorithms (like "none") are accepted.
3.  **Weak Cryptography:** The signing keys are weak (brute-forceable) or exposed.

### The Core Assumption
Developers often assume that because a token *looks* signed (it has three parts), it *is* valid. They might decode the payload to read the `username` without checking if the signature matches that payload.

---

## 🧠 3. How does it work? (The Mechanics)

To verify a token, the server takes the **Header** and **Payload** from the user, hashes them with a **Secret Key**, and compares the result to the **Signature** provided in the token.

**The Flaw:**
In many libraries, the *instruction* on how to verify the token is inside the token itself (the Header).
* *Attacker:* "Hey Server, verify this token using the 'None' algorithm."
* *Vulnerable Server:* "Okay, I see `alg: none`. I will skip the signature check. Welcome, Admin."

---

## 🔍 4. The Dangerous Headers: KID, JKU, JWK

Why do these headers even exist? In complex systems (like Microservices or OAuth), a server might rotate keys weekly or use different keys for different users. These headers help the server figure out **"Which key should I use to verify this token?"**

The vulnerability arises because the server trusts the *client* to tell it where to find the key.

### 🆔 `kid` (Key ID)
* **Legitimate Use:** It stands for **Key ID**. It is a hint string (like a filename or database ID) that tells the server: *"I was signed with Key #45, go look that up in your database."*
* **The Exploit:** If the server uses this input directly in a filesystem call (e.g., `open('/keys/' + kid)`), an attacker can perform **Directory Traversal**.
    * *Attack:* Set `kid: "../../../dev/null"`.
    * *Result:* The server reads `/dev/null` (empty), uses an empty string as the "secret," and verifies a forged token signed with an empty string.

### 🔗 `jku` (JSON Web Key Set URL)
* **Legitimate Use:** It stands for **JWK Set URL**. It tells the server: *"I was signed with a key that is hosted at this URL. Go fetch it to verify me."*
* **The Exploit:** If the server fetches *any* URL provided, the attacker can point it to their own server.
    * *Attack:* Set `jku: "https://evil.com/keys.json"`.
    * *Result:* The server downloads the attacker's public key from `evil.com` and uses it to verify the attacker's forged token.

### 🔑 `jwk` (JSON Web Key)
* **Legitimate Use:** It allows the token to carry its own public key embedded directly in the header. This is rarely used in production but is part of the spec.
* **The Exploit:** The attacker embeds their own malicious public key in the header.
    * *Attack:* Inject a generated RSA Public Key into the `jwk` parameter.
    * *Result:* The server blindly trusts this embedded key and uses it to verify the signature (which the attacker created with the matching private key).

---

## 📍 5. Where does it behave dangerously? (Attack Surface)

JWTs are dangerous when used in these contexts:

| **Context** | **Dangerous Behavior** | **Impact** |
| :--- | :--- | :--- |
| **Session Management** | Modifying `sub` or `role` claims without signature verification. | **Account Takeover** |
| **Information Storage** | Storing sensitive data (passwords, flags) in the payload. | **Info Disclosure** |
| **Key Retrieval** | Using `jku` or `kid` to fetch keys from external/local paths. | **SSRF / LFI** |

---

## 🛠️ 6. Attack Mechanics: How we inject

We don't just "edit the token." We manipulate the cryptographic negotiation.

### 1️⃣ Basic Signature Bypass (Unverified Signature)
The most embarrassing flaw: The server reads the payload but never calls the `verify()` function.
* **Technique:** Decode the token, change `"admin": false` to `"true"`, and send it back with the **original** signature.
* **Goal:** Privilege Escalation.

### 2️⃣ The "None" Algorithm
The `none` algorithm is a standard meant for debugging. It tells the server "this token is not signed."
* **Technique:** Set header to `{"alg": "none"}` and **remove the signature** bytes (but keep the trailing dot `.`).
* **Payload:** `<Header>.<Payload>.`

### 3️⃣ Weak Secret (Brute-Force)
If the token uses symmetric signing (`HS256`), the security relies entirely on the password/secret.
* **Technique:** Use tools like **Hashcat** (`mode 16500`) with a wordlist (e.g., `rockyou.txt`) to recover the secret.
* **Goal:** Once you have the secret, you can mint your own valid tokens.

### 4️⃣ Algorithm Confusion (RS256 → HS256)
The server supports both Asymmetric (RS256) and Symmetric (HS256).
* **Technique:** You verify the token using the server's **Public Key** but tell the server you are using **HS256**.
* **The Glitch:** The server looks at `HS256`, treats its own Public Key as a "Shared Secret" (HMAC), and verifies the signature. Since you (the attacker) have the Public Key, you can sign malicious tokens using that Public Key as the secret.

### 5️⃣ Header Parameter Injection (`kid`, `jku`, `jwk`)
* **`jwk` Injection:** Embed your own public key in the token header. The server uses *your* key to verify *your* token.
* **`jku` Injection:** Point the `jku` URL to your exploit server containing a malicious JSON Key Set.
* **`kid` Traversal:** Set `kid` to `../../dev/null`. If the server reads this file, it sees an empty string. You sign the token with an empty string.

---

## 🛡️ 7. Remediation & Defense Strategies

Understanding the attack is only half the battle. Here is how to secure JWT implementations.

### 1️⃣ Hardcode the Algorithm
Do not let the header dictate the algorithm. Explicitly force the verification library to use the expected algorithm (e.g., `algorithms=["RS256"]`).

### 2️⃣ Verify the Signature First
Never decode a token to use its data before the signature verification passes. If verification fails, discard the request immediately.

### 3️⃣ Use Strong, Random Secrets
For HMAC (`HS256`), use a high-entropy secret (at least 256-bit). Never use simple dictionary words like "secret" or "admin".

### 4️⃣ Validate `kid` and `jku`
* **`kid`:** Whitelist allowed Key IDs. Do not use the input directly in filesystem calls.
* **`jku`:** Only fetch keys from trusted, whitelisted domains.

---

## ❓ 8. Interview Corner: Common FAQs

If you are preparing for a security role, expect these questions about JWTs.

### Q1: What is the difference between `jwt.decode()` and `jwt.verify()`?
**Answer:** `jwt.decode()` only Base64-decodes the token so you can read the payload. It **does not** check if the token is valid or tampered with. `jwt.verify()` calculates the hash and checks the signature. Using `decode` without `verify` is a critical vulnerability.

### Q2: Why is the "None" algorithm dangerous?
**Answer:** It allows an attacker to bypass the signature mechanism entirely. By setting `alg: none` and stripping the signature, the attacker can modify claims (like `admin: true`) and the server accepts them as valid.

### Q3: Can I store the user's password or social security number in a JWT?
**Answer:** **No.** JWT payloads are just Base64 encoded, not encrypted. Anyone who captures the token (via XSS or network sniffing) can decode it and read the data. Sensitive data should be kept on the backend.

### Q4: Explain the "Algorithm Confusion" attack.
**Answer:** It happens when a server supports both symmetric (HS256) and asymmetric (RS256) signing. An attacker sends a token signed with HS256 but uses the server's **Public Key** as the secret. If the server implementation is flawed, it uses its Public Key to verify the HMAC signature, allowing the attacker to forge tokens.

### Q5: How does `kid` injection lead to Directory Traversal?
**Answer:** The `kid` (Key ID) header is often used to lookup a key file. If the application doesn't sanitize this input, an attacker can set `kid: ../../../dev/null`. This forces the application to use an empty file as the secret key, allowing the attacker to sign tokens with an empty string.

### Q6: If I use `jku` to point to my own server, what prevents the victim server from just rejecting it?
**Answer:** Nothing, unless the developer explicitly implemented a whitelist. By default, many libraries will fetch whatever URL is in the `jku`. The defense is strictly whitelisting allowed domains (e.g., only allowing `https://auth.mycompany.com`).

### Q7: What is a "Replay Attack" in the context of JWTs, and how do we stop it?
**Answer:** A Replay Attack happens when an attacker captures a valid token and re-sends it later to impersonate the user. We stop this by:
1.  Using short expiration times (`exp` claim).
2.  Using HTTPS to prevent interception.
3.  Implementing a "jti" (JWT ID) claim and tracking used IDs to prevent reuse (though this adds statefulness).

---

## 🎭 9. Scenario-Based Questions

### 🎭 Scenario 1: The "Encrypted" Token
**Interviewer:** "A developer tells you their JWTs are safe because they are encrypted. You look at a token and it starts with `eyJ...`. What do you tell them?"
**Answer:** "That is likely **Encoded**, not **Encrypted**. `eyJ` is the Base64 header for a standard JSON object. Unless it's a JWE (JSON Web Encryption), the data is readable by anyone. I would decode it immediately to check for sensitive data leakage like passwords or PII".

### 🎭 Scenario 2: The Cross-Service Relay
**Context:** You have valid credentials for `App B` (a low-security forum). You notice `App A` (a high-security payment portal) uses the same JWT signing secret.
**Question:** How do you exploit this?
**Answer:** I would perform a **Cross-Service Relay Attack**. I can take my valid token from `App B` and send it to `App A`. If `App A` does not validate the `aud` (audience) claim, it might accept the token. If I have admin rights on `App B`, I might inadvertently gain admin rights on `App A`.

### 🎭 Scenario 3: The "Stateless" Logout Problem
**Interviewer:** "We want to ban a user immediately, but their JWT is valid for another hour. Since JWTs are stateless, we can't just delete their session. What do we do?"
**Answer:** "You have to introduce a 'Blocklist' (or Denylist). You store the `jti` (Unique Token ID) of the banned token in a Redis cache with a TTL equal to the token's remaining life. The middleware checks this cache for every request. This reintroduces some state, but it is necessary for immediate revocation."

---

## 🛑 Summary of Part 1
1.  **Concept:** JWTs allow stateless auth, trusting the client to hold session data.
2.  **Flaw:** Servers often trust the *Header instructions* (`alg`, `kid`) instead of their own config.
3.  **Attack:** We modify the header to trick the server into skipping verification or using a key we control.
