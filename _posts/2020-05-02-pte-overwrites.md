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

SMEP kicks in, and we can see the offending address is that of our user mode shellcode.

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_1.png" alt="">

Recall, from a [previous blog](https://connormcgarr.github.io/x64-Kernel-Shellcode-Revisited-and-SMEP-Bypass/) of mine, that SMEP kicks in whenever code that resides in current privilege level (CPL 3) of the CPU (CPL 3 code = user mode code) is executed in context of CPL 0 (kernel mode).

But _HOW_ does SMEP know to take over? Recall that SMEP is enforced in two ways. Globally, SMEP is controlled by the 20th bit of the CR4 register.

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_4.png" alt="">

The 20th bit in the above image refers to the `1` in the beginning of `0x170678`

Carrying on, as we can see, SMEP is enabled globally on the system. 

However, there is a second way SMEP is enforced- and that is on a per memory page basis, via the `U/S` PTE control bit. This is what we are going to be taking a look at in this post.

Take a look again at the Bug Check image two images ago. We can see the offending virtual address, and we can also see `Argument 3` contains the `PTE contents` value. Let's see what this looks like in WinDbg.

<img src="{{ site.url }}{{ site.baseurl }}/images/PTE_3.png" alt="">

Let's break this above image down. Our user mode shellcode was stored in an allocation at the address `0x2620000` and was configured with the following attributes (via the PTE control bits)

1. `D`- The "dirty" bit has been set, meaning a write to this address has occured (`KERNELBASE!VirtualAlloc()`)
2. `A`- The "access" bit has been set, meaning this address has been referenced at some point
3. `U`- The "user" bit has been set here. When the memory manager unit reads in this address, it recognizes is as a user mode address
4. `W`- The "write" bit has been set here, meaning this memory page is writeable
5. `E`- The "executable" bit has been set here, meaning this memory page is executable
6. `V`- The "valid" bit is set here, meaning that the PTE is a valid PTE.

Notice that most of these control bits were set with our call earlier to `KERNELBASE!VirtualProtect()` in the psuedo code snippet.

Now that we know how our shellcode allocation looks from a memory perspective, let's talk about the `U/S` bit.

As we can see, our memory page where our shellcode resides is that of a "user mode" page. However, it is possible to turn this page into a kernel mode page, and "trick" the memory manager unit into executing this page in context of the kernel!
