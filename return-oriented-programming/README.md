
# Background Knowledege

## Stack layout for C programs

Under the [cdecl calling convention](https://en.wikipedia.org/wiki/X86_calling_conventions#cdecl) the stack can generally be though of as a series of frames. Each function has its own frame and roughly follows the format below:

```text
==========================
|         arg 0          |
==========================
|         arg 1          |
========================== 
|    return address      |
========================== 
|  previous base pointer |
========================== <-- base pointer
|      local var 1       |
==========================
|      local var 2       |
========================== <-- stack pointer
```

 - **arg0 and arg1** are arguments passed to the current function
 - **return address** is the address of the instruction that should be executed *after* this function returns
 - **previous base pointer** holds the value of the previous base pointer to be restored after exiting this function
 - **local var 1 and local var 2** are variables declared locally within this function
 - **base pointer** points to the base of the current stack frame
 - **stack pointer** points to the top of the stack

The return address, previous base pointer, and arguments passed to the current function are all pushed to the stack by the caller function and so belong to the caller function's stack frame. 

The current function's frame starts at **base pointer** and ends at **stack pointer**. Only **local var 1** and **local var 2** are within the current function's stack frame. If the current function were to call another function first the arguments would be pushed onto the stack (in reverse order), next the current instruction pointer would be pushed, and lastly the base pointer would be pushed to the stack before calling the next function.

## Classic Buffer Overflow Attack

In this attack the attacker overflows a buffer on the stack to place some malicious code onto the stack while also overwriting the return address to point to the malicious code.

Take the example function:
```C
void vulnerable(char* buffer) {
    char local_buff[5];
    strcpy(local_buff, buffer);
}
```

The stack for this function will look something like:

```text
========================== 
|    return address      |
========================== 
|  previous base pointer |
========================== 
|                        |
|     5 char buffer      |
|                        |
========================== 
```


If the **buffer** string that is passed to the vulnerable function is more than 5 characters long, **local_buff** will be overflowed.

An attack can use this to their advantage to make the stack look something like the following:

```text
========================== 
|                        |
|     malicious code     |
|                        |
========================== <-- 0xffeacb4e9
|      0xffeacb4e9       | (return address)
========================== 
|    garbage values      | (previous base pointer)
========================== 
|                        |
|    garbage values      | (5 char buffer)
|                        |
========================== 
```

By overflowing the **local_buff** an attacker is able to insert malicious code onto the stack and overwrite the return address to point to the malicious code.

When the vulnerable function returns it will jump to the location fo the malicious code at **0xffeacb4e9** instead of the code that was intended to be executed.

### Additional Reading

- [Smashing The Stack For Fun And Proft][2]
- [Buffer Overflow Seed Labs Exercise][3]

[2]: http://cecs.wright.edu/people/faculty/tkprasad/courses/cs781/alephOne.html
[3]: http://www.cis.syr.edu/~wedu/seed/Labs_16.04/Software/Buffer_Overflow/

## Return to libc

An easy and popular defense that defeats the classic buffer overflow is to disallow executing arbitrary data as code.

Using this technique, all parts of a binary are explicitly declared as either code or data.
 
 - Code can be executed but not modified
 - Data can be modified but not executed

The classic attack relies on placing malicious code on the stack and executing it. Disallowing execution on the stack makes the previous attack impossible.

Attackers in turn found a way to gain control of a program using only code already marked as being executable. In this case, the attackers use functions present in the shared library **[libc](https://en.wikipedia.org/wiki/C_standard_library)** that is loaded with almost every program in linux.

The attacker must now manipulate the stack to mimic a call to a function in **libc**. One popular example is to make the following function call:

```C
system("/bin/sh");
```

If done in a process that runs as the root user the attacker would gain a root shell.

For this attack to work the attack must manipulate the stack in the following way

```text
    Before Overflow                 After Overflow

                               ==========================
                               |  pointer to "/bin/sh"  |
                               ==========================
                               |    address of exit()   |
==========================     ========================== 
|    return address      |     |   address of system()  |
==========================     ========================== 
|  previous base pointer |     |                        |
==========================     ========================== 
|                        |     |                        |
|     5 char buffer      |     |                        |
|                        |     |                        |
==========================     ========================== 
```

The return address is modified to point to the **system()** function in libc. 

The next value is set to point to the address of the **exit()** function in libc. This value is where the **system()** function will return to. It is not necessary to point it to anything in particular. If not explicitly set, the program will likely attempt to return to an invalid area and crash. By setting the return to **exit()** the program will silently close instead of crashing.

The final value points to the string argument to be passed to the **system()** function. The simplest method of getting a pointer that points to a string "/bin/sh" is to set an environment variable containing the string "/bin/sh". Environment variables sit at the very top of the address space and are present in every program.

An attacker can write their own program to find the addresses of the libc functions and the "/bin/sh" environment variable. Once the addresses are known they can be hard coded into an exploit that makes use of a buffer overflow exploit using the method shown above.

This method assumes that the attacker is able to know the address of the libc functions an environment variables in the vulnerable program. In modern computers this is not the case. [Address Space Layout Randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization) (ASLR) offsets various parts of memory by a random amount each time a program is run. With ASLR enabled an attacker cannot guess the addresses of libc functions and environment variables because they will be different each time a program is run.

### Additional Reading

 - [Return-to-Libc Seed Lab Exercise][4]

[4]: http://www.cis.syr.edu/~wedu/seed/Labs_16.04/Software/Return_to_Libc/


# Return Oriented Programming

This attack relies on chaining together small segments of code already present in the binary to execute arbitrary code. This attack does not rely on inserting malicious code onto the stack, will still work even if shared libraries remove functions capable of being used maliciously, and in some cases can be used ot defeat ASLR.

## Gadgets

One of the key ideas behind ROP is the ability to find small segments of code in the binary ending in a **ret** instruction. These segments, called *gadgets* can be strung together to form complex programs. 

Gadgets can be executed one after another by placing the address of each gadget on the stack.

```text
==========================
|         ....           |
==========================
|       gadget 3         |
========================== 
|       gadget 2         |
========================== 
|       gadget 1         | (original return address)
==========================
```

Because each gadget must always end in a **ret** instruction, after **gadget 1** executes the program will return and begin to execute **gadget 2** which will return and execute **gadget 3** and so on.

Provided there are a sufficient number of gadgets present in a binary, this method can be used to execute arbitrary code. The next step is to show that there are indeed a large number of gadgets present in a non-trivial binary.

### Finding gadgets

A binary will have a large number of gadgets independent of the number **ret** instructions present in the program. Additional gadgets can be found by looking for the bytes corresponding to a **ret** instruction within other instructions.

In Hovav Schacham's [paper][5], he uses the english language as an example. The word "dress" can be found in the word "Ad**dress**". The word "head" can be found in the phrase "T**he ad**dress". The same method can be used to find gadgets in a binary program.

In architectures with non-fixed sized instructions such as x86, even the same **ret** instruction can provide multiple gadgets. Another example from Hovav Schacham's [paper][5], take the following set of instructions

```text
f7 c7 07 00 00 00        test $0x00000007, %edx
0f 95 45 c3              setnzb -61(%ebp)
```

If interpreted without the **f7** byte, the set of instructions becomes

```text
c7 07 00 00 00 0f        movl $0x0f000000, (%edi)
95                       xchg %ebp, %eax
45                       inc  %ebp
c3                       ret
```

Even just offsetting the start point can create a new gadget

So, to find all gadgets in a binary one simply needs to scan for all bytes corresponding to a **ret** instruction (**c3** on x86) and decode backwards to obtain gadgets.

As an example, take the following random byte sequence ending in **c3**:

```text
14 4b 3b d7 c3
```

Starting from **c3** and decoding backwards, all possible gadgets from this sequence of bytes are:

```text
d7         xlat BYTE PTR ds:[ebx]
c3         ret

3b d7      cmp edx, edi
c3         ret

4b         dec ebx
3b d7      cmp edx, edi
c3         ret

14 4b      adc al, 0x4b
3b d7      cmp edx, edi
c3         ret
```

Even a random 5 byte sequence ending in **c3** corresponds to 4 potentially useful gadgets.


### Using gadgets

Once a series of useful gadgets have been found they can be chained together to form a program.

This is done by pushing the address of each gadget onto the stack. When one gadget ends by calling **ret** the next gadget's address will be popped off the stack and used as the location of the next instruction to be executed.

As an example, say you found the following gadgets in a binary

```text
  Address  | Label |  Instruction
------------------------------------
0x08056c2c |   G0  |  pop edx; ret
0x080c5126 |   G1  |  pop eax; ret
0x0808fe0d |   G2  |  move dword ptr [edx], eax; ret
```

You could use the above gadgets to load a string into a data segment 4 bytes at a time

```text
0x080f4060 (@ .data)  # Address of some writable data segment



======Initial Stack=======
|       0x0808fe0d       | (G2)
==========================
|         '/bin'         |
==========================
|       0x080c5126       | (G1)
========================== 
|       0x080f4060       | (@ .data)
========================== 
|       0x08056c2c       | (G0)
========================== 
```

Remember, the **eip** register holds the next instruction to be executed and **ret** is essentially equivalent to **pop eip**.

After the target function returns, the address of **G0** is loaded into the program counter register (%eip)and removed from the stack. The stack now looks like this:

```text
==========================                         eip: 0x08056c2c (G0)
|       0x0808fe0d       |
==========================
|         '/bin'         |
==========================
|       0x080c5126       | (G1)                    ---Next Instructions----
==========================                         pop edx
|       0x080f4060       | (@ .data)               ret
========================== 
```

The two instructions in gadget G0 will pop the address of the data section into **edx** and **ret** will pop the address of the G1 gadget into **eip**. After executing G0 the stack will look like this:

```text
                                                  eip: 0x080c5126 (G1)
                                                  edx: 0x080f4060 (@ .data)


==========================
|       0x0808fe0d       | (G2)                    ---Next Instructions----
==========================                         pop eax
|         '/bin'         |                         ret
========================== 
```

The two instructions in gadget G1 will execute next. The first instruction pops the 4 byte string '/bin' into **eax** and is followed by **ret** which pops the address of **G2** into **eip**.

```text
                                                  eip: 0x0808fe0d (G2)
                                                  edx: 0x080f4060 (@ .data)
                                                  eax: '/bin'

                                                  ---Next Instructions----
                                                  move dword ptr [edx], eax
                                                  ret
==========================
```

The next instruction loads the value stored in **eax** ('/bin') into the location pointed to by **edx** (.data). This stores the string "/bin" in the binary starting at location .data.

This process could be repeated to store add more characters to the string stored in .data. The following ROP chain will cause add the characters '/sh;' to the string stored in .data making the full string "/bin/sh;".

```text
==========================
|       0x0808fe0d       | (G2)
==========================
|         '/sh;'         |
==========================
|       0x080c5126       | (G1)
========================== 
|       0x080f4064       | (@ .data+4)
========================== 
|       0x08056c2c       | (G0)
========================== 
```

This ROP chain is nearly identical to the previous. The only differences are the string being stored is now '/sh;' and the destination address is now at .data+4 because the first four bytes of .data already hold "/bin".

### Additional Reading

- [The Geometry of Innocent Flesh on the Bone:
Return-into-libc without Function Calls (on the x86)][5]
- [When Good Instructions Go Bad:
Generalizing Return-Oriented Programming to RISC][6]
- [ROPgadget tool][7]
- [x86 cheat sheet][8]

[5]: https://cseweb.ucsd.edu/~hovav/dist/geometry.pdf
[6]: https://cseweb.ucsd.edu/~hovav/dist/sparc.pdf
[7]: https://github.com/JonathanSalwan/ROPgadget
[8]: http://pages.cs.wisc.edu/~remzi/Classes/354/Fall2012/Handouts/Handout-x86-cheat-sheet.pdf

## Defending against ROP

=== To be continued Friday ===
