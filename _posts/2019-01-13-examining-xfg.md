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

CFG at this point would now be enabled for the program- or in the case of Microsoft binaries, already have CFG enabled (most of them). This causes a bitmap to be created, which essentially is made up of all functions that are "protected by CFG". Since this is a post about XFG, not CFG, we will skip over the technical details of CFG. However, if you are interested to see how CFG works at a lower level, Morten Schenk has an excellent [post](https://improsec.com/tech-blog/bypassing-control-flow-guard-in-windows-10) about its implementation in user mode (the Windows kernel has been compiled with CFG, known as kCFG, since Windows 10 1703). On any indirect function call (e.g. `call [rax]` where RAX contains a function address or a function pointer), which is a call to a function that initiates a control flow transfer to a different part of an application, the call is firsly checked by CFG.

Let's take a look at a very simple program that performs a control flow transfer.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG2.png" alt="">

> Note that you will need Microsoft Visual Studio 2019 Preview 16.5 or greater in order to follow along

The above program essentially has a `main()` function that calls a user-defined function called `noCFG()`. However, instead of directly invoking the function in `main()`, the `noCFG()` function is firstly assigned to a function pointer called `void (*functionPointer)`. This function pointer is what will be technically invoked in `main()`. However, since this function pointer points to `noCFG()`, `noCFG()` will execute eventually. In the end, this whole program will simply just print "You don't have CFG enabled! Shame on you. Here is a random number: 10".

However, that is not what is important. What is important is that this will create a control flow transfer, as the main function will perform a `call` to the function pointer `void (*functionPointer)`. Since this function pointer points to the`noCFG()` function, the `noCFG()` function is going to be executed. Let's compile this program, so we can view it in IDA.

To compile with the command line tool `cl`, type in "x64 Native Tools Command Prompt for VS 2019 Preview" in the Start menu and run the program as an administrator.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG3.png" alt="">

This will drop you into a special Command Prompt. From here, you will need to navigate to the installation path of Visual Studio, and you will be able to use the `cl` tool for compilation.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG4.png" alt="">

Let's compile our program now!

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG5a.png" alt="">

The above command essentially compiles the program with the `/Zi` flag and the `/INCREMENTAL:NO` linking option. Per [Microsoft Docs](https://docs.microsoft.com/en-us/cpp/build/reference/compiler-options-listed-alphabetically?view=vs-2019), `/Zi` is used to create a .pdb file for symbols (which will be useful to us). `/INCREMENTAL:NO` has been set to instruct `cl` not to use the Incremental linker. This is because the Incremental linker is essentially used for optimization, which can create things like jump thunks, which are used for optimization. Jump thunks are essentially a small function which only perform a jump to another function, which will hinder the ability to showcase why CFG (And XFG) are very important. It is also important to note that this "dummy program" is being built in "Debug" mode, which has Incremental linking on by default. Release builds do not- so essentially we would like to to "simulate a Release build" of a project.

The result of the compilation command will place the output file, named `Source.exe` in this case, into the current directory along with a symbol file (.pdb). Now, we can open this application in IDA (you'll need to run IDA as an administrator, as the application is in a privileged directory). Let's take a look at the `main()` function.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG6b.png" alt="">

Let's examine the assembly above. The above function loads `noCFG()` into RAX. RAX is then moved to `[rsp+38h+var_18]` and since `var_18` is assinged to negative 0x18, we can assume that this will place RAX at `[rsp+0x20]` (a.k.a cause RSP + 0x20 to point to `noCFG()`. Eventually, a call to `[rsp+38h+var_18]` is made. So, we can make a determination that this will call `noCFG()` via a pointer.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG7.png" alt="">

Essentially what is happening here, is that the program is performing a control flow transfer to the `noCFG()` function from the `main()` function

Nice! We know that our program will redirect execution from `main()` to `noCFG()`! Let's say as an attacker, we have an arbitrary write primitive and we were able to overwrite a function pointer, such as `noCFG()`.




