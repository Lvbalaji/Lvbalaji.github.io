---
layout: default
title: "Clickjacking (UI Redressing): The Theory & Mechanics (Part 1)"
date: 2025-12-20
categories: [Web Security, Theory, Clickjacking]
toc: true
---

# Clickjacking: Understanding the Mechanics

Clickjacking, formally known as **UI Redressing**, is an interface-based attack where a user is tricked into clicking on actionable content on a hidden website while believing they are clicking on something else entirely.

Unlike XSS or CSRF which target data or requests, Clickjacking targets the user's **physical interaction** with the application.

---

## 🔸 1. What is Clickjacking?

Clickjacking occurs when an attacker uses transparent or opaque layers (iframes) to trick a user into clicking on a button or link on another page when they intended to click on the top level page.

**The Analogy:**
Imagine an ATM machine. An attacker places a fake, transparent keypad over the real one. When you type your PIN, you think you are pressing the real buttons, but you are actually pressing the attacker's invisible buttons first (or simultaneously).


---

## ⚙️ 2. The Mechanics: The Invisible Iframe

The core of a Clickjacking attack relies on the HTML `<iframe>` element and CSS styling.

### The Attack Architecture
1.  **The Decoy (Attacker's Site):** The attacker creates a page with an enticing button (e.g., "Win a Prize!", "Play Video").
2.  **The Victim (Target Site):** The attacker embeds the target website (e.g., your bank) inside an `<iframe>`.
3.  **The Invisibility Cloak:** The attacker sets the iframe's CSS `opacity` to `0` (or close to 0). This makes the bank website invisible to the human eye, but it is still fully functional and interactive in the browser.
4.  **The Overlay (Z-Index):** The attacker positions the invisible iframe *on top* of the decoy button using CSS positioning (`absolute`) and `z-index`.

**The Result:**
When the user clicks the "Win a Prize" button, their mouse click actually passes through to the invisible iframe sitting on top. They unknowingly click "Transfer Money" or "Delete Account" on the target site.

---

## 🧨 3. Attack Variations

Clickjacking isn't just about single clicks. It has evolved into sophisticated scenarios.

### 1️⃣ Classic Clickjacking
A single click performs a state-changing action.
**Target:** "Delete Account" button.
 **Setup:** One invisible iframe overlay.

### 2️⃣ Multistep Clickjacking
Some actions require confirmation (e.g., "Delete" -> "Are you sure? Yes/No").
**Target:** Multi-stage workflows.
**Setup:** The attacker creates multiple "zones" on their page. The user is tricked into clicking "Play Level 1" (which clicks Delete), then "Play Level 2" (which clicks Yes).

### 3️⃣ Clickjacking + DOM XSS
If a site has a DOM XSS vulnerability that requires a click to trigger (e.g., a button that executes input), Clickjacking can be used to weaponize it.
**Scenario:** A search page reflects input but only executes script when "Search" is clicked.
**Attack:** Pre-fill the search with XSS via URL, then Clickjack the "Search" button.

### 4️⃣ URL Pre-filling
Forms (like "Update Email") are harder to clickjack because the user has to *type*.
**Attack:** The attacker looks for URL parameters that pre-fill the form (e.g., `?email=hacker@evil.com`).
**Execution:** The iframe loads with the hacker's data already filled in. The victim just needs to click "Update".

---

## 🛡️ 4. Defenses: The Anti-Framing Shield

Defending against Clickjacking requires preventing your site from being framed by unauthorized domains.

### A. X-Frame-Options (XFO)
This is the older, but widely supported HTTP header.
**`DENY`:** The page cannot be displayed in a frame, regardless of the site attempting to do so.
**`SAMEORIGIN`:** The page can only be displayed in a frame on the same origin as the page itself.
**`ALLOW-FROM uri`:** (Deprecated) Allows specific origins.

### B. Content-Security-Policy (CSP): frame-ancestors
This is the modern, more flexible standard. It obsoletes X-Frame-Options in supporting browsers.
**`frame-ancestors 'none'`:** Equivalent to XFO `DENY`.
**`frame-ancestors 'self'`:** Equivalent to XFO `SAMEORIGIN`.
**`frame-ancestors https://trusted.com`:** Whitelists specific domains.

### C. Frame Busting Scripts (Legacy)
Before headers were standardized, developers used JavaScript to stop framing.
```
if (top != self) { top.location = self.location; }
```

**Weakness:** These can often be bypassed using the HTML5 `sandbox` attribute on the iframe.
 

---

## 🛠️ 5. Bypassing Defenses

### 1️⃣ Bypassing Frame Busters (HTML5 Sandbox)

Attackers can neutralize frame-busting scripts by using the `sandbox` attribute on their iframe.

**Technique:** `<iframe sandbox="allow-forms">`
    
**Result:** The victim site loads, but its JavaScript execution is restricted. It cannot check `top.location` or navigate the top window, so the "buster" fails, but the form submission (the vulnerability) still works.
    

### 2️⃣ SameSite Cookie Confusion

Standard CSRF tokens _do not_ stop Clickjacking. Why?

**Reason:** The request originates from the user's real session inside the iframe. The browser sends the correct cookies and the correct CSRF token because the user _is_ legitimately interacting with the site (visually deceived, but technically legitimate interaction).
    
**Note:** `SameSite=Strict` cookies _can_ prevent Clickjacking if the action requires the cookie, because the iframe request is technically cross-site (from the top window's perspective). `Lax` cookies are usually sent in top-level navigations but behavior in iframes varies by browser and operation.
    

---

## ❓ 6. Interview Corner: Common FAQs (Pentest & AppSec)

If you are interviewing for a security role, expect these questions.

### Q1: Does a CSRF token prevent Clickjacking?

Answer:

No. CSRF tokens defend against forged requests (where the attacker creates a fake request). In Clickjacking, the user is interacting with the real application inside the iframe. The state (cookies, CSRF tokens) is valid because the session is legitimate. The issue is the user's intent is hijacked, not their session data.

### Q2: What is the difference between `X-Frame-Options` and `CSP: frame-ancestors`?

**Answer:**

**`X-Frame-Options` (XFO):** Older, header-based. Only supports `DENY` or `SAMEORIGIN` reliably. Cannot whitelist multiple specific domains easily.
    
**`frame-ancestors`:** Part of CSP. Newer, more flexible. Can accept multiple domains (`https://site-a.com https://site-b.com`). It takes precedence over XFO in modern browsers if both are present.
    

### Q3: How do you check for Clickjacking manually?

Answer:

I would create a simple HTML file on my local machine:

```
<iframe src="[https://target.com](https://target.com)" width="500" height="500"></iframe>
```

If the target site loads successfully inside this frame, it is potentially vulnerable. I would then check the response headers for `X-Frame-Options` or `Content-Security-Policy`.

### Q4: Can you Clickjack a login page? Why would you?

Answer:

Yes. This is often used for Login Redressing to capture credentials (though harder than standard phishing) or to trick users into enabling permissions (like OAuth authorizations). Forcing a user to click "Authorize" on a hidden OAuth prompt is a classic Clickjacking scenario.

### Q5: What is "Cursorjacking"?

Answer:

A variant of Clickjacking where the attacker hides the real mouse cursor and displays a fake cursor offset from the real one. The user moves the "fake" cursor to a safe button, but the "real" (invisible) cursor is hovering over a dangerous button.

### Q6: Why is `SameSite=Strict` effective against Clickjacking?

Answer:

If the target site sets session cookies with SameSite=Strict, the browser will not send these cookies when the site is loaded inside an iframe on a cross-origin (attacker) site. The iframe will load as if the user is logged out, rendering the attack useless for authenticated actions.

### Q7: Explain the "Frame Busting" bypass using `sandbox`.

Answer:

Frame busting scripts rely on JavaScript (e.g., top.location = self.location) to detect if they are framed. An attacker can use <iframe sandbox="allow-forms"> to enable form submission but disable JavaScript execution or top-level navigation. This effectively "silences" the protection script while keeping the vulnerability open.

### Q8: Is Clickjacking possible on mobile devices?

Answer:

Yes, often called "Tapjacking." It works similarly but exploits touch events. Mobile browsers and OSs have implemented overlays detections to mitigate this, but web-based tapjacking is still a valid threat vector.

### Q9: What is "Likejacking"?

Answer:

A specific form of Clickjacking targeting social media "Like" or "Share" buttons. The attacker places an invisible "Like" button over a video player or other content, tricking users into promoting the attacker's content socially.

### Q10: If a site has `X-Frame-Options: SAMEORIGIN`, is it completely safe?

Answer:

Generally yes, for external attackers. However, if the site has a Cross-Site Scripting (XSS) vulnerability, the attacker can use the XSS to inject an iframe from the same origin, bypassing the XFO restriction.

---

## 🎭 7. Scenario-Based Questions

### 🎭 Scenario 1: The "Unframed" Defense

Context: A developer says: "I don't need X-Frame-Options because my site uses window.open() to launch critical actions in a new popup window."

The Question: Is this safe from Clickjacking?

The "Hired" Answer:

"It reduces the risk for that specific action, but it's bad UX and not a comprehensive fix.

1. **Popup Blockers:** Browsers might block the popup.
    
2. **Other Actions:** Are _all_ actions in popups? What about 'Update Profile' or 'Change Settings'? If any actionable page remains framable, it is vulnerable.
    
3. **The Fix:** Relying on window management is fragile. Implementing `frame-ancestors` is the correct, architectural fix."
    

### 🎭 Scenario 2: The "Widget" Problem

Context: Your company provides a "Chat Widget" that other websites embed via iframe. You need to allow embedding but prevent Clickjacking on the admin settings of the widget.

The Question: How do you configure CSP?

The "Hired" Answer:

"This requires a nuanced policy.

1. **Public Widget:** The endpoint serving the public chat widget needs `Content-Security-Policy: frame-ancestors *` (or a whitelist of customer domains) to allow embedding.
    
2. **Admin/Settings:** The endpoints for configuring the widget (which contain sensitive state) must serve `Content-Security-Policy: frame-ancestors 'self'` or `'none'` to prevent anyone from framing the admin controls."
    

### 🎭 Scenario 3: The Legacy App

Context: You are testing an old app. It has a frame-busting script: if(top!=self) top.location=self.location.

The Question: How do you demonstrate this is insufficient?

The "Hired" Answer:

"I would build a Proof of Concept (PoC) using the HTML5 sandbox attribute.

1. I will create an iframe: `<iframe src='target_url' sandbox='allow-forms'></iframe>`.
    
2. The `sandbox` attribute blocks the script from navigating the top window (`top.location`), so the app stays trapped in the frame.
    
3. Since `allow-forms` is present, I can still clickjack any buttons or forms on that page."
    

### 🎭 Scenario 4: The "Drag and Drop"

Context: The application has a file upload feature that supports Drag and Drop.

The Question: Can Clickjacking exploit this?

The "Hired" Answer:

"Yes. An attacker can trick a user into dragging a file (or data) from their desktop or another window and 'dropping' it onto the invisible iframe. This can be used to upload malicious files or transfer sensitive text data cross-origin if the drop target is not properly secured or visual feedback is spoofed."

### 🎭 Scenario 5: Double Defense?

Context: A site sends both X-Frame-Options: DENY and Content-Security-Policy: frame-ancestors 'self'.

The Question: Which one wins? Why use both?

The "Hired" Answer:

"In modern browsers, CSP's frame-ancestors takes precedence. However, we use both for Defense in Depth and Backward Compatibility. Older browsers (like Internet Explorer) do not support CSP frame-ancestors but do respect X-Frame-Options. Using both ensures coverage across all user agents."

---

## 🛑 Summary of Part 1

1. **Concept:** Tricking users into clicking invisible UI elements.
    
2. **Impact:** Unauthorized actions (Delete, Transfer, Like) performed with user consent.
    
3. **Attack:** Invisible Iframes, Z-Index layering, and Sandbox bypasses.
    
4. **Defense:** `Content-Security-Policy: frame-ancestors` and `X-Frame-Options`.
