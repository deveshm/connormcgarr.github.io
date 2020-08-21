---
title: "Exploit Development: Between a Rock and a (Xtended-Flow) Guard Place! Examining XFG"
date:  2019-01-13
tags: [posts]
excerpt: "Taking a look at Microsoft's new forward-edge CFI solution: Xtended Flow Guard"
---
Introduction
---
Previously, I have [blogged](https://connormcgarr.github.io/ROP2) about ROP and the benefits of understanding how it works. Not only is it a viable first-stage payload for obtaining native code execution, but it can also be leveraged for things like arbitrary read/write primitives and data-only attacks. Unfortunately, if your end goal is native code execution, there is a good chance you are going to need to overwrite a function pointer in order to hijack control flow. Taking this into consideration, Microsoft implemented [Control Flow Guard](https://docs.microsoft.com/en-us/windows/win32/secbp/control-flow-guard), or CFG, as an optional update back in Windows 8.1. Although it was released before Windows 10, it did not really catch on in terms of "mainstream" exploitation until recent years.

After a few years, and a few bypasses along the way, Microsoft decided they needed a new Control Flow Integrity (CFI) solution. David Weston gave an overview of XFG at his [talk](https://query.prod.cms.rt.microsoft.com/cms/api/am/binary/RE37dMC) at BlueHat Shanghai 2019. This "finer-grain" CFI solution will be the subject of this blog post. A few things before we start about what this post _is_ and what it _isn't_

1. This post is not an "XFG internals" post. I don't know every single low level detail about it.
2. Don't expect any crazy bypasses- this mitigation is still very new and not very explored.
3. We will spend a bit of time understanding what indirect function calls are via function pointers + what CFG is why it needs XFG in addition to itself.

This post is just going to be a brain dump of what I have learned about XFG after messing with it for a while now. This is simply an "organized brain dump" and isn't meant to be a "learn everything you need to know about XFG in one sitting" post.

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

The above command essentially compiles the program with the `/Zi` flag and the `/INCREMENTAL:NO` linking option. Per [Microsoft Docs](https://docs.microsoft.com/en-us/cpp/build/reference/compiler-options-listed-alphabetically?view=vs-2019), `/Zi` is used to create a .pdb file for symbols (which will be useful to us). `/INCREMENTAL:NO` has been set to instruct `cl` not to use the Incremental linker. This is because the Incremental linker is essentially used for optimization, which can create things like jump thunks. Jump thunks are essentially a small function which only perform a jump to another function, which will hinder the ability to showcase why CFG (And XFG) are very important. Since our "dummy program" is so simple and will get optimized very easily, we are disabling Incremental linking in order to simulat a "Release" build (we are currently building "Debug" builds). However, none of this is really prevalent here- just a point of contention to the reader. Just know we are doing it for our purposes.

The result of the compilation command will place the output file, named `Source.exe` in this case, into the current directory along with a symbol file (.pdb). Now, we can open this application in IDA (you'll need to run IDA as an administrator, as the application is in a privileged directory). Let's take a look at the `main()` function.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFGbb.png" alt="">

Let's examine the assembly above. The above function loads `noCFG()` into RAX. RAX is then moved to `[rsp+38h+var_18]` and since `var_18` is assinged to negative 0x18, we can assume that this will place RAX at `[rsp+0x20]` (a.k.a cause RSP + 0x20 to point to `noCFG()`. Eventually, a call to `[rsp+38h+var_18]` is made. So, we can make a determination that this will call `noCFG()` via a pointer.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG7.png" alt="">

Essentially what is happening here, is that the program is performing a control flow transfer to the `noCFG()` function from the `main()` function

Nice! We know that our program will redirect execution from `main()` to `noCFG()`! Let's say as an attacker, we have an arbitrary write primitive and we were able to overwrite a pointer. Since the function `noCFG()` will be pointed to by something else (`[rsp+38h+var_18]` in this case), we know that if we were able to overwrite that pointer on the stack for instance, we could change what the final `call [rsp+38h+var_18]` actually ends up calling! This is not good from a defensive perspective.

Let's go back and recompile our application with CFG this time.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG8.png" alt="">

This time, we add `/guard:cf` as a flag, as well as a linking option.

Disassembling the `main()` function in IDA again, we notice things look a bit different.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG9.png" alt="">

Very interesting! Instead of making a call directly to `noCFG()` this time, it seems as though the function `__guard_disaptch_icall_fptr` will be invoked. Let's set a breakpoint in WinDbg on the main function and see how this looks after invoking the CFG dispatch function.

After setting a breakpoint on the `main()` function, code execution hits the CFG dispatch function.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG10a.png" alt="">

The CFG disapatch function then performs a dereference and jumps to `ntdll!LdrpDispatchUserCallTarget`. 

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG11.png" alt="">

We won't get into the technical details about what happens here, as this post isn't built around CFG and Morten's blog already explains what will happen. But essentially, at a high level, this function will check the CFG bitmap for the `Source.exe` process and determine if the `Source!noCFG` function is a valid target (a.k.a is it in the bitmap). Obviously this function wasn't over written, so we should have no problems here. After stepping through the function, control flow transfers back to the `noCFG()` function seamlessly.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG12.png" alt="">
