---
title:  "Exploit Development: Exception Handlers and Egg Hunters!"
date:   2019-03-22
categories: posts
tags: [posts]
excerpt: "Introduction to SEH exploits and utilizing egg hunters to achieve code execution."
---
Introduction:
---
Error handling has become a vitale structure in the embodiment of software development. Far are the days from just getting programs to work- there needs to be the graceful exiting of a program in case of any type of error. As old adage states, "No good deed goes
unpunished." - the same applies to exception handlers. This blog post will give a brief introduction to exception handler exploits
and how we can use a technique known as an egg hunter to smuggle shellcode when there is not enough room on our stack pointer.

Exception Handlers: How do they work?
---
Briefly I will hit on how these work (in Windows at least). Let's get acquainted some acronyms firstly. nSEH stands for next structured exception handler (we will get to why here in a second). SEH  (structured exception handler) is generally the last resort exception handler implemented by the OS. There are two types of exception handlers- operating system and developer implemented handlers. As we know in an application implemented on a machine that utilizes a stack based data structure- each function/piece of the application will get its own stack frame to be put onto the stack for execution, and will eventually be popped of when execution has commenced. The same is true for an exception handler. If there is a function that has an exception handler embedded in itself- that exception handler will also get its own stack frame. Therefore, reason we have two acronyms for exception handlers is because all of the exception handlers in the various functions must keep track of each other. In order to do this, they form a data structure known as linked list. Exception handlers need to be able to point to the nSEH (next handler) and the current handler (SEH). When an exception occurs, all of the registers (EIP, ESP, ESI, EAX, ECX, EBP, etc. etc.) all are "zeroed out" - therefore removing the data an advesary could implement. While this sounds like a tried and true way to prevent things like stack basaed buffer overflows- this nomenclature is not without its flaws. We will get to how we use this newfound knowledge to obtain a shell later on.

Egg Hunters: I didn't know it was Easter yet?
---
