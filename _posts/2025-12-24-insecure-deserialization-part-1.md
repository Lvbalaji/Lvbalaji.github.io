---
layout: mission
title: "Insecure Deserialization: The Deep Dive (Part 1)"
date: 2025-12-24
categories: [Web Security, Theory, Deserialization]
toc: true
---

# Insecure Deserialization:

Insecure Deserialization is a vulnerability that occurs when an application uses untrusted data to reconstruct an object's state. Unlike SQL Injection or XSS, which manipulate queries or scripts, deserialization attacks manipulate the **logic flow** of the application by abusing the very objects it uses to run.

It is often called the "Holy Grail" because a successful exploit frequently leads to **Remote Code Execution (RCE)** or full system compromise.

---

## 🔸 1. Core Concepts: Serialization vs. Deserialization

To understand the attack, you must understand the mechanism.

**Serialization** is the process of converting a complex data structure (like an object in memory) into a linear format (byte stream, JSON, string) for:
1.  **Storage:** Saving a user's session to a file or database.
2.  **Transmission:** Sending data between a web server and a client (API calls).

**Deserialization** is the reverse: taking that stream and reconstructing the object in memory, restoring its class, attributes, and state.

### The Terminology Map
Different languages use different terms for this process:
**PHP / Java:** Serialization
**Python:** Pickling
**Ruby:** Marshalling.

---

## ⚙️ 2. Mechanics by Language

Identifying serialized data is the first step in detection.

### 🐘 PHP Serialization
PHP uses a human-readable string format.
**Signature:** `O:` (Object), `a:` (Array), `s:` (String).
**Example:** `O:4:"User":2:{s:4:"name";s:5:"admin";s:7:"isAdmin";b:1;}`
    1. `O:4:"User"`: An object of class "User" (4 bytes).
    2. `s:7:"isAdmin";b:1;`: The property `isAdmin` is set to boolean `true`.

### ☕ Java Serialization
Java uses a binary format.
**Signature:** Hex `AC ED 00 05`.
**Base64:** Often starts with `rO0...`.
**Mechanism:** Uses `ObjectInputStream.readObject()` to restore the object.

### 🐍 Python (Pickle)
Python uses the `pickle` library.
**Signature:** Specific opcodes like `(`, `.`, `cos`, `system`.
**Danger:** Python's pickle is a virtual machine. It allows defining a callable (function) that runs *during* unpickling, making RCE trivial compared to other languages.

---

## 🧨 3. Vulnerability Class 1: Data Tampering & Type Juggling

Even without RCE, attackers can manipulate the *state* of an object to bypass logic.

### A. Attribute Manipulation (Privilege Escalation)
If a user object is serialized in a cookie, an attacker can modify the attributes.

**Original:** `O:4:"User":1:{s:8:"username";s:3:"bob";s:7:"isAdmin";b:0;}`
**Attack:** The attacker changes `b:0` to `b:1` (True).
**Result:** Upon deserialization, the application instantiates a User object with Admin privileges.

### B. PHP Type Juggling (Authentication Bypass)
PHP allows **loose comparison** (`==`).

**The Quirk:** When comparing an integer `0` to a string: `0 == "AnyString"` evaluates to `TRUE` (in PHP < 8).
**The Scenario:** The code checks: `if ($user->password == $stored_password)`.
**The Attack:** The attacker modifies the serialized object to set the password type to **Integer** `0`.
     Payload: `...s:8:"password";i:0;...`
**Result:** The comparison becomes `0 == "SuperSecretPassword"`, which is True. Authentication bypassed.

---

## 🧨 4. Vulnerability Class 2: Magic Methods & Gadgets

This is where RCE happens. **Magic Methods** are special functions that execute automatically during an object's lifecycle. Attackers abuse these to trigger dangerous code.

### Common Magic Methods
**`__wakeup()` (PHP):** Called immediately when an object is deserialized (often used to re-establish DB connections).
**`__destruct()` (PHP):** Called when an object is destroyed (e.g., at the end of a script).
**`readObject()` (Java):** Called during deserialization to read the stream.

### The "Gadget" Concept
A **Gadget** is a snippet of valid code (a class or method) already present in the application that performs a specific action. By chaining these gadgets, attackers create a **POP Chain** (Property Oriented Programming) to achieve a goal (like RCE).

### Example: Arbitrary File Deletion
Imagine a class designed to clean up temporary files.
```
class FileCleaner {
    public $filename;
    function __destruct() {
        unlink($this->filename); // Deletes the file
    }
}
```

