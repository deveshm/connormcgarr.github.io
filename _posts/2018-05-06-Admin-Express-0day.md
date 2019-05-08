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

---
The Crash
---
While perusing about for some DOS exploits, I stumbled across [this](https://www.exploit-db.com/exploits/46711). Admin Express, which is an older application, developed in 2005 and maintained until about 2008 takes about 5000 bytes to crash. This application is a type of network analyzer. Being this is an older application downloaded about 20,000 times, I bet you may find it in some older networks like operational technology (OT) environments. Due to the sentiment above about 5000 bytes being needed to crash the application, I decided I was going to proceed with the exploit development lifecycle.

---
Let's Get This Party Started!
---
