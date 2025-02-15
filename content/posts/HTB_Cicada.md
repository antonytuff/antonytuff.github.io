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




###### Scanning through Nmap

## Enumeration

## Information Gathering

### Intial FootHold


## Privilege Escalation


### Persistence


## 📜 Conclusion
