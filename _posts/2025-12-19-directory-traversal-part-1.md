---
layout: default
title: "Directory Traversal: The Theory & Mechanics (Part 1)"
date: 2025-12-19
categories: [Web Security, Theory, Directory Traversal]
toc: true
---

# Directory Traversal: Understanding the Mechanics

Directory Traversal (also known as Path Traversal) is a vulnerability that allows an attacker to read arbitrary files on the server that is running an application. This might include application source code, sensitive system files, or credentials.

It is one of the most straightforward yet potentially devastating vulnerabilities, exploiting the fundamental way file systems are structured.

---

## 🔸 1. What is Directory Traversal?

Directory traversal exploits insufficient security validation of user-supplied filenames. It allows an attacker to "traverse" (move) out of the intended folder and access files elsewhere on the file system.

**Scenario:**
Imagine a photo gallery app. It loads images using a URL like:
`https://example.com/loadImage?filename=cat.png`

The backend code (e.g., PHP) might look like this:
```
// VULNERABLE
$file = $_GET['filename'];
$path = "/var/www/images/" + $file;
echo file_get_contents($path);
```

If the user requests cat.png, the server reads /var/www/images/cat.png.

However, if the attacker requests ../../../etc/passwd, the server constructs the path:

/var/www/images/../../../etc/passwd

The `../` sequences (dot-dot-slash) tell the operating system to move **up** one directory. The final resolved path becomes simply `/etc/passwd`.

---

## ⚙️ 2. The Mechanics: Understanding Paths

To exploit this, you need to understand how file paths work in different operating systems.

### The "Dot-Dot-Slash" (`../`)

This is the universal "parent directory" alias.

1. `/var/www/html/` + `../` = `/var/www/`
    
2. `/var/www/` + `../` = `/var/`
    
3. `/var/` + `../` = `/` (The Root Directory)
    

### Absolute vs. Relative Paths

1. **Relative Path:** Starts from the current working directory (e.g., `images/cat.png`). This is usually what the application expects.
    
2. **Absolute Path:** Starts from the root (e.g., `/etc/passwd` on Linux or `C:\Windows\win.ini` on Windows). If an application allows absolute paths, you don't even need `../`. You can just ask for the file directly.
    

---

## 🧨 3. Common Targets (What to Steal)

Once you have traversal, what should you look for?

### 🐧 Linux/Unix

1. `/etc/passwd`: List of all users on the system.
    
2.  `/etc/shadow`: Password hashes (requires root privileges, usually not accessible).
    
3.  `/etc/hosts`: Network configuration.
    
