---
layout: mission
title: "Lateral Movement: Techniques, Tools & Theory"
date: 2024-06-04
categories: [Active Directory, Lateral Movement, Exploitation]
toc: true
---

# 🏹 Lateral Movement: The Art of Pivoting

Lateral movement is not just about logging into another machine; it is about using the credentials or tokens you have harvested to expand your control across the network.

This guide explores the mechanics of **Remote Execution**, **Credential Theft**, and **Authentication Abuse**.

---

## 🚀 1. PowerShell Remoting (WinRM)

PowerShell Remoting is the modern, "native" way to manage Windows. Unlike older tools like `psexec` (which drop a noisy binary on the disk), WinRM uses an existing system service, making it much stealthier.

### **The Mechanics: How it Works**
**Protocol:** It runs over HTTP (5985) or HTTPS (5986), making it firewall-friendly.
**The Process:** When you connect, the target machine spawns a child process called `wsmprovhost.exe`. This is the process that actually executes your commands.
**Permissions:** You must generally be a **Local Administrator** on the target to use WinRM.
**Integrity:** The session runs as "High Integrity," meaning bypasses like UAC are usually not needed once connected.

### **Technique A: One-to-One (PSSession)**

**Concept:** Stateful, Interactive. Think of it like SSH. You connect, the session stays open, and variables are saved.

**Double Hop Problem:** If you PSRemote into `Server A`, and then try to copy a file from `Server B`, it will fail.
   
    **Why?** `Server A` cannot pass your credentials to `Server B` ("Double Hop") unless specific delegation settings (CredSSP/Unconstrained) are enabled.

**Command:**
    
    ```
    # Create a persistent session variable
    $sess = New-PSSession -ComputerName dcorp-adminsrv.dollarcorp.moneycorp.local
    # Enter the session
    Enter-PSSession -Session $sess
    ```
    
### **Technique B: One-to-Many (Invoke-Command)**
**Concept:** Stateless, Parallel. You send a "Script Block" (a chunk of code) to the target. The target executes it and sends back the text output.

**Power:** This is the most efficient way to hunt. You can ask 100 servers "Is the Domain Admin logged in?" simultaneously.

**Command:**
   
    ```
    # Execute a block of code on a list of servers
    Invoke-Command -ComputerName (Get-Content servers.txt) -ScriptBlock { Get-Process lsass }
    ```
    
**Fileless Execution:**
    
    ```powershell
    # Loads local script into remote memory. No file drops on target disk.
    Invoke-Command -FilePath C:\Tools\Enum.ps1 -ComputerName target-server
    ```

---


## 🥷 2. WinRS (Stealth Mode)

**WinRS (Windows Remote Shell)** is a built-in CLI tool that uses the same WinRM protocol as PowerShell but behaves differently in logs.

**The Evasion:** PowerShell Remoting creates heavy **Script Block Logs** (Event ID 4104) and **Module Logs**. WinRS executes commands via `cmd.exe` over WinRM, which often bypasses these specific PowerShell logging mechanisms.

**Usage:**
    
    ```
    winrs -r:dcorp-adminsrv cmd
    ```
    
**Port Forwarding:** WinRS can tunnel traffic, acting as a poor man's proxy.
   
    ```
    # Listens on 8080 locally, forwards to 172.16.x.x:80 via the target
    
    winrs -r:target "netsh interface portproxy add v4tov4 listenport=8080 connectport=80 connectaddress=172.16.x.x"
    ```

---

## 🔑 3. Credential Dumping (The Harvest)

Once you land on a box, you need to "live off the land" or bring tools to steal credentials. The target is usually **LSASS.exe** (Local Security Authority Subsystem Service), which holds active credentials in RAM.

### **What are we stealing?**
1.  **Cleartext Passwords:** (Rare in modern Windows due to WDigest fixes, but possible).
2.  **NTLM Hashes:** Can be used for Pass-the-Hash.
3.  **Kerberos Tickets (TGT/TGS):** Can be used for Pass-the-Ticket.
4.  **AES Keys:** Can be used for Overpass-the-Hash.

### **Tool Hierarchy (From Noisy to Stealthy)**
1.  **Mimikatz:** The gold standard, but highly flagged by AV/EDR.
    `sekurlsa::logonpasswords` (Dumps everything).
    `sekurlsa::ekeys` (Dumps AES/DES keys for tickets).
