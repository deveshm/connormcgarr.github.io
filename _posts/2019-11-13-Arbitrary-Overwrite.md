---
title:  "Exploit Development: Windows Kernel Exploitation Part 2 - Arbitrary Overwrites (Write-What-Where)"
date:   2019-11-13
tags: [posts]
excerpt: "An introduction to exploiting the ability to write a piece of data to an arbitrary location."
---
Introduction
---
In a previous post, I talked about [setting up a Windows kernel debugging environment](https://connormcgarr.github.io/Part-1-Kernel-Exploitation/). Today, I will be building on that foundation that was built within that post. Again, we will be taking a look at the [HackSysExtreme vulnerable driver](https://github.com/hacksysteam/HackSysExtremeVulnerableDriver/). The HackSysExtreme team implemented a plethora of vulnerablities here, based on the IOCTL code sent to the driver. The vulnerabilitiy we are going to take look at today is what is known as an __arbitrary overwrite__.

At a very high level what this means, is an adversary has the ability to write a piece of data (generally going to be a shellcode) to a particular location. As you may recall from my previous post, the reason why we are able to obtain local administrative privileges (__NT AUTHORITY\SYSTEM__) is because we have the ability to do the following:

1. Allocate a piece of memory in user land that contains our shellcode
2. Execute said shellcode from the context of ring 0 in kernel land

Since the shellcode is being executed in the context of ring 0, which runs as local administrator, the shellcode will be ran with administrative privileges. Since our shellcode will copy the __NT AUTHORITY\SYSTEM__ token to a `cmd.exe` process- our shell will be an administrative shell.

Code Analysis
---
First let's look at the [ArbitraryWrite.h](https://github.com/hacksysteam/HackSysExtremeVulnerableDriver/blob/master/Driver/HEVD/Windows/ArbitraryWrite.h) header file.

Take a look at the following snippet:

```c
typedef struct _WRITE_WHAT_WHERE
{
    PULONG_PTR What;
    PULONG_PTR Where;
} WRITE_WHAT_WHERE, *PWRITE_WHAT_WHERE;
```

[typedef](https://www.tutorialspoint.com/cprogramming/c_typedef.htm) in C, allows us to create our own data type. Just as `char` and `int` are data types, here we have defined our own data type.

Then, the `WRITE_WHAT_WHERE` line, is an alias that can be now used to reference the struct `_WRITE_WHAT_WHERE`. Then lastly, a aliased pointer is created called `PWRITE_WHAT_WHERE`. 

Most importantly, we have a pointer called `What` and a pointer called `Where`. Essentially now, `WRITE_WHAT_WHERE` refers to this struct containing `What` and `Where`. `PWRITE_WHAT_WHERE`, when referenced, is a pointer to this struct.

Moving on down the header file, this is presented to us:

```c
NTSTATUS
TriggerArbitraryWrite(
    _In_ PWRITE_WHAT_WHERE UserWriteWhatWhere
);
```

Now, the variable `UserWriteWhatWhere` has been attributed to the datatype `PWRITE_WHAT_WHERE`. As you can recall from above, `PWRITE_WHAT_WHERE` is a pointer to the struct that contains `What` and `Where` pointers (Which will be exploited later on). From now on `UserWriteWhatWhere` also points to the struct.

Let's move on to the source file, [ArbitraryWrite.c](https://github.com/hacksysteam/HackSysExtremeVulnerableDriver/blob/master/Driver/HEVD/Windows/ArbitraryWrite.c).

The above function, `TriggerArbitraryWrite()` is passed to the source file.

Then, the `What` and `Where` pointers decalred earlier in the struct, are initialized as __NULL__ pointers:

```c
PULONG_PTR What = NULL;
PULONG_PTR Where = NULL;
```

Then finally, we reach our vulnerability:

```c
#else
        DbgPrint("[+] Triggering Arbitrary Write\n");

        //
        // Vulnerability Note: This is a vanilla Arbitrary Memory Overwrite vulnerability
        // because the developer is writing the value pointed by 'What' to memory location
        // pointed by 'Where' without properly validating if the values pointed by 'Where'
        // and 'What' resides in User mode
        //

        *(Where) = *(What);
```

As you can see, an adversary could write the value pointed by `What` to the memory location referenced by `Where`. The real issue is that there is no validation using a Windows API function such as `ProbeForRead()` or `ProbeForWrite()`, that validate whether or not the values of `What` and `Where` reside in user land. Knowing this, we will be able to utilize our user land shellcode going forward for the exploit.

IOCTL
---
