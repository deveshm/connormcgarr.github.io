---
title:  "Exploit Development: 0day! Admin Express v1.2.5.485 Folder Path Local SEH Alphanumeric Encoded Buffer Overflow"
date:   2019-05-06
tags: [posts]
excerpt: "A 0day I found in an application called Admin Express, how to, by hand, alphanumerically encode shellcode, align the stack properly, and explaining the integral details of this exploit."
---
Introduction:
---
Get an internship at a Fortune 500 company in information security...? check! Obtain the OSCP before I graduate college...? check! Discover a vulnerability and obtain code execution...? Oh, that is depressing. These are my first three goals I have set for myself since I had the inclination to obtain a career within information security. Reviewing the elusive last item on this checklist only marred my confidence, and that did not sit well with me. After months of probing [Exploit DB](https://www.exploit-db.com) for some Denial of Service exploits, I am very happy to report that I have taken a crash of an application and revitalized it to obtain code execution! 

This blog post may get lengthy, but I hope there is some good information you can take away about what I learned about alphanumeric shellcode, encoding shellcode, aligning the stack, assembler, hexadecimal math, shellcode that directly references the kernel, and those famous SEH exploits. This writeup will assume you understand exception handler exploits. If you do not know much about those, or want a refresher, you can find my writeup [here](https://connormcgarr.github.io/Exception-Handlers-and-Egg-Hunters/). Let's get into it!
