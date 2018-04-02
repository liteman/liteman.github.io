---
layout: post
date: 2018-04-01 16:14:00 UTC-10
title: Micro Corruption - 03 - Hanoi
---

This is level 3 of the Micro Corruption CTF. 

Note: This level highlighted an important lesson for me personally. I often make things too complicated and jump to the most sophisticated solution too quickly. I jump to conclusions. With a little more patience and just a little more time reviewing the big picture, this level would have been a much quicker solve. WORK SMART NOT HARD!!

<!--excerpt-->

On the previous two revisions of the LockIT Pro -- the password was available for extraction directly from the debugger. For a walk-through of level 2 - Sydney, click [here]({{ site.baseurl }}{% post_url 2018-03-17-Micro Corruption - 02 - Sydney %})

According to the latest update, the new lock revision b.01 employs a separate hardware security module (HSM) which will store the password separate from the lock firmware. In theory, the HSM contents will not be directly accessible from the debugger. 

![hardwareVersionB_notes]( {{ "/assets/images/han_hardwareVersionB_update.PNG" | absolute_url }} )

Scrolling through the debugger, it looks like the **main** function simply makes a call to **login**. So I will examine the **login** function to get an idea of how this lock will operate.

![login_ufnction]( {{ "/assets/images/han_login_func.PNG" | absolute_url }} )

As with previous locks, we see some preliminary **call puts** operations to display prompts to the user. Previous locks used a **get_password** function to get user input, but this lock employs calls to **getsn**. The lock manual (provided with the debugger) has this to say about the gets function:

![getsn_manual]( {{ "/assets/images/han_getsn_manual.PNG" | absolute_url }} )

The function takes two arguments - an address pointer to a buffer in memory and an integer specifying the number of bytes to read from the input device. 

Looking at the two instructions just before **call getsn**, we can see what the values for these arguments are:

```
4534: 3e40 1c00 mov	#0x1c, r14
4538: 3f40 0024 mov	#0x2400, r15
453c: b012 ce45 call    #0x45ce <getsn>
```

The function declaration, as shown in the documentation, shows the first argument is a pointer to a buffer. In the debugger, working backwards from the call operation, this argument is: **mov #0x2400, r15**. The input specified by the user will be placed in memory at address **0x2400**.

Continuing on to the second argument - the length of the input to copy - **mov #0x1c, r14**. The length is **0x1c bytes(hex)** or **28 bytes(decimal)**. This is interesting. 

According to the user prompt a few instructions earlier:
```
452c: 3f40 9e44 mov	#0x449e "Remember: passwords are between 8 and 16 characters.", r15
```

Passwords are not expected to be any larger than than _16_ characters (bytes), but the **getsn** function will read in up to _28_ bytes (only stopping if a null byte - **0x00** - is encountered in the input ). If the firmware is only expecting 16 characters, but we can input 28 there may be an opportunity for a buffer overflow.

Before I take a deep dive in debugging specifics, I want to first see what _normal_ looks like. The prompt says passwords should be between 8 and 16 characters. I will run the program and submit a string of 12 characters and observe the result. 
```
Test password: TWELVECHARS!
```

As expected, the string "**TWELVECHARS!**" is not the correct password and the lock exited normally.

![exit_screenshot]( {{"/assets/images/han_exit_screenshot.PNG" | absolute_url }})

Next, I will reset the lock and try a string of 18 characters - two more than expected.
```
Test password: EIGHTEENCHARACTERS
```
Even with two extra characters the program exits normally. Let's try the full 28 characters and see what happens.
```
Test password: TWENTYEIGHTCHARACTERPASSWORDAA
```
Even with 28 characters there was no abnormal functionality. This is where my initial investigation went off the rails. As soon as I saw buffer overflow could be an option, I had assumed the overflow would overwrite a return address somewhere which would lead to a program crash. Analyzing the crash, I thought I would be able to control execution and solve this level with custom shellcode. I spent a lot of time trying to figure out why my 28 byte input wasn't crashing the lock.

As it turns out - I was overthinking this problem by a mile.

Taking another look at the **login** function and at the **I/O console output** - I noticed something interesting. 

![io_console]( {{ "/assets/images/han_io_console.PNG" | absolute_url }} )

The string **"Testing if password is valid"** is only output to the console _after_ the call to the **test_password_valid** function. Why? 

![disassembly]( {{ "/assets/images/han_disassembly.PNG" | absolute_url }} )

Directly after the string **"Testing if password is valid"** is output to the screen, we see a **cmp.b** operation. 
```
455a:  f290 4c00 1024 cmp.b	#0x4c, &0x2410
4560:  0720     jne	#0x4570 <login+0x50>
```
This is checking if the byte found at address **0x2410** is equal to **0x4c** or capital **L** (ascii). The next operation will jump to offset **0x50** in the login function if the comparison is NOT equal (Password is not correct). 

If the result of the comparison is *equal* - we will get an **"Access Granted"** message and the **unlock_door** function will be called.
```
4562:  3f40 f144      mov	#0x44f1 "Access granted.", r15
4566:  b012 de45      call	#0x45de <puts>
456a:  b012 4844      call	#0x4448 <unlock_door>
```

User input is saved at offset **0x2400** on the stack. The software expects the password to be at most 16 characters long which would take up memory from **0x2400 - 0x240F**. The very next byte, **0x2410**, is a flag value set by Hardware Security Module (HSM) 1. If the correct password is entered, HSM 1 sets this flag to capital "L".

Since password input is stored at 0x2400, and I can input more than 16 characters, I can control what byte ends up at position **0x2410**. 

**0x2410** is *17* bytes in to the input buffer. I will use a password such that the 17th character is capital L.
```
Test password: SEVENTEENTHCHRISL
```

Success! With this password we get the satisfying door unlocked message.

![door_unlocked]( {{ "/assets/images/han_door_unlocked.PNG" | absolute_url }} )








