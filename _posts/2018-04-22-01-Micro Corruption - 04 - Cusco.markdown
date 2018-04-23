---
layout: post
date: 2018-04-22 16:04:00 UTC-10
title: Micro Corruption - 04 - Cusco
---

We are now on Lockitall LockIT Pro, revision b.02

What changed in this edition?

"We have fixed issues with passwords which may be too long"

![overview-screenshot]( {{ "/assets/images/cus_overview_screenshot.PNG" | absolute_url }} )

<!--excerpt-->

Click [here]({{ site.baseurl }}{% post_url 2018-04-01-01-Micro Corruption - 03 - Hanoi %}) to read about cracking Level 3 - Hanoi.

### Level 4 - Cusco
In reviewing the **login** function in the Disassembly window, I can see that the **cmp.b** operation from revision b.01 has been removed. And, even though the overview mentions _corrected_ issues with passwords which may be too long, I can immediately see this has not been fixed at all. 

At **0x4514** the length parameter for this **getsn** function is passed to r14 (**mov #0x30, r14**. The user-prompt text still says passwords are between 8 and 16 characters ( at **0x450c**) but the **getsn** function will read in _48_ bytes (0x30 hex).  

Another interesting fact about this **getsn** function call is the buffer location.  At **0x4518**, the instruction **mov sp, r15** moves the current Stack Pointer to r15 to be used as the input buffer.

Putting user input directly on the stack without sanitization and length checks in place is very dangerous. This looks like a valid attack vector, so I will investigate further. 

I set a breakpoint at **0x451a** - the call to **getsn**. Once this breakpoint is reached, I want to look at the value of the **sp** register and look in the Live Memory Dump at that location.

![registers-breakpoint]( {{ "/assets/images/cus_registers_breakpoint.PNG" | absolute_url }})

When I hit the breakpoint, the **sp** register value is **0x43ee**. The Live Memory Dump window confirms this by helpfully pointing out the location of the current stack pointer. 

![livemem-view]( {{ "/assets/images/cus_livemem_screenshot.PNG" | absolute_url }})
 
Starting with **0x43ee** we see a series of 0x00 bytes. This is the buffer in which the user input will be stored. There are enough 0x00 bytes to handle a 16 byte string. Anything more than 16 bytes will start to overwrite existing values in memory. The first two bytes that would be overwritten are - **3c 44** at **0x43fe**.

![memoverwrite]( {{ "/assets/images/cus_memoverwrite_before.PNG" | absolute_url }})

We know that the MSP430 is little-endian architecture. If we reverse those two bytes to **0x443c**, we can see that this value looks pretty close to our location in memory. 

Looking through the disassembly window for instructions at or near **0x443c**, we eventually find:  
```
4438:  b012 0045      call	#0x4500 <login>
443c <__stop_progExec__>
```  
And we can see that the instruction just before 443c is the original call to the login function. This is important.

When a **call** instruction is executed, the memory address immediately after the call is pushed to the stack. This value is the _return address_ of the called function. In other words, when the called function is done executing, the return address will be read from the stack and execution will continue at the designated address.

The bytes we identified in the Live Memory Dump represent the return address for the login function. If we can overwrite those bytes with a return address of our choosing then we can control this program. The question now becomes, which address do we want? 

Let's take another look at the login function.

![login_func]( {{ "/assets/images/cus_login_func.PNG" | absolute_url }})

We can see that if the **test\_password\_valid** at **0x4520** function returns **1** then the **unlock\_door** function will be called. Since we don't know the password, we have no hope of this happening; however, we _can_ see that the address of the **unlock\_door** function is **#0x4446**. If we cause the login function to _return_ to the **unlock\_door** function we'll be golden.

So we know the stack allows for 16 bytes of user input and that bytes _17_ and _18_ will overwrite the existing return address. We need to put together a "password" which has the correct values at bytes 17 and 18.

We can do this manually by typing 16 hex values plus the return address (keeping little-endian in mind)
```
414141414141414141414141414141414644
```  
This is fine for small strings and a single memory address; but, if we happen to need longer strings or more complicated byte calculations in the future, having a programmatic solution will be useful.

Convert using python:

```
{{ syntax highlight python }}
>>> import struct
>>> ('A' * 16 + struct.pack('<H',0x4446)).encode('hex')

# Output: '414141414141414141414141414141414644'
{{ syntax end }}
```

The [struct](https://docs.python.org/2/library/struct.html) library makes handling binary data much easier - especially when we need to be concerned with endianness (MSP430 uses little-endian byte order).

Breaking down the python code above:

**'A' * 16** --> create a string which repeats the character 'A' 16 times

The **'+'** symbol tells python to append what comes next to the previously created string of 'A' characters.

**struct.pack(\'<H\', 0x4446)** --> The **<** tells struct to use little-endian byte order. The **\'H\'** tells struct that the data-type of the supplied parameter will be **unsigned short (integer)**. **0x4446** is the memory address that we need to pack in to our string.

**().encode('hex')** --> converts the final string in to hex bytes.

**Note:** The actual bytes in use here - **0x4446** - happen to correspond to ASCII characters. So, we could just as easily submit the password in ASCII format as well. The **encode('hex')** is not strictly necessary in this case.  
```
414141414141414141414141414141414644  (hex)
or
AAAAAAAAAAAAAAAAFD  (ascii)
```

Using the hex:

![usinghex]( {{ "/assets/images/cus_usinghex.PNG" | absolute_url }})

Problem Solved!

![solved]( {{ "/assets/images/cus_solved.PNG" | absolute_url }})