2.  **SafetyKatz:** Dumps LSASS memory to a temp file, then runs Mimikatz on the file. This prevents Mimikatz from interacting directly with the live LSASS process (which crashes less often).
3.  **Dumpert / Comsvcs.dll:** "Living off the Land". Uses built-in Windows DLLs to create a dump file.

    **Mechanism:** `comsvcs.dll` has a function `MiniDump` intended for debugging. Attackers abuse it.

    **Command:**

      ```
        rundll32.exe C:\windows\System32\comsvcs.dll, MiniDump <LSASS_PID> C:\Temp\lsass.dmp full
        ```

---

## 🎭 4. Authentication Attacks (The Reuse)

You have the credentials. Now, how do you use them?

### **A. Pass-the-Hash (PtH)**
**Protocol:** **NTLM**.

**Mechanism:** The NTLM protocol requires the client to encrypt a "Challenge" using their password hash. If you have the hash, you can do the math without knowing the password.

**Limitation:** You typically cannot access services that *require* Kerberos (like some SQL configs or file shares with specific policies) or machines in "Protected Users" groups.

**Command:**
   
    ```
    # Spawns a new PowerShell window authenticated as the target
    Invoke-Mimikatz -Command '"sekurlsa::pth /user:Admin /domain:target /ntlm:<HASH> /run:powershell.exe"'
    ```

### **B. Overpass-the-Hash (OPtH)**

**Protocol:** **Kerberos**.

**Mechanism:** You use the user's NTLM hash or AES Key to perform a full Kerberos authentication flow (AS-REQ) with the Domain Controller. The DC validates the encryption and sends back a valid **TGT (Ticket Granting Ticket)**.

**Why use it?** It converts a "Hash" into a "Ticket." Tickets are more powerful because they allow access to services via Hostname (e.g., `\\fs01.corp.local`) and behave like a legitimate logon.

**OpSec Note:** Always use **AES256 keys** if possible. Using NTLM (RC4) to request a TGT is a deprecated behavior that modern SOCs flag immediately.

**Command (Rubeus):**
   
    ```
    # Requests TGT using AES Key and caches it in the current session
    Rubeus.exe asktgt /user:admin /aes256:<KEY> /opsec /ptt
    ```

---

## 👑 5. DCSync (Domain Dominance)

DCSync is not an exploit; it is a feature abuse. It uses the **DRS (Directory Replication Service)** protocol, which Domain Controllers use to sync data.

### **The Mechanics**
1.  **Impersonation:** The attacker tells the DC, "Hello, I am also a Domain Controller. Please send me the latest password updates."
2.  **Permissions:** This requires the `DS-Replication-Get-Changes-All` extended right.
3.  **No Code Execution:** Unlike dumping LSASS (which touches memory on the target), DCSync is purely network traffic. You do not need to log in to the DC itself.

### **The Targets**
**KRBTGT:** The account that signs all Kerberos tickets. Dumping this allows **Golden Tickets**.

**Specific Users:** Dumping the hash of a CEO or Admin without touching their laptop.

---

## 🐧 6. Impacket (Linux-to-Windows Movement)

If you are attacking from a Linux machine (like Kali), you cannot use PowerShell natively. You use Python scripts from the **Impacket** library.

### **wmiexec.py (The Stealthy Choice)**
**Mechanism:** Uses WMI (Windows Management Instrumentation) (Port 135/445).

**How:** It executes commands via DCOM, redirects output to a temp file on the target, reads the file, and deletes it.

**Pros:** Very stealthy; doesn't drop binaries.

### **psexec.py (The Reliable Choice)**

**Mechanism:** SMB (Port 445).

**How:** It uploads a service binary (`remcomsvc`) to the `ADMIN$` share, installs it as a service, and executes it.

**Cons:** Very noisy. Dropping a binary and creating a service triggers almost every AV/EDR.

---

## ❓ 7. Interview Corner: 10 Core Questions

**Q1: What is the technical difference between Pass-the-Hash and Overpass-the-Hash?**

**Answer:** Pass-the-Hash uses the NTLM hash to perform NTLM authentication directly with the target server. Overpass-the-Hash uses the NTLM hash (or AES keys) to authenticate with the KDC (Domain Controller) to obtain a Kerberos TGT, which is then used to access network resources.

**Q2: Why does PowerShell Remoting (WinRM) require the "Double Hop" workaround?**

**Answer:** By default, Kerberos does not allow a server to delegate your credentials to a second server to prevent unconstrained impersonation. When you jump Server A -> Server B, Server A cannot prove to Server B that it is you, unless CredSSP or Delegation is explicitly configured.

**Q3: Why is dumping LSASS via `comsvcs.dll` considered "Living off the Land"?**

