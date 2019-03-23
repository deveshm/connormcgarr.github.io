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
Briefly I will hit on how these work (in Windows at least). Let's get acquainted some acronyms firstly. nSEH stands for next structured exception handler (we will get to why here in a second). SEH  (structured exception handler) is the current exception handler. There are two types of handlers- operating system and developer implemented handlers. As we know, an application executed on a machine that utilizes a stack based data structure will give each function/piece of the application its own stack frame to be put onto the stack for execution. The stack frame of each function/piece will eventually be popped of when execution has commenced. The same is true for an exception handler. If there is a function that has an exception handler embedded in itself- that exception handler will also get its own stack frame. Refer to the names of our acronyms above. Notice anything? It looks like you could have a list of exception handlers. If you thought this, then you are correct! The handlers form a data structure known as linked list. Exception handlers need to be able to point to the nSEH (next handler) and the current handler (SEH). Here is what the excpetion handlers work when they encounter an error- when an exception occurs, a majority of the registers (ESI, EAX, ECX, etc. etc.) are "zeroed out" - therefore removing the data an advesary could implement. While this sounds like a tried and true way to prevent things like stack based buffer overflows- this nomenclature is not without its flaws. We will get to how we use this newfound knowledge to obtain a shell later on. Before we move on, take one note of a crucial attribute of SEH at the time an exception is raised. SEH will be located at `esp+8`. This means the location of ESP plus 8 bytes will show the location of SEH.

Egg Hunters: I didn't think Easter was here yet?
---
Let me preface this paragraph by saying egg hunters are not making an inference to the Easter bunny. Egg hunters are a small amount of code generated (around 32 bytes in most cases) that have a string embedded in them. The word that has gained the most notoriety is the tag `w00t`. This term, at a high level, is embedded in a small piece of shellcode. This would be implemented in a place where you could not smuggle a large payload, like an encoded reverse shell- which would be around 350-400 bytes. This small piece of shellcode, essentially, will recursively search for your large second stage payload (a reverse shell). You should firstly append two instances of your egg to the beginning of your second stage payload. `w00tw00t` would be the term you should implement prior to your string of large shellcode, like a reverse shell. Egg hunters are a very unique and cool way to jump to shellcode. Here is a good white paper on egg hunters: [Egg Hunters](https://www.exploit-db.com/docs/english/18482-egg-hunter---a-twist-in-buffer-overflow.pdf)Let's move on :)

Down to the Nitty-Gritty
---
We will be taking crack at 
