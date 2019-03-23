---
title:  "Exploit Development: Exception Handlers and Egg Hunters!"
date:   2019-03-22
categories: posts
tags: [posts]
excerpt: "Introduction to SEH exploits and utilizing egg hunters to achieve code execution."
---
Introduction:
---
Error handling has become a vitale element within the embodiment of software development. Far are the days from just getting programs to work. Developers need to be implement graceful exiting of a program in case of any type of error. As old adage states, "No good deed goes
unpunished." - the same applies to exception handlers. This blog post will give a brief introduction to exception handler exploits
along with how we can leverage a technique known as an egg hunter to smuggle shellcode when there is not enough room on our stack pointer.

Exception Handlers: How do they work?
---
Briefly I will hit on how these work (in Windows at least). Let's get acquainted some acronyms firstly. nSEH stands for next structured exception handler (we will get to why here in a second). SEH  (structured exception handler) is the current exception handler. There are two types of handlers- operating system and developer implemented handlers. As we know, an application executed on a machine that utilizes a stack based data structure will give each function/piece of the application its own stack frame to be put onto the stack for execution. The stack frame of each function/piece will eventually be popped of when execution has commenced. The same is true for an exception handler. If there is a function that has an exception handler embedded in itself- that exception handler will also get its own stack frame. Refer to the names of our acronyms above. Notice anything? It looks like you could have a list of exception handlers. If you thought this, then you are correct! The handlers form a data structure known as linked list. Exception handlers need to be able to point to the nSEH (next handler) and the current handler (SEH). Here is what the excpetion handlers work when they encounter an error- when an exception occurs, a majority of the registers (ESI, EAX, ECX, etc. etc.) are "zeroed out" - therefore removing the data an advesary could implement. While this sounds like a tried and true way to prevent things like stack based buffer overflows- this nomenclature is not without its flaws. We will get to how we use this newfound knowledge to obtain a shell later on. Before we move on, take one note of a crucial attribute of SEH at the time an exception is raised. SEH will be located at esp+8. This means the location of ESP plus 8 bytes will show the location of SEH.

Egg Hunters: I didn't know it was Easter yet?
---
