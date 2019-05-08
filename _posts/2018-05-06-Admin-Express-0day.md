---
title:  "Exploit Development: 0day! Admin Express v1.2.5.485 Folder Path Local SEH Alphanumeric Encoded Buffer Overflow"
date:   2019-05-06
tags: [posts]
excerpt: "A 0day I found in an application called Admin Express, how to, by hand, alphanumerically encode shellcode, align the stack properly, and explaining the integral details of this exploit."
---
Introduction
---
Get an internship at a Fortune 500 company in information security...? check! Obtain the OSCP before I graduate college...? check! Discover a vulnerability and obtain code execution...? Oh, that is depressing. These are the first three goals I set for myself, since I've had the inclination to obtain a career within information security. Reviewing the elusive last item on this checklist only marred my confidence, and that did not sit well with me. After months of probing [Exploit DB](https://www.exploit-db.com) for some Denial of Service exploits, I am very happy to report that I have taken a crash of an application and revitalized it to obtain code execution! 

This blog post may get lengthy, but I hope there is some good information you can take away about what I learned about the following domains: alphanumeric shellcode, encoding shellcode, aligning the stack, assembler, hexadecimal math, shellcode that directly references the kernel, and those famous SEH exploits. This writeup will assume you understand exception handler exploits. If you do not know much about those, or want a refresher, you can find my writeup [here](https://connormcgarr.github.io/Exception-Handlers-and-Egg-Hunters/). Let's get into it!

The Crash
---
While perusing about for some DOS exploits, I stumbled across [this](https://www.exploit-db.com/exploits/46711). Admin Express, which is an older application developed in 2005 and maintained until about 2008, takes about 5000 bytes to crash. Admin Express functions as a type of network analyzer and asset management program via network enumeration. Being that this is an older application, which has about 20,000 downloads, I bet you may find it in some older networks like operational technology (OT) environments. Due to the sentiment above about 5000 bytes being needed to crash the application, I decided I was going to proceed with the exploit development lifecycle. This exploit can be completed on any Windows machine that does not default to using [ASLR and DEP](https://security.stackexchange.com/questions/18556/how-do-aslr-and-dep-work). [Bypassing ASLR and DEP](https://www.exploit-db.com/docs/english/17914-bypassing-aslrdep.pdf) is not within the scope of this post.

Let's Get This Party Started!
---
The vulnerability in this application arises from the __Folder Path__ field within the __System Compare__ configuration tab. [Open the application](https://admin-express.en.softonic.com/download), attach it to [Immunity Debugger](https://www.immunityinc.com/products/debugger/), and run it via Immunity.

Going forward, we will be utilizing this proof of concept (PoC) Python script that will aid us in the exploit development lifecycle.



Here are the next steps to get started:
* 1. After starting the application in Immunity, click on the __System Compare__  tab in Admin Express.