**Answer:** "Living off the Land" means using binaries already present on the OS to perform attacks. `comsvcs.dll` is a legitimate Windows library. Using it allows an attacker to dump memory without bringing malicious tools like Mimikatz onto the disk, aiming to bypass static AV detection.

**Q4: Which encryption type should you use for Overpass-the-Hash to remain stealthy?**

**Answer:** You should use **AES256**. Requesting a Kerberos ticket using RC4 (which is what NTLM hashes use) is known as "Encryption Downgrade" and is heavily monitored by tools like Microsoft Defender for Identity (MDI).

**Q5: What rights are required to perform a DCSync attack?**

**Answer:** The account requires `Replicating Directory Changes` and `Replicating Directory Changes All`. These are typically held by Domain Admins, Enterprise Admins, and Domain Controller machine accounts.

**Q6: How does `wmiexec.py` execute commands without dropping a binary?**

**Answer:** It uses WCOM/DCOM to instantiate a shell process (`cmd.exe`) on the target. It redirects the output of the command to a temporary text file on the target's admin share (like `C:\Windows\Temp`), reads that file over SMB, displays the output, and then deletes the file.

**Q7: What is the "AdminCount" attribute and why does it matter?**

**Answer:** The `AdminCount=1` attribute marks users and groups that are (or were) protected high-privilege accounts (like Domain Admins). It signifies that the **SDProp** process controls their permissions. Attackers search for `AdminCount=1` users to identify high-value targets.

**Q8: Explain "Logon Type 9" in the context of Mimikatz.**

**Answer:** When you perform `sekurlsa::pth` with the `/run` command, Mimikatz creates a new process with **Logon Type 9 (NewCredentials)**. This means the process effectively says: "I am the local user for all local tasks, but for any outbound network connection, use these injected credentials/hashes."

**Q9: Can you perform a DCSync if you only have local admin on a Domain Controller?**

**Answer:** Yes. If you are a Local Admin on a DC, you can elevate to `SYSTEM`. The DC's machine account (`DC01$`) has the replication rights required for DCSync. You can run the attack in the context of the machine account.

**Q10: What is the primary indicator of a standard `psexec` attack?**

**Answer:** The creation of a new Service on the target machine (often named `PSEXESVC` by default) and the dropping of a binary executable into the `ADMIN$` share (`C:\Windows`).


**Q11: What process is spawned on the target machine when you use PowerShell Remoting?**

**Answer:** The wsmprovhost.exe process.

**Q12: Why should you prefer AES keys over NTLM hashes when performing Overpass-the-Hash?**

**Answer:** Requesting a Kerberos ticket using an NTLM hash forces the encryption type to RC4. In modern AD environments, RC4 usage is often flagged as an anomaly by security tools like Microsoft Defender for Identity (MDI). AES is the standard/expected encryption.


**Q13: How can you move laterally if WinRM (Ports 5985/5986) is blocked?**

**Answer:** I would look for other protocols like SMB (Port 445) to use tools like psexec or smbexec, or WMI (Port 135) to execute commands via wmic or wmiexec.py.

**Q14: What is the risk of dumping LSASS using rundll32 and comsvcs.dll?**

Answer: While it uses built-in binaries (Living off the Land), the specific command line arguments are well-known signatures. Most EDRs will instantly flag the execution of MiniDump via comsvcs.dll.

**Q15: Can you use PowerShell Remoting with a local user account?**

**Answer:** By default, no. Determine scenarios (e.g., non-domain joined) require specific registry changes (LocalAccountTokenFilterPolicy) to allow remote administrative access for local accounts.

**Q16: What is a "Silver Ticket"?**

**Answer:** A Silver Ticket is a forged Service Ticket (TGS). It is created using the password hash of a specific Service Account (like SQL or IIS). It grants access only to that specific service, unlike a Golden Ticket which grants access to everything.

**Q17: Why involves "One-to-Many" remoting (Invoke-Command)?**

**Answer:** It allows an attacker (or admin) to execute the same script block on hundreds of machines in parallel. This is highly effective for hunting specific files, processes, or local admin rights across the entire domain quickly.

**Q18: What is the "Double Hop" problem in PowerShell Remoting?**

**Answer:** When you PSRemote into Server A, your credentials are used to authenticate to Server A. If you try to jump from Server A to Server B, it fails because Server A cannot delegate your credentials to the second hop (unless CredSSP or unconstrained delegation is configured).

---

## 🎭 8. Scenario-Based Questions (Bar Raiser)

**Scenario 1: The Invisible Wall**

**Context:** You have a Domain Admin password. You try to RDP into a server, but it fails. You try `psexec`, it fails. However, you know the credentials are valid.

