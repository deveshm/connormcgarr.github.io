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
    c_int(len(payload)),              # dwSize
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
