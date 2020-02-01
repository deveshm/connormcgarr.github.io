---
title: "Exploit Development: Panic! At The Kernel - Token Stealing Payloads Revisited on x64 and Bypassing SMEP (UNDER CONSTRUCTION)"
date:  2020-02-01
tags: [posts]
excerpt: "Revisiting token stealing payloads on Windows 10 x64 and diving into bypassing SMEP"
---
Introduction
---
Same ol' story with this blog post- I am trying to continue and further my research/overall knowledge on Windows kernel exploitation in order to prepare for [AWE](https://www.blackhat.com/us-20/training/schedule/index.html#advanced-windows-exploitation-19158), and to garner more experience with exploit development. Previously I have talked about [a](https://connormcgarr.github.io/Part-2-Kernel-Exploitation/) [couple](https://connormcgarr.github.io/Part-1-Kernel-Exploitation/) of vulnearbility classes on x86 with Windows 7, which has minimal protections