**The Logic:** When the object is destroyed, it deletes the file specified in `$filename`.
    
**The Attack:** An attacker serializes a `FileCleaner` object but sets `$filename` to `/var/www/html/index.php`.
    
**The Execution:**
    
    1. Attacker sends the payload.
        
    2. Server calls `unserialize()`. The object is created in memory.
        
    3. Script finishes. PHP destroys the object.
        
    4. `__destruct()` runs automatically.
        
    5. `unlink("/var/www/html/index.php")` is executed. The site is deleted.

---

## 🧨 5. Vulnerability Class 3: Object Injection (Arbitrary Classes)

If the application does not validate _which_ class is being deserialized, an attacker can inject an object of **any class** available in the codebase.

Why is this dangerous?

Even if the application expects a User object, if the attacker sends a serialized FileCleaner object (from a library), PHP will happily instantiate a FileCleaner. This allows the attacker to access methods (like __destruct) that were never meant to be reachable from the user input context.

---

## 🛡️ 6. Detection & Remediation

### Detection

**Whitebox:** grep for `unserialize`, `readObject`, `pickle.load`, `Marshal.load`.
    
**Blackbox:** Look for Base64 strings starting with `rO0` (Java) or strings starting with `O:` (PHP). Attempt to alter logical attributes (e.g., changing IDs or booleans) and observe errors.
    

### Defense Strategies

1. **Stop Deserializing:** Use JSON or XML. They transfer data, not objects.
    
2. **Integrity Checks:** If you must use serialization, sign the data with an HMAC signature. Verify the signature _before_ deserialization.
    
3. **Strict Type Checking:**
    
    **Java:** Use `ObjectInputFilter` to whitelist allowed classes.
        
    **PHP:** Use `allowed_classes` option in `unserialize()`.   

---

## ❓ 7. Interview Corner: 10 Advanced Questions

### Q1: Explain the concept of a "Gadget Chain".

Answer:

A Gadget Chain is a sequence of code snippets (classes/methods) already present in the application that can be chained together to perform a malicious action. An attacker starts with a "Trigger" gadget (like __destruct in PHP or readObject in Java) and manipulates the object's properties to pass data to subsequent gadgets, eventually reaching a "Sink" gadget (like exec or unlink).

### Q2: What is the difference between `__wakeup()` and `__destruct()`?

**Answer:**

`__wakeup()` is executed **immediately** when `unserialize()` is called. It is typically used to re-establish connections (database/socket) that were lost during serialization.
    
`__destruct()` is executed when the object is no longer referenced or the script terminates. It is used for cleanup (deleting temp files, closing logs). Both are prime entry points for POP chains.
    

### Q3: How does PHP Type Juggling facilitate Authentication Bypass in deserialization?

Answer:

If an application compares a deserialized property (like a token) using the loose operator (==), PHP attempts to convert types. An integer 0 compares as True against any string that doesn't start with a number (e.g., 0 == "secret_token"). By modifying the serialized object to change the token type to an integer 0, attackers bypass the check.

### Q4: Why is `ysoserial` relevant to Java Deserialization?

Answer:

ysoserial is a tool that generates serialized payloads for common Java libraries (like Apache Commons Collections, Spring, Hibernate). These libraries contain known gadget chains that allow for RCE. If a target application has one of these libraries on its classpath and deserializes untrusted data, ysoserial can generate the exploit payload.

### Q5: What signatures would you look for to identify PHP serialized data?

Answer:

I would look for strings formatted like O:4:"User":2:{...} (Object) or a:2:{...} (Array). The pattern s:N:"value" (String of length N) is also a strong indicator. Often, these are URL-encoded or Base64 encoded in cookies.

### Q6: Can you exploit insecure deserialization in a black-box test?

Answer:

Yes. I would look for cookies or parameters that appear to be Base64 encoded. Decoding them might reveal serialized signatures (rO0, O:). I would then try modifying attributes (e.g., isAdmin, role) or changing data types (Type Juggling) to see if it affects application logic. For RCE, I might blind-spray common gadget payloads (like CommonsCollections) and check for time delays or OOB interactions.

### Q7: What is "Object Injection"?

Answer:

It is the act of passing a serialized object of an unexpected class to the deserialization function. If the application doesn't verify the class type before instantiation, it creates an object of the attacker-chosen class. This allows the attacker to leverage the methods (gadgets) of that class.

