---
title: "Computer Literacy"
date: 2023-01-19T00:00:00+02:00
draft: false
author: "Nathaniel Harari"
tags: ["tech"]
categories: ["tech"]
---
# Cicada- Hack the Box Walkthrough

## Information Gathering
As usual, I started with information gathering to detect open ports and identify services running on the target server. I ran an Nmap scan to probe the system for vulnerabilities and service banners.

- **53 (DNS):** Might be a domain controller. A prime target for domain enumeration.
- **88 (Kerberos):** Active Directory? Kerberos authentication service is active. This confirms the presence of Active Directory and potential ticket or AD related attacks.
- **135/445 (SMB):** File shares? We can dig out to see if there are any shares for any sensitive files or credentials.
- **389/636 (LDAP/LDAPS):** User/group hunting grounds.
- **5985 (WinRM):**  WinRM is enabled, opening up the possibility of remote command execution. if we have a valid username we can see if we can get a shell access.