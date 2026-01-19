---
layout: mission
title: "DOM-Based Vulnerabilities: The Theory & Mechanics (Part 1)"
date: 2026-01-28
categories: [Web Security, Theory, Client-Side]
toc: true
---

# DOM-Based Vulnerabilities: Understanding the Mechanics

Before we can exploit client-side logic, we must understand the fundamental architecture of the Document Object Model (DOM). DOM-based vulnerabilities are not about server-side flaws; they occur entirely within the user's browser when **JavaScript handles user-controllable data insecurely**.

---

## 🔸 1. What is the DOM?

The **Document Object Model (DOM)** is the browser's internal representation of a webpage as a tree-like structure. When you make a request to a web application, the HTML in the response is loaded as the DOM in the browser. In essence, the DOM is the programmatic view of the web application that the user sees.

JavaScript can dynamically manipulate the DOM to update content, styles, and behavior.

### The Evolution: SPAs vs. Traditional Apps

1. **Traditional Models:** Conventional web applications were built where the response to each web request would refresh the entire DOM.
2. *Single Page Applications (SPAs):** With the rise of modern frontend frameworks, SPAs were born. They are loaded only once when the user visits the website for the first time, and all code is loaded in the DOM. Instead of reloading the page, the DOM is automatically updated via JavaScript.

> [!NOTE]
> ✅ **DOM manipulation itself is not dangerous.**
> ⚠️ Vulnerabilities arise only when data from an **attacker-controlled source** is passed to a **dangerous function** (sink) without proper sanitization.

---

## 🧨 2. The Vulnerability: Taint Flow Architecture

To understand DOM attacks, you must visualize the flow of data (Taint Flow). A DOM-based vulnerability exists when there is a connected path from a **Source** to a **Sink**.

### 🟡 The Source (Input)
A **Source** is a JavaScript property or API that accepts user-supplied input. If an attacker can manipulate it, it is a potential source.

**Common Sources:**
1. *Navigation Properties:** `location.search` (Query String), `location.hash` (Fragment), `location.href`, `document.documentURI`, `document.URL`.

2. *Storage/History:** `document.cookie`, `localStorage`, `sessionStorage`, `window.name`, `history.pushState()`.
* **Communication:** `document.referrer`, `postMessage()`.
* **Files:** `IndexedDB`, `FileReader()`.

### 🔴 The Sink (Execution)
A **Sink** is a function or DOM object that can cause harmful behavior if it receives malicious input.

| Vulnerability Type | 🔴 Dangerous Sink Examples |
| :--- | :--- |
| **DOM XSS** | `document.write()`, `innerHTML` |
| **Open Redirection** | `window.location`, `location.href`, `location.assign()` |
| **Cookie Manipulation** | `document.cookie` |
| **JavaScript Injection** | `eval()`, `setTimeout()`, `setInterval()`, `Function()` |
| **WebSocket Poisoning** | `WebSocket()` |
| **Link Manipulation** | `element.setAttribute()`, `element.href` |
| **Data Injection** | `JSON.parse()`, `localStorage.setItem()` |

---

## 🧠 3. Deep Dive: DOM-Based Cross-Site Scripting (DOM XSS)

DOM XSS occurs when an application contains some client-side JavaScript that processes data from an untrusted source in an unsafe way, usually by writing the data back to the DOM.

Unlike Reflected XSS, where the server reflects the payload, **DOM XSS happens entirely on the client side**.

### A. The Classic Sinks: `document.write` & `innerHTML`

#### 1️⃣ `document.write()`
This sink writes data directly into the HTML stream.
***Payload:** `document.write('<script>alert(1)</script>');`
**Context:** You may need to close tags or adjust the payload to fit surrounding content.

#### 2️⃣ `innerHTML`
This sink sets the HTML content of an element.
**The Trap:** Modern browsers do **not** execute `<script>` tags inserted via `innerHTML` due to security specifications.
**The Bypass:** You must use an element that executes code via an event handler.

