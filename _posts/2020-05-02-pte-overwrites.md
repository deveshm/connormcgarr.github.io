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

Note, the above code is syntactically incorrect, but it is there nonetheless to help us understand what is going on.

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
4. Clears the PTE control bit for `U/S` by subtracting `4` from the PTE control bit contents. (Note, I found this to be the correct value after trial and error).

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
