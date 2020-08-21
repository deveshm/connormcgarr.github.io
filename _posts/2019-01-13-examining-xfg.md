---
title: "Exploit Development: Between a Rock and a (Xtended-Flow) Guard Place! Examining XFG"
date:  2019-01-13
tags: [posts]
excerpt: "Taking a look at Microsoft's new forward-edge CFI solution: Xtended Flow Guard"
---
Introduction
---
Previously, I have [blogged](https://connormcgarr.github.io/ROP2) about ROP and the benefits of understanding how it works. Not only is it a viable first-stage payload for obtaining native code execution, it can also be leveraged for things like arbitrary read/write primitives and data-only attacks. Unfortunately, if your end goal is native code execution, there is a good chance you are going to need to overwrite a function pointer in order to hijack control flow. Taking this into consideration, Microsoft implemented [Control Flow Guard](https://docs.microsoft.com/en-us/windows/win32/secbp/control-flow-guard), or CFG, as an optional update back in Windows 8.1. Although it was released before Windows 10, it did not really catch on in terms of "mainstream" exploitation until recent years.

The Blueprint for XFG: CFG
---

CFG is a pretty well documented exploit mitigation, and I have done [my fair share](https://www.crowdstrike.com/blog/state-of-exploit-development-part-1/) of documenting it as well. However, for completeness sake, let's talk about how CFG works and what any potential shortcomings could be.

> Note that before we begin, Microsoft deserves recognition for being among the first to implement a Control Flow Integrity (CFI) solution.

Firstly, a program is compiled and linked with the `/guard:cf` flag. This can be done through the Microsoft Visual Studio tool `cl` (which we will look at later). However, more easily, this can be done by opening Visual Studio and navigating to `Project -> Properties -> C/C++ -> Code Generation` and setting `Control Flow Guard` to `Yes (/guard:cf)`



<img src="{{ site.url }}{{ site.baseurl }}/images/XFG1.png" alt="">

CFG at this point would now be enabled for the program- or in the case of Microsoft binaries, already have CFG enabled (most of them). This causes a bitmap to be created, which essentially is made up of all functions that are "protected by CFG". Since this is a post about XFG, not CFG, we will skip over the technical details of CFG. However, if you are interested to see how CFG works at a lower level, Morten Schenk has an excellent [post](https://improsec.com/tech-blog/bypassing-control-flow-guard-in-windows-10) about its implmenetation in user mode (the Windows kernel has been compiled with CFG, known as kCFG, since Windows 10 1703). On any indirect function call (e.g. `call [rax]`), which is a call to a function that initiates a control flow transfer to a different part of an application, the call is firsly checked by CFG.

Let's take a look at a very simple program that performs a control flow transfer.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG2.png" alt="">

> Note that you will need Microsoft Visual Studio 2019 Preview 16.5 or greater in order to follow along

The above program essentially has the `main()` function call a user-defined function that prints the number specified in `main()`. This will create a control flow transfer, as the main function (which will be the first function to execute), will perform a `call` to the `noCFG()` function. Let's compile this program, so we can view it in IDA.

To compile with the command line tool `cl`, type in "x64 Native Tools Command Prompt for VS 2019 Preview" in the Start menu and run the program as an administrator.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG3.png" alt="">

This will drop you into a special Command Prompt. From here, you will need to navigate to the installation path of Visual Studio, and you will be able to use the `cl` tool for compilation.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG4.png" alt="">