**Payload:** `<img src=x onerror=alert(1)>`
     **Why it works:** The browser tries to load the image source "x", fails (because it's not a URL), triggers the `onerror` event, and executes the JS.

### B. Framework-Specific Vectors

#### 🔹 jQuery `.attr()` (Attribute Injection)
The `$` function in jQuery is shorthand for `jQuery()`. The `.attr()` method is used to dynamically change HTML attributes.

**Vulnerable Code:**
    ```javascript
    $('#link').attr('href', new URLSearchParams(location.search).get('returnUrl'));
    ```
**Attack:** If the URL parameter `returnUrl` is `javascript:alert(1)`, the `href` attribute becomes a JavaScript execution sink.

#### 🔹 AngularJS (Expression Injection)
AngularJS scans HTML nodes containing the `ng-app` directive. It allows the execution of JavaScript expressions within double curly braces `{{ }}`.

**The Technique:** You can access the `constructor` property of a function to get the global Function constructor (equivalent to `new Function()`).

**Payload:** `<div ng-app>{{constructor.constructor('alert(1)')()}}</div>`
     **Breakdown:** This effectively runs `(new Function('alert(1)'))()`, immediately executing the alert.

#### 🔹 jQuery `hashchange` Event
The `hashchange` event fires whenever the fragment identifier (the part after `#`) changes. SPAs often use this to change views without reloading.
**Vulnerability:** If the application reads `location.hash` and inserts it into the DOM without sanitization, it is exploitable.
**Advanced Exploit:** You can use an `iframe` to exploit this.
```html
<iframe src="[https://vulnerable.com](https://vulnerable.com)#" onload="this.src+='<img src=x onerror=alert(1)>'"></iframe>
```
     **Mechanics:** The iframe loads the page. The `onload` event appends the malicious hash. The `hashchange` event fires in the victim page, processing the payload.

---

## 🧩 4. DOM-Based Open Redirection

This vulnerability occurs when a script writes attacker-controlled data into a navigation sink like `window.location` or `location.href` **without proper validation**.

### The Mechanics
**Sinks:** `location`, `location.href`, `location.assign()`, `location.replace()`, `window.open()`.
**Risk:** Phishing attacks where users trust the domain but are silently redirected to a malicious site.

### Regex Logic Flaws
Developers often try to validate URLs using Regular Expressions (Regex) but fail to cover all edge cases.

**Vulnerable Code:**
```javascript
let url = /https?:\/\/.+/.exec(location.hash);
if (url) { location = url[0]; }
```

**Why it fails:** The `exec()` function extracts the URL without validating that it belongs to a trusted domain.
    
**Payload:** `#https://evil.com` forces the browser to navigate to the attacker's site.
    
**Protocol Injection:** If the regex is loose, an attacker might inject `javascript:alert(1)`.
  

---

## 🍪 5. DOM-Based Cookie Manipulation

This occurs when attacker-controllable data (like `location.hash`) is written into `document.cookie` without sanitization.

**Vulnerable Code:**

JavaScript

```
document.cookie = 'cookieName=' + location.hash.slice(1);
```

**Impact:**

1. **Session Fixation:** The attacker sets a known session ID.
    
2. **Role Escalation:** The attacker injects `role=admin`.
    
3. **Exploit Chain:** If the application later reads this cookie unsafely, it can lead to XSS.
    

Payload Example:

Visiting https://site.com/#admin sets the cookie to cookieName=admin.

---

## 💉 6. DOM-Based JavaScript Injection

This is the most critical variant, where input is passed to a sink that executes strings as code.

### Execution Sinks

`eval(str)`: Executes string as JS code (Most Dangerous).
    
`Function(str)`: Creates a new function from the string.
    
`setTimeout(str)` / `setInterval(str)`: Executes code after a delay.
    

**Vulnerable Code:**

JavaScript

```
eval(location.hash.slice(1));
```

**Payload:** `https://victim.com/#alert(1)` executes the alert immediately.

---

## 🌐 7. Advanced DOM Vectors

### A. `document.domain` Manipulation

The `document.domain` property enforces the **Same-Origin Policy (SOP)**. Two pages can access each other's data only if they share the same origin.

**Attack:** If an attacker can force a page to set `document.domain` to a parent value (e.g., `evil.com`), they can bypass SOP restrictions and access data between origins.
    
**Code:** `document.domain = location.hash.slice(1);`.
    

### B. WebSocket-URL Poisoning

WebSockets enable persistent communication. If the connection URL is built using user input, it can be hijacked.

**Vulnerable Sink:** `new WebSocket(location.hash.slice(1));`.
    
**Impact:**
    
    1. **Data Theft:** Intercept chat messages or tokens.
        
    2. **Logic Subversion:** Inject malicious payloads back into the client application.
        

### C. Link Manipulation

This occurs when a script dynamically modifies `href` or `src` attributes of links or forms using user input.

**Vulnerable Code:** `document.getElementById("download-link").href = location.hash;`.
    
**Payload:** `../../../../evil.com/malware.exe` changes the download link to an external malware source.
  

---

## 📨 8. DOM-Based Web Message Manipulation (`postMessage`)

The `window.postMessage()` API allows windows (e.g., iframes and their parents) to communicate across different origins, effectively drilling a hole in the Same-Origin Policy.

### The Core Vulnerability: Blind Trust

The flaw arises when a script adds an event listener for "message" events but **fails to verify the origin** of the sender.

**Vulnerable Receiver:**

JavaScript

```
window.addEventListener('message', function(e) {
   eval(e.data); // ❌ Unsafe Sink + No Origin Check
});
```

### Exploitation Mechanics

1. **The Setup:** Attacker hosts a page with an `iframe` pointing to the victim site.
    
2. **The Trigger:** Use `postMessage` to send a malicious payload.
    
    JavaScript
    
    ```
    iframe.contentWindow.postMessage('alert(1)', '*'); // '*' means send to any origin
    ```
    
3. **The Execution:** The victim's browser receives the message and executes the `eval()` sink.
    

### Broken Origin Verification Patterns

Developers often attempt validation using weak string methods:

`indexOf('trusted.com')`: Bypassed by `attacker-trusted.com`.
    
`startsWith('https://trusted')`: Bypassed by `https://trusted.attacker.com`.
    
`endsWith('trusted.com')`: Bypassed by `malicioustrusted.com`.
    

### JSON Sinks

Even if `eval()` isn't used, `JSON.parse()` can be a vector if the parsed data is used unsafely.

**Vulnerable Logic:**

    ```
    var data = JSON.parse(e.data);
    document.body.innerHTML = data.payload;
    ```
    
**Payload:** `postMessage('{"payload":"<img src=x onerror=alert(1)>"}', '*')`.
    

---

## 🛠️ 9. Testing & Discovery Strategy

### 1️⃣ Manual Analysis (DevTools)

1. **Trace the Source:** Inject a unique string (e.g., `test123`) into the URL or hash.
    
2. **Find the Sink:** Open Chrome DevTools (Elements tab) and search (`Ctrl+F`) for your string to see where it lands in the DOM.
    
3. **JavaScript Debugging:** Use `Ctrl+Shift+F` to search for sources in the loaded JS files. Set breakpoints to trace variables at runtime.
    

### 2️⃣ Automated Tools

**Burp Suite DOM Invader:** A specialized tool in the Burp browser that highlights sources and sinks and tracks taint flow automatically.
    

---

## ❓ 10. Interview Corner: 10 Critical Questions

### Basic / Intermediate

Q1: What defines a DOM-based vulnerability?

Answer: A vulnerability where data from an attacker-controlled Source (like location.hash) propagates to a dangerous Sink (like innerHTML) within the browser's JavaScript, without proper sanitization.

Q2: Why is innerHTML dangerous, and how does it differ from innerText?

Answer: innerHTML parses content as HTML, meaning tags like <img onerror...> will execute code. innerText treats content as plain text, neutralizing HTML tags.

Q3: Explain why <script>alert(1)</script> doesn't work in innerHTML.

Answer: HTML5 security specifications prevent scripts inserted via innerHTML from executing. Attackers must use event handlers (like onerror or onload) on other elements to achieve execution.

Q4: What is the risk of using indexOf to validate a URL origin?

Answer: indexOf only checks if a substring exists anywhere in the string. http://trusted.com matches http://attacker-trusted.com or javascript://trusted.com, allowing bypasses.

Q5: How does DOM XSS differ from Reflected XSS?

Answer: Reflected XSS relies on the server returning the payload in the response body. DOM XSS executes entirely client-side; the server response may be clean, but the browser's JS processes the malicious input.

### Advanced

Q6: How can postMessage lead to DOM XSS?

Answer: If an application listens for messages (window.addEventListener) without validating event.origin and passes event.data to a sink like eval() or innerHTML, any site can execute code in the victim's context.

Q7: What is WebSocket-URL Poisoning?

Answer: A vulnerability where the WebSocket connection URL is built using unvalidated user input. Attackers can redirect the connection to their own server to intercept tokens or inject malicious data.

Q8: Explain "Source" vs "Sink" in Taint Analysis.

Answer: A Source is where untrusted data enters the application (e.g., location.search). A Sink is where that data is executed or rendered (e.g., document.write). The path between them is the Taint Flow.

Q9: What is document.domain manipulation used for?

Answer: It is used to relax the Same-Origin Policy. If an attacker can inject values into document.domain, they can force a victim page to share access with a malicious page on a parent domain.

Q10: Can JSON.parse() be a vulnerability?

Answer: JSON.parse() itself is safe (it just parses text), but it often acts as a gateway. If the resulting object is used to set iframe.src or innerHTML, it becomes part of the exploit chain.

---

## 🎭 11. Scenario-Based Questions ("Bar Raiser")

### 🎭 Scenario 1: The "Empty" Page

Interviewer: "You are testing a website. You view the page source, but it's just a <body> with a single <div id='app'>. However, the page is rich with content. How do you find XSS here?"

Answer:

"This indicates a Single Page Application (SPA). 'View Source' is useless.

1. **Shift Strategy:** Open **DevTools** > **Sources** tab to inspect the loaded JavaScript files (e.g., `app.js`).
    
2. **Hunt for Sinks:** Search JS bundles for framework-specific sinks like `v-html` (Vue), `dangerouslySetInnerHTML` (React), or `bypassSecurityTrustHtml` (Angular).
    
3. **Trace:** Use breakpoints to see if user input reaches these sinks".
    

### 🎭 Scenario 2: The "Secure" Angular App

Interviewer: "A developer claims their Angular app is secure because Angular sanitizes input. You found a reflection inside {{ }}. How do you exploit it?"

Answer:

"I would attempt Template Injection. Angular expressions allow access to the constructor. I would use a payload like {{constructor.constructor('alert(1)')()}}. This creates a new function from a string and executes it, effectively bypassing Angular's standard sanitization".

### 🎭 Scenario 3: The `postMessage` JSON

Interviewer: "You see code that parses JSON from a postMessage and uses a URL field to load an iframe. The developer says JSON.parse makes it safe. Is it?"

Answer:

"No. JSON.parse only checks syntax, not safety. If the code uses iframe.src = data.url, I can inject javascript:alert(1). The vulnerability is the missing Origin Validation (event.origin) and the unsafe sink, not the parsing itself".

### 🎭 Scenario 4: The Internal Redirect

Interviewer: "Code uses location.href = params.url. The WAF blocks http: and https:. Is it exploitable?"

Answer:

"Yes.

1. **Protocol Bypass:** I can use `javascript:alert(1)` to execute code.
    
2. **Protocol-Relative:** I can use `//evil.com`, which browsers interpret as a valid URL but might bypass simple 'http' keyword filters.
    
3. **Encoding:** URL encoding or double encoding might bypass the WAF while the browser still decodes and executes it".
    

### 🎭 Scenario 5: WebSocket Chat ID

Interviewer: "A chat app connects via new WebSocket('wss://api.com/' + user_id). user_id comes from the URL. What is the risk?"

Answer:

"This is WebSocket Poisoning.

1. **Redirection:** I could inject `@attacker.com` to treat the first part as credentials and redirect the connection to my server.
    
2. **Hijacking:** Once connected to my server, I can capture the initialization handshake (often containing auth tokens) or inject fake chat messages back to the user".
    

---

## 🛑 Summary of Part 1

1. **Concept:** The DOM is the browser's dynamic view. Vulnerabilities happen when JS handles data insecurely.
    
2. **Mechanism:** It requires a **Source** (bad input) and a **Sink** (dangerous execution).
    
3. **Attack:** We manipulate sources (URL, Hash, Cookies) to force sinks (`innerHTML`, `location`, `eval`) to execute our malicious intent.
