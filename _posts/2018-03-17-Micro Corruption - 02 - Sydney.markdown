---
layout: post
date: 2018-03-17 10:26:00 UTC-10
title: Micro Corruption - 02 - Sydney
---


The pop-up documentation for this new lock in Sydney - specifies that the firmware has been revised and will no longer contain the password in memory. 

![release_notes_excerpt]({{"/assets/images/syd_revision_text.PNG" | absolute_url }})

<!--excerpt-->

For a walk through of the previous revision of this lock, click [here]({{ site.baseurl }}{% post_url 2018-02-26-Micro Corruption Walkthrough (A Series) %})

I start by scrolling through the disassembler to review the main function and see the basic flow of the program. 

![img_main_func]({{"/assets/images/syd_main_func.PNG" | absolute_url }})

Starting at offset **0x443c** see a **mov** instruction followed by a **call** to **puts()** - prompting the user to enter a password. I then see a **get\_password()** function which will read the users input. Next, the **check_password()** function will be called. 

I'll take a closer look at **check_password** to see how it has changed.

![img_checkpass_func]({{"/assets/images/syd_check_password_func.PNG" | absolute_url }})

In the **check_password** function I see a series of **cmp** operations. 


    448a: cmp #0x3d5d, 0x0(r15)
    ...
    4492: cmp #0x5f22, 0x2(r15)
    ...
    449a: cmp #0x7d78, 0x4(r15)
    ...
    44a4: cmp #0x3a47, 0x6(r15)


After each **cmp** there is a **jnz** (jump not zero) to exit the function and set the return value to 0 if the compare failed.

```php
    
    4490: jnz $+0x1c  # If not equal, jump 0x1c (28) bytes forward to 44ac
    
    44ac: clr r14  # set r14 to zero
    44ae: mov r14, r15  # set r15 as the return value
    44b0: ret  # return to main
```
If I look closer at the **cmp** operations, it looks like each 2 byte literal value is compared with the bytes found at some offset of the address stored in r15. _0x0(r15)_ - the two bytes at the start of r15
_0x2(r15)_ -- the two bytes starting at offset 0x2 of r15
_0x4(r15)_ -- the two bytes starting at offset 0x4 of r15

If I string the two-byte literal values from the instructions together I get:

**3D5D5F227D783A47**

I could convert these bytes to ASCII characters, but since the micro corruption input dialog will accept hex values I can enter this string of bytes directly to test.

I enter the 'continue' command in the debugger console to start program execution. When execution reaches the **get_password** function, I am prompted for input:

Testing the hex string:

![img_input_dialoag]({{"/assets/images/syd_input_dialog.PNG" | absolute_url }})

Since the debugger automatically breaks/pauses after user-input, I must enter 'continue' in the console again. Program execution completes but I do _not_ get a "solved" notification. Interesting...

![img_not_solved]({{"/assets/images/syd_end_no_solution.PNG" | absolute_url }})

I will run through again, this time with a breakpoint at the beginning of the **check_password** function **0x448a**. 

![img_breakpoint]({{"/assets/images/syd_breakpoint.PNG" | absolute_url }})

After setting the break point, I enter 'reset' in the Debugger Console and allow program execution until the break point is reached.

With the debugger paused at offset **0x448a**, I check the Live Memory Dump -- specifically at the address stored in **r15** -- **0x439c.**

![img_mem_dump]({{"/assets/images/syd_memdump.PNG" | absolute_url }})

The byte sequence found at **0x439c** is our input from the password prompt - unchanged. Stepping through the function ('step' on the console) shows that the very first **cmp** operation fails. Why? 

The answer: _endianness_. It turns out the MSP430 micro-controller this lock runs on uses little endian architecture. This means the byte at address **0x439c** ( 0x0(r15) ) is the least significant byte and the next byte over at **0x439d** ( 0x1(r15) ) is the most significant byte.

In other words, bytes 0x3da in the Live Memory Dump will be read as **0x5d3d** in the **cmp** operation. With this in mind, I will re-work the bytes I collected earlier to account for the little endian architecture.

    Original:      3D5D5F227D783A47
    Little-Endian: 5D3D225F787D473A

I will reset the lock and run through the prompts again. 

This time, with the new byte sequence, I get the satisfying problem-solved notification.

![img_solved]({{"/assets/images/syd_solved.PNG" | absolute_url }})







