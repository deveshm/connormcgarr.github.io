---
title: "Exploit Development: Between a Rock and a (Xtended-Flow) Guard Place: Examining XFG"
date:  2019-01-13
tags: [posts]
excerpt: "Taking a look at Microsoft's new forward-edge CFI solution: Xtended Flow Guard"
---
Introduction
---
Previously, I have [blogged](https://connormcgarr.github.io/ROP2) about ROP and the benefits of understanding how it works. Not only is it a viable first-stage payload for obtaining native code execution, but it can also be leveraged for things like arbitrary read/write primitives and data-only attacks. Unfortunately, if your end goal is native code execution, there is a good chance you are going to need to overwrite a function pointer in order to hijack control flow. Taking this into consideration, Microsoft implemented [Control Flow Guard](https://docs.microsoft.com/en-us/windows/win32/secbp/control-flow-guard), or CFG, as an optional update back in Windows 8.1. Although it was released before Windows 10, it did not really catch on in terms of "mainstream" exploitation until recent years.

After a few years, and a few bypasses along the way, Microsoft decided they needed a new Control Flow Integrity (CFI) solution- hence XFG, or Xtended Flow Guard. David Weston gave an overview of XFG at his [talk](https://query.prod.cms.rt.microsoft.com/cms/api/am/binary/RE37dMC) at BlueHat Shanghai 2019, and it is pretty much the only public information we have at this time about XFG. This "finer-grained" CFI solution will be the subject of this blog post. A few things before we start about what this post _is_ and what it _isn't_:

1. This post is not an "XFG internals" post. I don't know every single low level detail about it.
2. Don't expect any bypasses from this post- this mitigation is still very new and not very explored.
3. We will spend a bit of time understanding what indirect function calls are via function pointers, what CFG is, and why XFG doesn't mean the end of CFG.

This is simply going to be an "organized brain dump" and isn't meant to be a "learn everything you need to know about XFG in one sitting" post. This is just simply documenting what I have learned after messing around with XFG for a while now.

The Blueprint for XFG: CFG
---

CFG is a pretty well documented exploit mitigation, and I have done [my fair share](https://www.crowdstrike.com/blog/state-of-exploit-development-part-1/) of documenting it as well. However, for completeness sake, let's talk about how CFG works and what any potential shortcomings could be.

> Note that before we begin, Microsoft deserves recognition for being the first to implement a Control Flow Integrity (CFI) solution.

Firstly, to enable CFG, a program is compiled and linked with the `/guard:cf` flag. This can be done through the Microsoft Visual Studio tool `cl` (which we will look at later). However, more easily, this can be done by opening Visual Studio and navigating to `Project -> Properties -> C/C++ -> Code Generation` and setting `Control Flow Guard` to `Yes (/guard:cf)`

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG1.png" alt="">

CFG at this point would now be enabled for the program- or in the case of Microsoft binaries, they would already be CFG enabled (most of them). This causes a bitmap to be created, which essentially is made up of all functions within the process space that are "protected by CFG". Then, before an indirect function call is made (we will explore what an indirect call is shortly if you are not familiar), the function being called is sent to a special CFG function. This function checks to make sure that the function being called is a part of the CFG bitmap. If it is, the call goes through. If it isn't, the call fails.

Since this is a post about XFG, not CFG, we will skip over the technical details of CFG. However, if you are interested to see how CFG works at a lower level, Morten Schenk has an excellent [post](https://improsec.com/tech-blog/bypassing-control-flow-guard-in-windows-10) about its implementation in user mode (the Windows kernel has been compiled with CFG, known as kCFG, since Windows 10 1703. Note that Virtualization-Base Security, or VBS, is required for kCFG to be enforced. However, even when VBS is disabled, kCFG has some limited functionality. This is beyond the scope of this blog post).

Moving on, let's examine how an indirect function call (e.g. `call [rax]` where RAX contains a function address or a function pointer), which initiates a control flow transfer to a different part of an application, looks without CFG or XFG. To do this, let's take a look at a very simple program that performs a control flow transfer.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFGCode1s.png" alt="">

> Note that you will need Microsoft Visual Studio 2019 Preview 16.5 or greater in order to follow along.

Let's talk about what is happening here. Firstly, this code is intentionally written this way and is obviously not the most efficient way to do this. However, it is done this way to help simulate a function pointer overwrite and the benefits of XFG/CFG.

Firstly, we have a function called `void cfgTest()` that just prints a sentence. This function is then assigned to a function pointer called `void (*cfgTest1)`, which actually is an array. Then, in the `main()` function, the function pointer `void (*cfgTest1)` is executed. Since `void (*cfgtest1)` is pointing to `void cfgTest()`, this will actually just cause `void (*cfgtest1)` to just execute `void cfgTest()`. This will create a control flow transfer, as the `main()` function will perform a call to the `void (*cfgTest1)` function, which will then call the `void cfgTest()` function.

To compile with the command line tool `cl`, type in "x64 Native Tools Command Prompt for VS 2019 Preview" in the Start menu and run the program as an administrator.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG3.png" alt="">

This will drop you into a special Command Prompt. From here, you will need to navigate to the installation path of Visual Studio, and you will be able to use the `cl` tool for compilation.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG4.png" alt="">

Let's compile our program now!

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG5a.png" alt="">

The above command essentially compiles the program with the `/Zi` flag and the `/INCREMENTAL:NO` linking option. Per [Microsoft Docs](https://docs.microsoft.com/en-us/cpp/build/reference/compiler-options-listed-alphabetically?view=vs-2019), `/Zi` is used to create a .pdb file for symbols (which will be useful to us). `/INCREMENTAL:NO` has been set to instruct `cl` not to use the incremental linker. This is because the incremental linker is essentially used for optimization, which can create things like jump thunks. Jump thunks are essentially small functions that only perform a jump to another function. An example would be, instead of `call function1`, the program would actuall perform a `call j_function1`. `j_function1` would simply be a function that performs a `jmp function1` instruction. This functionality will be turned off for brevity. Since our "dummy program" is so simple, it will be optimized very easily. Knowing this, we are disabling incremental linking in order to simulate a "Release" build (we are currently building "Debug" builds) of an application, where incremental linking would be disabled by default. However, none of this is really prevalent here- just a point of contention to the reader. Just know we are doing it for our purposes.

The result of the compilation command will place the output file, named `Source.exe` in this case, into the current directory along with a symbol file (.pdb). Now, we can open this application in IDA (you'll need to run IDA as an administrator, as the application is in a privileged directory). Let's take a look at the `main()` function.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFGbb.png" alt="">

Let's examine the assembly above. The above function loads `noCFG()` into RAX. RAX is then moved to `[rsp+38h+var_18]` and since `var_18` is assinged to negative 0x18, we can assume that what will actually happen here is that `noCFG()` will placed into RAX, which will be moved to `[rsp+0x20]` (a.k.a cause RSP + 0x20 to point to `noCFG()`). Eventually, a call to `[rsp+38h+var_18]` is made. So, we can make a determination that this will call `noCFG()` via a pointer. 

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG7.png" alt="">

Why does it call the pointer, instead of just performing a direct call to `noCFG()`? Remember we assigned `noCFG()` to a function pointer. This is why a call to an address which points to the function is made.

Essentially what is happening here is that the program is performing a control flow transfer to the `noCFG()` function from the `main()` function.

Nice! We know that our program will redirect execution from `main()` to `noCFG()`! Let's say as an attacker, we have an arbitrary write primitive and we were able to overwrite a pointer. Since the function `noCFG()` will be pointed to by something else (`[rsp+38h+var_18]` in this case), we know that if we were able to overwrite that pointer on the stack for instance, we could change what the final `call [rsp+38h+var_18]` actually ends up calling! This is not good from a defensive perspective.

Can we mitigate this issue? Let's go back and recompile our application with CFG this time and find out.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG8.png" alt="">

This time, we add `/guard:cf` as a flag, as well as a linking option.

Disassembling the `main()` function in IDA again, we notice things look a bit different.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG9.png" alt="">

Very interesting! Instead of making a call directly to `noCFG()` this time, it seems as though the function `__guard_disaptch_icall_fptr` will be invoked. Let's set a breakpoint in WinDbg on `main()` and see how this looks after invoking the CFG dispatch function.

After setting a breakpoint on the `main()` function, code execution hits the CFG dispatch function.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG10a.png" alt="">

The CFG disapatch function then performs a dereference and jumps to `ntdll!LdrpDispatchUserCallTarget`. 

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG11.png" alt="">

We won't get into the technical details about what happens here, as this post isn't built around CFG and Morten's blog already explains what will happen. But essentially, at a high level, this function will check the CFG bitmap for the `Source.exe` process and determine if the `Source!noCFG` function is a valid target (a.k.a if it's in the bitmap). Obviously this function hasn't been overwritten, so we should have no problems here. After stepping through the function, control flow should transfter back to the `noCFG()` function seamlessly.

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG12.png" alt="">

<img src="{{ site.url }}{{ site.baseurl }}/images/XFG13a.png" alt="">

Execution has returned back to the `noCFG()` function. Additionally what is nice, is the lack of overhead that CFG put on the program itself. The check was very quick because Microsoft opted to use a bitmap instead of indexing an array or some other structure. Let's see if we can take this even further.

CFG: Potential Shortcomings
---

As mentioned earlier, CFG checks functions to make sure they are part of the "CFG bitmap" (a.k.a protected by CFG). This means a few things from an adversarial perspective. If we were to use `VirtualAlloc()` to allocate some virtual memory, and overwrite a function pointer that is protected by CFG with the returned address of the allocation- CFG would make the program crash.

Why? `VirtualAlloc()` (for instance) would return a virtual address of something like `0xdb0000`. When the application in question was compiled with CFG, obviously this memory address wasn't a part of the application. Therefore, this address wouldn't be "protected by CFG". However, this is not very practical. Let's think about what an adversary tries to accompish with ROP.

Let's say that there is a vulnerability is `KERNELBASE.dll`. Let's also say that `KERNELBASE.dll` is compiled with CFG. This means that all of the functions within `KERNELBASE.dll` are a part of the CFG bitmap. Because CFG only validates if something is in the CFG bitmap (it doesn't technically validate if a function pointer was overwritten), it would be possible to overwrite a function pointer _WITH ANY_ other function protected by the current processes's CFG bitmap.

Let's look at a practical example of this.

Firstly, let's update our program in order to visualize this.

