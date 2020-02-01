---
title: "Exploit Development: Panic! At The Kernel - Token Stealing Payloads Revisited on x64 and Bypassing SMEP (UNDER CONSTRUCTION)"
date:  2020-02-01
tags: [posts]
excerpt: "Revisiting token stealing payloads on Windows 10 x64 and diving into bypassing SMEP"
---
Introduction
---
Same ol' story with this blog post- I am trying to continue and further my research/overall knowledge on Windows kernel exploitation in order to prepare for [AWE](https://www.blackhat.com/us-20/training/schedule/index.html#advanced-windows-exploitation-19158), and to garner more experience with exploit development. Previously I have talked about [a](https://connormcgarr.github.io/Part-2-Kernel-Exploitation/) [couple](https://connormcgarr.github.io/Part-1-Kernel-Exploitation/) of vulnearbility classes on x86 with Windows 7, which has minimal protections. I wanted to do a deeper dive into token stealing payloads, which I have previously talked about on x86, and see what differences x64 will have and see what more clarity I can provide. I also wanted to dive into x64, which is a far more common architecture in 2020, and understand protections such as Supervised Mode Execution Prevention (SMEP).

Gimme Dem Tokens!
---
As apart of Windows, there is something known as the SYSTEM process. The SYSTEM process, PID of 4, houses the majority of kernel mode system threads. The threads stored in the SYSTEM process, only run in context of kernel mode. Recall that a process is a "container" for threads. A thread is the actual item within a process that performs the execution of code. You may be asking "how does this help us?" Especially, if you did not see my last post. In Windows, each process object, known as `_EPROCESS`, has something known as an [access token](https://docs.microsoft.com/en-us/windows/win32/secauthz/access-tokens). This determines the security context of a process or a thread. Since the SYSTEM process houses execution of kernel mode code, it will need to run in a security context that allows it to access the kernel. This would require system or administrative privilege. This is why our goal will be to identify the access token value of the SYSTEM process and copy it to a process that we control, or the process we are using to exploit the system. From there, we can spawn `cmd.exe` from the now privileged process, which will grant us `NT AUTHORITY\SYSTEM privileges`.