**Question:** What security control is likely in place?

**Answer:** The user might be in the **"Protected Users"** security group. This group enforces strict security: it blocks NTLM authentication (killing psexec/standard RDP if Kerberos fails), prevents credential caching, and forces Kerberos-only authentication with strong encryption.

**Scenario 2: The "Jump" Box**

**Context:** You are on a compromised web server (`Web01`). You found keys for the Database Admin. You want to query the Database Server (`SQL01`), but `Web01` cannot reach `SQL01` directly due to network segmentation. However, `App01` can reach `SQL01`.

**Question:** How do you execute the query?

**Answer:** I would use **WinRS** to execute a command on `App01` (using the DB keys), but use the command to chain the connection to `SQL01`. Or, simpler: PSRemote into `App01`, load the session, and then from `App01`, connect to `SQL01`. This effectively pivots through the jump box.

**Scenario 3: The Empty LSASS**

**Context:** You act as a local admin on a machine and run Mimikatz `logonpasswords`. The output shows `(null)` for passwords and mostly empty NTLM fields.

**Question:** Why is this happening and how do you get around it?

**Answer:** The machine likely has **WDigest** disabled (preventing cleartext storage) or is running **Credential Guard**, which isolates LSASS secrets in a virtualized container. To bypass Credential Guard, you cannot just "dump" LSASS. You would need to use specific bypasses (rare) or focus on capturing the token while it's being used (Token Impersonation).

**Scenario 4: The Linux Pivot**

**Context:** You have compromised a Linux web server joined to the AD domain. You found a keytab file.

**Question:** How do you move laterally to a Windows server?

**Answer:** I would use the **keytab** file to request a Kerberos TGT using the `kinit` command on Linux. Once I have the ticket cached in the `KRB5CCNAME` environment variable, I can use **Impacket** tools (like `psexec.py -k` or `wmiexec.py -k`) to authenticate to the Windows servers using Kerberos authentication from the Linux box.

**Scenario 5: The Noisy Service**

**Context:** You are red teaming. You used `psexec` to jump to a server. 2 minutes later, your access is cut.

**Question:** What happened?

**Answer:** The creation of the `PSEXESVC` service triggered a high-severity alert in the EDR/SIEM. The SOC isolated the host. In a red team engagement, always prefer `WinRM` or `WMI` (fileless) over `psexec` (service creation) to avoid this exact signature.


**Scenario 6: The Blocked Hash**

**Context:** You have the NTLM hash of the Administrator. You try sekurlsa::pth but get an "Access Denied" when trying to access the C$ share of a target.

**Question:** Why might this be failing even if the hash is correct?

**Answer:** The target might have KB2871997 patch or LocalAccountTokenFilterPolicy set to 0, which prevents remote administrative access for local accounts (unless it's the built-in RID 500 Admin). Or, the target might be in a "Protected Users" group which disables NTLM authentication entirely.

**Scenario 7: The Silent Movement**

**Context:** You need to execute a command on a remote server, but you want to generate minimal logs. Enter-PSSession creates 4624 login events and PowerShell transaction logs.

**Question:** What tool would you use?

**Answer:** I would use WinRS (winrs -r:target cmd). It runs over the same WinRM protocol but typically generates less verbose script block logging than a full PowerShell session.

**Scenario 8: The Persistence Plan**

**Context:** You have Domain Admin rights. You want to ensure you can always get back in, even if the DA password changes.

**Question:** What attack allows this?

**Answer:** I would perform a DCSync to dump the krbtgt hash. With this hash, I can create a Golden Ticket (forged TGT) valid for 10 years, allowing me to impersonate any user at any time.

**Scenario 9: The Legacy App**

**Context:** You compromised a web server running an old app. You dumped credentials and found an NTLM hash. You want to use it to access a file share.

**Question:** Should you use PtH or OPtH?

**Answer:** If you just need to access a file share (SMB), Pass-the-Hash (PtH) is sufficient and simpler. Overpass-the-Hash is better if you need a TGT to access services that require Kerberos authentication or to bypass NTLM restrictions.

**Scenario 10: The "NetOnly" Process**

**Context:** Rubeus created a process with /createnetonly. You type whoami in that window and it says "StudentUser" (your low-priv user).

**Question:** Did the exploit fail?

**Answer:** No. CreateNetOnly creates a process that looks like the local user but authenticates across the network using the injected ticket/credentials. whoami checks the local token. To verify the exploit, try dir \\dc01\c$. If that works, the network authentication is using the elevated credentials.
