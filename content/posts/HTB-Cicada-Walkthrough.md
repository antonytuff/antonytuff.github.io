---
title: "HacktheBox Walkthrough -Cicada"
date: 2025-01-10 T00:00:00+02:00
draft: false
author: "Anthony Tuff"
tags: ["htb"]
categories: ["htb"]
---
# Cicada- Hack the Box Walkthrough

## Information Gathering
As usual, I started with information gathering to detect open ports and identify services running on the target server. I ran an Nmap scan to probe the system for vulnerabilities and service banners.

- **53 (DNS):** Might be a domain controller. A prime target for domain enumeration.
- **88 (Kerberos):** Active Directory? Kerberos authentication service is active. This confirms the presence of Active Directory and potential ticket or AD related attacks.
- **135/445 (SMB):** File shares? We can dig out to see if there are any shares for any sensitive files or credentials.
- **389/636 (LDAP/LDAPS):** User/group hunting grounds.
- **5985 (WinRM):**  WinRM is enabled, opening up the possibility of remote command execution. if we have a valid username we can see if we can get a shell access.

```bash
masscan -e tun0 -p1-65535 10.10.11.35 --rate=1000
nmap -sV  -A  -sC --min-rate=1000s 10.10.11.35

The screenshot shows the mass results output


#### **Open Ports and Services:**
``` bash
domain :`cicada.htb`.
Subject Common Name (CN): `CICADA-DC.cicada.htb`
```
Here is the table with the additional "Possible Attack Vectors" column:

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
|        |                |                                                                                                           |                                                                                                      |

Using Crackmapexec to check the hostname and Windows server type
I also decided to test if we can perform **RID Brute-forcing** using crackmapexec to enumerate user accounts in the Domain Controller (DC). This method works by leveraging Relative Identifier (RID) values and exploits the ability to query Security Identifiers (SIDs) and resolve them to usernames, often without requiring authentication
Thanks to ippsec you can check manager box

*Hulla!* We have successfully enumerated a list of possible usernames. With this list, we can now attempt AS-REP Roasting attack to check if any accounts allow pre-authentication and retrieve their hashed credentials.) this didn't yield out any results as shown in the screenshot below
##### RID Brute-forcing with CrackMapExec




##### SMB Loot
Since port 445 (SMB) was open, we can look for potential SMB shares. I fired up SMB Client and found two interesting shares:. 
```
- DEV: Custom share (possibly containing files).
- HR: Custom share (Contains a file named Notice from HR.txt).
```
When looking for shares, I managed to enumerate the below shares. 
- **ADMIN$**: Remote Admin share (usually for administrative tasks).
- **C$**: Default administrative share for the C: drive.
- **DEV**: Custom share (possibly containing files).
- **HR**: Custom share (could be sensitive—requires exploration).
- **IPC$**: Used for inter-process communication. 
- **NETLOGON**: Scripts for logging in domain accounts.
- **SYSVOL**: Policies and scripts for domain controllers