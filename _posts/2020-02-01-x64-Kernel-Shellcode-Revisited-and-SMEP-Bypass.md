---
title: "Exploit Development: Panic! At The Kernel - Token Stealing Payloads Revisited on Windows 10 x64 and Bypassing SMEP (UNDER CONSTRUCTION)"
date:  2020-02-01
tags: [posts]
excerpt: "Revisiting token stealing payloads on Windows 10 x64 and diving into bypassing SMEP"
---
Introduction
---
Same ol' story with this blog post- I am trying to continue and further my research/overall knowledge on Windows kernel exploitation in order to prepare for [AWE](https://www.blackhat.com/us-20/training/schedule/index.html#advanced-windows-exploitation-19158), and to garner more experience with exploit development. Previously I have talked about [a](https://connormcgarr.github.io/Part-2-Kernel-Exploitation/) [couple](https://connormcgarr.github.io/Part-1-Kernel-Exploitation/) of vulnearbility classes on x86 with Windows 7, which has minimal protections. I wanted to do a deeper dive into token stealing payloads, which I have previously talked about on x86, and see what differences x64 will have and see what more clarity I can provide. I also wanted to dive into x64, which is a far more common architecture in 2020, and understand protections such as Supervised Mode Execution Prevention (SMEP).

Gimme Dem Tokens!
---
As apart of Windows, there is something known as the SYSTEM process. The SYSTEM process, PID of 4, houses the majority of kernel mode system threads. The threads stored in the SYSTEM process, only run in context of kernel mode. Recall that a process is a "container" for threads. A thread is the actual item within a process that performs the execution of code. You may be asking "how does this help us?" Especially, if you did not see my last post. In Windows, each process object, known as `_EPROCESS`, has something known as an [access token](https://docs.microsoft.com/en-us/windows/win32/secauthz/access-tokens). This determines the security context of a process or a thread. Since the SYSTEM process houses execution of kernel mode code, it will need to run in a security context that allows it to access the kernel. This would require system or administrative privilege. This is why our goal will be to identify the access token value of the SYSTEM process and copy it to a process that we control, or the process we are using to exploit the system. From there, we can spawn `cmd.exe` from the now privileged process, which will grant us `NT AUTHORITY\SYSTEM` priviled code execution.

Identifying the SYSTEM Process Access Token
---
We will use Windows 10 x64 to outline this overall process. First, boot up WinDbg on your debugger machine and start kernel debugging you debuggee machine (see my [post](https://connormcgarr.github.io/Part-1-Kernel-Exploitation/) on setting up a debugging enviornment. In addition, I noticed on Windows 10, I had to execute the following command on my debugger machine after completing the `bcdedit.exe` commands: `bcdedit.exe /dbgsettings serial debugport:1 baudrate:115200`)

Once that is setup, execute the following command, to dump the process list:

`!process 0 0`

<img src="{{ site.url }}{{ site.baseurl }}/images/64_1.png" alt="">

This returns a few elements of each process. We are most interested in the "Process address", which has been outlined in the image above at address `0xffffe60284651040`. Since we have enumerated that address, we can enumerate much more detailed information about the `_EPROCESS` structure of the SYSTEM process.

`dt nt!_EPROCESS <Process address>`

<img src="{{ site.url }}{{ site.baseurl }}/images/64_2.png" alt="">

`dt` will display information about various variables, data types, etc. As you can see from the image above, various information about the SYSTEM process has been displayed. If you continue down the kd windows in WinDbg, you will see the `Token` element, at an offset of 0x358.

<img src="{{ site.url }}{{ site.baseurl }}/images/64_3.png" alt="">

What does this mean? That means for each process on Windows, the access token is located at an offset of 0x358. We will for sure be using this information later. Before moving on though, let's take a look at how the `Token` element is stored.

As you can see from the above image, there is something called `_EX_FAST_REF`, or an Executive Fast Reference union. The difference between a union and a structure, is that a union stores data types at the same memory location (notice there is no difference in offset of the elements of `_EX_FAST_REF` in the image below). This is what the access token of a process is stored in. Let's take a closer look at this structure.

`dt nt!_EX_FAST_REF`

<img src="{{ site.url }}{{ site.baseurl }}/images/64_4.png" alt="">

Take a look at the `RefCnt` element. This is a value, appended to the access token, that keeps track of references of the access token. On x86, this is 3 bits. On x64 (which is our current architecture) this is 4 bits, as shown above. We want to clear these bits out, using logical AND. That way, we just extract the actual value of the `Token`, and not other unnecessary metadata.

To extract the value of the token, we simply need to view the `EX_FAST_REF` union of the SYSTEM process at an offset of 0x358 (which is where our token resides). From there, we can figure out how to go about clearing out `RefCnt`.

`dt nt!_EX_FAST_REF <Process address>+0x358`

<img src="{{ site.url }}{{ site.baseurl }}/images/64_5.png" alt="">

As you can see, `RefCnt` is equal to 0y0111. 0y denotes a binary value. So this means `RefCnt` in this instance equals 7 in decimal.

So, let's use logical AND to try to clear out those last few bits.

`? TOKEN & 0xf`

<img src="{{ site.url }}{{ site.baseurl }}/images/64_6.png" alt="">

As you can see, the result is 7. This is not the value we want- it is actually the inverse of it. Logic tells us, we should take the inverse of 0xf, -0xf.

<img src="{{ site.url }}{{ site.baseurl }}/images/64_7.png" alt="">

So- we have finally extracted the value of the raw access token. At this point, let's see what happens when we copy this token to a normal `cmd.exe` session.

Openenig a new `cmd.exe` process on the debuggee machine:

<img src="{{ site.url }}{{ site.baseurl }}/images/64_7_a.png" alt="">

After spawning a `cmd.exe` process on the debuggee, let's identify the process address in the debugger.

`!process 0 0 cmd.exe`

<img src="{{ site.url }}{{ site.baseurl }}/images/64_8.png" alt="">

Now that we have the process address located at `0xffffe6028694d580` for `cmd.exe`. We also know, based on our research earlier, that the `Token` of a process is located at an offset of 0x358 from the process. Let's Use WinDbg to overwrite the `cmd.exe` access token with the access token of the SYSTEM process.

<img src="{{ site.url }}{{ site.baseurl }}/images/64_9.png" alt="">

Now, let's take a look back at our previous `cmd.exe` process.

<img src="{{ site.url }}{{ site.baseurl }}/images/64_10.png" alt="">

As you can see, `cmd.exe` has become a privileged process! Now the only question remains- how do we do this dynamically with a piece of shellcode?
