---
title:  "(UNDER CONSTRUCTION) Exploit Development: Leveraging Page Table Entries for Windows Kernel Exploitation"
date:   2020-05-02
tags: [posts]
excerpt: "Exploiting page table entries through arbitrary read/write primitives to circumvent SMEP, NonPagedPoolNx, and page table randomization"
---
Introduction
---

Taking the prerequisite knowledge from my [last blog post](https://connormcgarr.github.io/paging/), let's talk additional ways to bypass [SMEP](https://en.wikipedia.org/wiki/Supervisor_Mode_Access_Prevention) other than [flipping the 20th bit of the CR4 register](https://connormcgarr.github.io/x64-Kernel-Shellcode-Revisited-and-SMEP-Bypass/)- or completely avoid SMEP all together by bypassing [NX](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/no-execute-nonpaged-pool) in the kernel!