4.  `~/.ssh/id_rsa`: Private SSH keys (if you can guess the user's home directory).
    
5.  `/var/log/apache2/access.log`: Web server logs (useful for Log Poisoning).
    

### 🪟 Windows

1. `C:\Windows\win.ini`: Basic system file (good for proof of concept).
    
2. `C:\boot.ini`: Boot configuration.
    
3. `C:\Windows\System32\drivers\etc\hosts`: DNS hosts file.
    

---

## 🛠️ 4. Attack Mechanics: Bypassing Filters

Developers use various methods to stop traversal. Here is how we break them.

### A. Absolute Path Bypass

The developer removes `../` but forgets to block absolute paths.

1. **Payload:** `filename=/etc/passwd`
    
2. **Why it works:** Many file system functions treat an input starting with `/` as an absolute path, ignoring any directory prepended by the developer.
    

### B. Nested Traversal (Non-Recursive Stripping)

The developer removes `../` once.

1. **Payload:** `....//` or `..././`
    
2. **Why it works:** The filter removes the inner `../`, and the remaining characters collapse to form a valid `../` again.
    

### C. Encoding Bypasses

The developer checks for dots and slashes before decoding the input.

1. **URL Encoding:** `..%2f`
    
2. **Double URL Encoding:** `..%252f` (The `%` becomes `%25`)
    
3. **Why it works:** The web server decodes `%25` to `%` first. The WAF/Filter sees `%2f` (safe). The application decodes `%2f` to `/` (unsafe).
    

### D. Null Byte Injection (Legacy)

The developer checks if the file ends with `.png`.

1. **Payload:** `../../../etc/passwd%00.png`
    
2. **Why it works:** In C/C++ based systems (like older PHP/Java), the string reading stops at the Null Byte (`%00`). The validation sees `.png` (safe), but the file system sees `passwd` (exploit).
    

### E. Required Folder Prefix

The developer checks if the path _starts_ with `/var/www/images`.

1. **Payload:** `/var/www/images/../../../etc/passwd`
    
2. **Why it works:** We satisfy the prefix check, then traverse back out immediately.

---

## 🛡️ 5. Remediation & Defense Strategies

### 1️⃣ Use Indirect Object References

Do not expose actual filenames or paths to the user. Use an ID map.

1. **Safe:** `?id=1` -> Server looks up `1` in database -> Database returns `cat.png`.
    
2. **Unsafe:** `?file=cat.png`.
    

### 2️⃣ Validate against a Whitelist

If you must accept filenames, allow only alphanumeric characters.

1. **Regex:** `^[a-zA-Z0-9]+\.png$`
    
2. **Reject:** Any input containing `/`, `\`, or `..`.
    

### 3️⃣ Use `realpath` (Canonicalization)

Before opening a file, resolve the path to its canonical (absolute) form and check if it starts with the expected directory.

```
$base_dir = '/var/www/images/';
$real_path = realpath($base_dir . $_GET['filename']);

if (strpos($real_path, $base_dir) === 0 && file_exists($real_path)) {
    // Safe to open
}
```

---

## ❓ 6. Interview Corner: Common FAQs (Pentest & AppSec)

If you are interviewing for a security role, expect these questions.

### Q1: What is the difference between Directory Traversal and LFI?

**Answer:**

1. **Directory Traversal (Path Traversal):** Allows you to **read** the contents of a file (e.g., `cat /etc/passwd`). The server returns the text.
    
2. **Local File Inclusion (LFI):** Allows you to **execute** the file as code (e.g., PHP, JSP). If you include `/etc/passwd`, it might just print it. But if you include a log file containing PHP code, that code runs on the server.
    

### Q2: How many `../` should you use in a payload?

Answer:

It depends on the depth of the current directory, but it's best to use "too many" rather than too few. Using ../../../../../../etc/passwd works because once you hit the root directory (/), additional ../ are simply ignored by the OS.

### Q3: You found a traversal vulnerability but can't see the output (Blind). How do you exploit it?

Answer:

This is uncommon for traversal (usually you want to read), but if it's blind (e.g., a file deletion vuln):

1. **Time-Based:** Try to access a massive file or a special device file (`/dev/urandom`) that might cause the application to hang while reading.
    
2. **Error-Based:** Try to access a file that doesn't exist vs one that does, and look for differences in error codes.
    

### Q4: Why is `null byte` injection (%00) rare in modern apps?

Answer:

Null byte injection relies on the difference between string handling in high-level languages (like PHP/Java) and the underlying C libraries. Modern versions of PHP (5.3.4+), Java (7u40+), and Python protect against this by treating null bytes in paths as fatal errors or stripping them correctly.

### Q5: How do you bypass a WAF blocking `../`?

**Answer:**

1. **Encoding:** `%2e%2e%2f` (URL), `%252e%252e%252f` (Double URL).
    
2. **Unicode/UTF-8:** `..%c0%af` or `..%e0%80%af` (Overlong UTF-8 representations of `/`).
    
3. **Non-Standard Separators:** `..\` (Windows accepts backslash).
    

### Q6: Can you exploit Directory Traversal to get RCE?

Answer:

Directly? No, traversal just reads files.

Indirectly? Yes.

1. **Read Configs:** Steal passwords/keys to log in via SSH/Admin panel.
    
2. **Log Poisoning:** If it's actually LFI, inject code into logs and traverse to the log file.
    
3. **File Upload Chain:** Upload a shell as `image.php`, then use traversal to access/execute it if the upload folder is restricted.
    

### Q7: What are "Wrapper" protocols in PHP?

Answer:

PHP supports streams like php://filter (to encode file contents, bypassing extension checks) or zip:// (to read files inside archives). These are powerful tools in LFI/Traversal attacks to bypass filters or extract source code.

### Q8: How does `chroot` protect against traversal?

Answer:

chroot changes the root directory for the running process. If the web server is chrooted to /var/www/, then /var/www/ becomes the new /. Even if an attacker uses ../../../../, they cannot ascend higher than the new root, preventing access to /etc/passwd.

### Q9: Why might `....//` work when `../` fails?

Answer:

This exploits non-recursive filtering. If the developer replaces ../ with an empty string once, the input ....// becomes ../ after the middle part is removed. To fix this, the developer should perform the replacement in a loop until no matches remain (or simply reject the input).

### Q10: What is the risk of using `include($_GET['file'])` in PHP?

Answer:

This is the classic recipe for Local File Inclusion (LFI) and Remote File Inclusion (RFI). It allows an attacker to specify any file to be executed by the PHP interpreter. This leads directly to RCE.

---

## 🎭 7. Scenario-Based Questions

### 🎭 Scenario 1: The "Cloud" Traversal

Context: You have directory traversal on a web app running on an AWS EC2 instance. /etc/passwd doesn't give you much. What do you look for?

The Question: How do you leverage this to compromise the cloud account?

The "Hired" Answer:

"I would look for cloud credentials.

1. Check for environment variables in `/proc/self/environ`.
    
2. Check for AWS credentials files: `~/.aws/credentials`.
    
3. If it's an LFI (executable), I'd try to use `http://169.254.169.254/latest/meta-data/` (SSRF) if the file function supports URLs."
    

### 🎭 Scenario 2: The "PDF Generator"

Context: An app takes a URL, fetches the HTML, and converts it to a PDF. You control the URL.

The Question: Is this Directory Traversal?

The "Hired" Answer:

"It sounds more like SSRF (Server-Side Request Forgery) or Local File Read via the PDF engine.

- **Attack:** I would try `file:///etc/passwd` as the URL.
    
- **Result:** If the PDF generator renders local file URIs, the content of `/etc/passwd` will be printed inside the generated PDF."
    

### 🎭 Scenario 3: Windows vs. Linux

Context: You found a vulnerability where filename=..\..\windows\win.ini works. But filename=../../windows/win.ini is blocked.

The Question: Why is this happening?

The "Hired" Answer:

"The server is running on Windows. Windows file systems support both / and \ as separators. The application's WAF or input filter likely only blocks the standard forward slash ../ used in Linux/Web exploits, forgetting that backslash ..\ is valid and dangerous on Windows."

### 🎭 Scenario 4: The "Secure" ID

Context: A developer says: "I validated that the user input is just a number, so include($id) is safe."

The Question: Is he right?

The "Hired" Answer:

"Usually yes, if the validation is strict (e.g., is_numeric() or ^\d+$).

However, if the validation allows Hex or scientific notation, or if there's a type juggling issue (PHP), I might be able to bypass it. But generally, strict type casting to Integer is a strong defense against traversal."

### 🎭 Scenario 5: Log Poisoning

Context: You have LFI but can't upload files. You can read /var/log/apache2/access.log.

The Question: How do you get a shell?

The "Hired" Answer:

"I would use Log Poisoning.

1. I will send a request to the server with a malicious User-Agent header: `<?php system($_GET['c']); ?>`.
    
2. Apache writes this header into `access.log`.
    
3. I use the LFI to include the log file: `?file=/var/log/apache2/access.log&c=ls`.
    
4. The server executes the PHP code stored in the log file, giving me RCE."
    

---

## 🛑 Summary of Part 1

1. **Concept:** Traversing out of the web root to read system files (`/etc/passwd`).
    
2. **Impact:** Information Disclosure (Source code, credentials, system files).
    
3. **Attack:** `../` sequences, absolute paths, and encoding bypasses.
    
4. **Defense:** Validate against whitelists, use `realpath`, avoid direct file access.
