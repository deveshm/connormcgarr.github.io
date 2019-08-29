---
title:  "Cracking The Perimeter and OSCE Thoughts"
date:   2019-08-24
tags: [posts]
excerpt: "My thoughts on the Cracking The Perimeter course/OSCE Exam and how I came to learn that one must learn to walk before learning to run."
---
Introduction
---

<img src="{{ site.url }}{{ site.baseurl }}/images/offsec-student-certified-emblem-rgb-osce.png" alt="">

"Can you please update the course materials?" "I'll take this course when the materials are updated!" These are some of the retorts I frequently see, in response to words of commendation and praise that the Offensive Security community attribute to the [_Cracking The Perimeter_](https://www.offensive-security.com/information-security-training/cracking-the-perimeter/) course and [OSCE](https://www.offensive-security.com/information-security-certifications/osce-offensive-security-certified-expert/) exam.

_Cracking The Perimeter_, stylized as _CTP_, is the accompanying course to the Offensive Security Certificed Expert (OSCE) certification. This course is often seen as "outdated". There is a reason why Offensive Security certifications do not have an expiration date. There is a reason why the courses receive strategic updates. There are reasons why Offensive Security alumni are frequently sought out in the information security market. I will hit on all of these notions in the middle and latter parts of this post. Always remember to trust the process, when it comes to Offensive Security. There is a reason why they have accrued so much noteriety.

What are the prerequisites to this course?
---

Are you an [OSCP](https://www.offensive-security.com/information-security-certifications/oscp-offensive-security-certified-professional/) alumni? Did you take joy in the exploit development segment of the [_PWK_](https://www.offensive-security.com/information-security-training/penetration-testing-training-kali-linux/) course? Do you want to demistify the "magic" behind binary exploitation? Do you just have an overall affection for the exploit development lifecycle, x86 assembly, web applications, and infrastructure exploitation?

If you answered "yes" to any of these questions, this course and exam are probably for you. One thing to note as well, there is a common misconception that one must have the OSCP and/or have completed the PWK course. This is not true. Although beneficial, it is not necessary.

Here are the recommended [prerequisites](https://www.offensive-security.com/ctp-syllabus/#pre-req) to the course.

As stated by Offensive Security, there does need to be a slight tolerance for pain and suffering. They are referring to the fact that indeed, you will be stuck from time to time. Sometimes it may feel like there is no end in sight. Never give up when you feel this way. Read/review until you understand a concept. Utilize your student forums, too! Your fellow peers won't let you go through it alone.

I would recommend dabbling in and being familiar with CPU registers within the x86 (32 bit) architecture, down to the 8 bit reigsters within them. This [article](https://en.wikibooks.org/wiki/X86_Assembly/X86_Architecture) is a good place to start.

I would, in addition, recommend getting intimate with either [Immunity Debugger](https://debugger.immunityinc.com/ID_register.py) or [OllyDbg](http://www.ollydbg.de/download.htm).

What's the content of the course like?
---

The content of the course is heavily focused on exploit development and assembly, after the first two modules.

The full syllabus can be found [here](https://www.offensive-security.com/documentation/cracking-the-perimeter-syllabus.pdf).

After the first two modules, the course focuses on things like: signature antivirus bypassing, backdooring portable executables, fuzzing, [egghunters](https://connormcgarr.github.io/Exception-Handlers-and-Egg-Hunters/), [structured exception handler (SEH) exploits](https://connormcgarr.github.io/Exception-Handlers-and-Egg-Hunters/), alphanumeric shellcode, and Cisco exploitation.

For me, one of the two modules before any assembly/exploit development, that involved manually finding [Local File Inclusion (LFI)](https://www.owasp.org/index.php/Testing_for_Local_File_Inclusion) vulnerabilities and chaining them with other attack vectors to obtain remote code exeuction, was eye opening.

I most enjoyed the modules about the exploit development cycle. This included: fuzzing to identify vulnerabilities, creating POCs, making precise calculations, defeating constrained buffer space, defeating ASLR on Windows Vista, and adhering to specific alphanumeric constraints.

The content of the course really opens your eyes to what goes on under the hood.

The exam?
---

I will try not to discourage any readers I may have, but this exam was brutal. I have a short lived information securiry career at the time of this post, but up unil right now (the time this post was written), it was easily the hardest thing I have ever done.

It is, however possible.

Although I won't be able to hit on any specifics, there are a small number of objectives that need to be satisfied within the 48 hour alloted time slot. The total amount of points from the objectives is 90, and the successful examinee will be able to acquire 75 of those 90 points.

As far as advice goes, never give up.

There is a reason there is an accompanying course to the exam. Take inspiration from the course, and any research you may have expanded on throughout the course. Think laterally, creatively, and with a purpose.

A few questions I always reiterate to myself when I don't know where else to turn are: "What am I trying to accomplish? What do I already know? How can I expand on what I know? What kinds of quesitons should I ask to accomplish this goal?"

tHiS cOuRsE iS sO oUtDaTeD oMg oFfSeC tEaCh mE sOmEtHiNg rElEvAnT
---
