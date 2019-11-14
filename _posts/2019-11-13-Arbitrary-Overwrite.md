---
title:  "Exploit Development: Windows Kernel Exploitation Part 2 - Arbitrary Overwrites (Write-What-Where)"
date:   2019-11-13
tags: [posts]
excerpt: "An introduction to exploiting the ability to write a piece of data to an arbitrary location."
---
Introduction
---
In a previous post, I talked about [setting up a Windows kernel debugging environment](https://connormcgarr.github.io/Part-1-Kernel-Exploitation/). Today, I will be building on that foundation that was built within that post. Again, we will be taking a look at the [HackSysExtreme vulnerable driver](https://github.com/hacksysteam/HackSysExtremeVulnerableDriver/). The HackSysExtreme team implemented a plethora of vulnerablities here, based on the IOCTL code sent to the driver. The vulnerabilitiy we are going to take look at today is what is known as an __arbitrary overwrite__.

At a very high level what this means, is an adversary has the ability to write a piece of data (generally going to be a shellcode) to a particular location. As you may recall from my previous post, the reason why we are able to obtain local administrative privileges (NT AUTHORITY\SYSTEM) is because we have the ability to do the following:

1. Allocate a piece of memory in user land that contains our shellcode
2. Execute said shellcode from the context of ring 0 in kernel land

Since the shellcode is being executed in the context of ring 0, which runs as local administrator, the shellcode will be ran with administrative privileges. Since our shellcode will copy the NT AUTHORITY\SYSTEM token to a `cmd.exe` process- our shell will be an administrative shell.