### Q8: How does Python's `pickle` vulnerability work?

Answer:

Python's pickle serialization allows specifying a callable (function) and arguments that should be executed to reconstruct the object. Attackers use the __reduce__ method to tell the unpickler to execute os.system with a malicious command (e.g., whoami) immediately upon loading.

### Q9: Is it safe to deserialize data if I encrypt it?

Answer:

Encryption provides confidentiality, not integrity. An attacker could potentially manipulate the encrypted blob (bit-flipping) if the encryption mode is malleable (like CBC without MAC). The only safe way is to use an Integrity Check (like HMAC) to ensure the data hasn't been tampered with before attempting to decrypt and deserialize it.

### Q10: What is a "Magic Byte"?

Answer:

Magic bytes are specific file signatures used to identify file formats. In Java serialization, the hex stream always starts with AC ED 00 05. Identifying these bytes is the quickest way to detect binary serialized traffic.

---

## 🎭 8. Scenario-Based Questions

### 🎭 Scenario 1: The "Legacy" Cleanup

Context: You find a legacy PHP class TempFile with a __destruct method: unlink($this->path);.

The Question: You have no way to upload files. Can you still cause damage?

The "Hired" Answer:

"Yes. I can use this as an Arbitrary File Deletion gadget.

1. I verify if I can inject a serialized object (via cookie or param).
    
2. I construct a `TempFile` object and set the `$path` property to a critical system file, like the application's configuration file (`config.php`) or `.htaccess`.
    
3. When the request ends, `__destruct` runs and deletes the config file, causing a Denial of Service (DoS) or re-installing the app to take control."
    

### 🎭 Scenario 2: The "Secure" Comparison

Context: The developer says: "I use == for checking the API key in the deserialized object, but the key is a random alphanumeric string, so it's unguessable."

The Question: How do you exploit this?

The "Hired" Answer:

"The == operator is vulnerable to Type Juggling. I don't need to guess the string.

1. I capture the serialized object.
    
2. I change the API key property type from String (`s`) to Integer (`i`) and set the value to `0`.
    
3. PHP will evaluate `0 == "randomString"` as `True`.
    
4. I bypass the API key check entirely."
    

### 🎭 Scenario 3: The Java Stack Trace

Context: You send a malformed payload to a Java endpoint and get a stack trace mentioning org.apache.commons.collections.

The Question: What is your next move?

The "Hired" Answer:

"This confirms the presence of the Apache Commons Collections library, which is a known vector for RCE.

1. I would use `ysoserial` to generate a payload using the `CommonsCollections` gadget chain.
    
2. I would try a benign command first (like `nslookup` or `sleep`) to confirm execution blindly.
    
3. If confirmed, I would escalate to a reverse shell."
    

### 🎭 Scenario 4: Source Code Review

Context: You are reviewing PHP code. You see: $user_data = unserialize($_COOKIE['data']);.

The Question: What specific things do you search for in the rest of the code?

The "Hired" Answer:

"I am looking for Gadgets.

1. I will search for any class definition with **Magic Methods** (`__wakeup`, `__destruct`, `__toString`).
    
2. Inside those methods, I look for 'Sinks'—dangerous functions using object properties (e.g., `eval`, `include`, `unlink`, `system`).
    
3. If I find a class where a Magic Method passes a controllable property to a Sink, I have an exploit chain."
    

### 🎭 Scenario 5: The "Backup" File

Context: You found index.php~ (a backup file) which contains a class Logger that writes to a file defined in $this->logFile.

The Question: How can this lead to RCE?

The "Hired" Answer:

"I can use this for Arbitrary File Write.

1. I create a serialized `Logger` object.
    
2. I set `$this->logFile` to `shell.php`.
    
3. I set the content to be logged (if controllable) to `<?php system($_GET['c']); ?>`.
    
4. When the object is processed (via `__destruct` or another method), it writes my shell to the disk. I then access `shell.php` to execute commands."
    

---

## 🛑 Summary of Part 1

1. **Concept:** Injecting malicious objects into an application that blindly reconstructs them.
    
2. **Logic Flaws:** Attribute manipulation and Type Juggling (PHP).
    
3. **RCE Mechanism:** Chaining "Magic Methods" (`__wakeup`, `__destruct`) into Gadget Chains.
    
4. **Defense:** Do not deserialize untrusted data; use HMAC signatures.
