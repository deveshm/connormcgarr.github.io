---
title: "Exploit Development: Between a Rock and a (Xtended-Flow) Guard Place! Examining XFG"
date:  2019-01-13
tags: [posts]
excerpt: "Taking a look at Microsoft's new forward-edge CFI solution: Xtended Flow Guard"
---
Introduction
---
I have previously [blogged](https://connormcgarr.github.io/ROP2) about ROP and the benefits of understanding how it works. Not only is it a viable first-stage payload for obtaining native code execution, it can be leveraged for things like arbitrary read/write primitives and data-only attacks. Unfortunately, if your end goal is native code execution, there is a good chance you are going to need to overwrite a function pointer in order to hijack control flow. Understanding this exploitation path, Microsoft implemented [Control FLow Guard](https://docs.microsoft.com/en-us/windows/win32/secbp/control-flow-guard), or CFG, back in Windows 8.1. Although it was released before Windows 10, it did not really catch on in terms of "main stream" exploitation until recent years.
