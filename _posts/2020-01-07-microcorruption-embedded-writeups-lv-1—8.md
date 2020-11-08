The Microcorruption Embedded CTF platform provides challenges that require you to understand security issues in custom electronic lock firmware, emulated on a `Texas Instruments MSP430` microcontroller.

You are provided with the disassembly of the firmware, a debugger, run-time stack inspector, controller register-inspector and a manual containing documentation on the instruction set and interrupt vector tables for the TI-MSC430, HSM-1 and HSM-2.

*Levels are written up in reverse order*


## Level 8: Montevideo
*Requires knowledge of memory overflow/overwrite - see Level 7 write-up*

The release notes state that the software has been revamped and now follows ‘better security practices’. However, the fact that strcpy and memset are introduced in the firmware likely says otherwise.

The login routine calls getsn which takes in user input, which is stored at `0x2400`. This address is loaded into r14, and address 0x43ec is loaded into `r15`. `strcpy` is then called, which copies input from the former to the latter address.

`0x2400` is loaded into `r15` and `memset` is invoked. This clears out the original input in memory.

`conditional_unlock_door` is invoked, the same way that it was in the previous lock.

Fuzzing the input with a series of characters — `AABBCCDDEEFFGGHHIIJJKK`, and continuing execution, we get an `insn address misaligned` error. On examining the stack pointer/program counter registers, we see the following: 

We are able to overwrite `4949` into `pc`:

![mc_8_1.png]({{site.baseurl}}/assets/img/mc_8_1.png)

The code stores the next instruction to be executed after a 16 byte input buffer:

![mc_8_2.png]({{site.baseurl}}/assets/img/mc_8_2.png)

This looks very similar to the previous lock. 16 bytes of input, followed by 2 bytes for the program counter. We can use this to redirect code execution. Once again, there is no routine we can jump to within the code, but we can compile our own shellcode, enter it as input and redirect execution. Let’s use the same input as earlier, overflowing the `pc` with random values for now:

`30127f00b012324541414141424242424343`

Upon entering this, we’ll break right after `strcpy` and examine memory to make sure our input was copied.

Our input is at the top: `0x2400`, copied input at `0x43ee`, starting at the `stack pointer marker`:

![mc_8_3.png]({{site.baseurl}}/assets/img/mc_8_3.png)

Our input made it to `0x2400`, however, only the first 3 bytes got copied over to `0x43ee`? This is because of the implementation of `strcpy` — it copies until it sees a null byte.

The assembly of the instruction itself contains a null byte.

```
push #0x7f         ->     30127f00 ; null byte at the end - 007f
call #0x454c       ->     b0124c45
```

### How do we get around this?
Brute-forcing the addresses might work, but let's take a look at the logic behind it. We require `0x7f` on the stack ie., decimal `127`.

*I tried ending up at this value by using the BIC instruction to clear target bits in a memory location/register, however, still required at least one 0x00. From the manual: BIC arg1 arg2 → arg2 &= arg1 (clear the bits in arg1 from arg2)*

A simpler approach is to use registers to carry out instructions that result in this number and then push that register onto the stack. However, none of those can have null bytes in them either. Below is some simple Python to print hexadecimal numbers, `0x7f` apart. Pick a pair that has no null bytes in it.

```python
for i in range(10):
    print(‘0x{:04x}’.format(127*(i+1)), ‘0x{:04x}’.format(127*i))
```

Program output:

![mc_8_4.png]({{site.baseurl}}/assets/img/mc_8_4.png)

Using the pair `0x01fc` and `0x017d`, assemble the following code:

```
mov #0x01fc, r12
sub #0x017d, r12 ; essentially, 508 - 381 = 127 = 0x7f
push r12
call #0x454c
```

The above amounts to 14 bytes. We now need 2 filler bytes to complete our 16-byte user input buffer followed by 2 bytes to redirect execution to the start of this input. ( remember, we have to enter the bytes in reverse order due to little endianness ).

```
Input: 3c40fef03c807ff00c12b0124c45 4444 ee43
```

Upon entering the above, the ‘incorrect password entered’ path would be taken as the HSM2 would return an invalid signal, and then execution would be redirected to the above, calling interrupt `0x7f` and unlocking the lock.

## Level 7: Whitehorse

Following suit with Lock 6, writing more than 16 bytes (8 ASCII characters) causes stack corruption. Conveniently, the return address is stored on the stack right after user input and can be overwritten to control execution.

### Where do we redirect code execution to, though?
True to the release notes, the unlock routine has been removed from the firmware and the HSM2 is responsible for directly unlocking the door upon successful password validation.

Things get interesting here. We could assemble our own instructions, input them into the buffer and redirect execution to the start of the buffer. This is the fundamental idea behind overflow errors and their consequences.

Here is what the interrupt vector documentation states:

```
INT 0x7D
Interface with the HSM-1. Set a flag in memory if the password passed in is correct. Takes two arguments. The first argument is the password to test, the second is the location of a flag to overwrite if the password is correct. 

INT 0x7E
Interface with the HSM-2. Trigger the deadbolt unlock if the password is correct. Takes one argument: the password to test.

INT 0x7F
Interface with deadbolt to trigger an unlock if the password is correct. Takes no arguments.
```

We push `0x7f` onto the stack and then trigger the interrupt. Here is what the assembly looks like:

![mc_7_1.png]({{site.baseurl}}/assets/img/mc_7_1.png)

This amounts to 8 bytes. We now require 8 more bytes to fill up the input buffer, and then 2 bytes pointing to the start of this input — which lies at `0x3808`.

Input (starting at addr. `3808`): 

`30127f00b0123245 0000000000000000 0838`

**This is a fundamental concept to understand — the injection of code at address ‘X’, followed by redirection of the program counter to address X, causing execution. Below is an illustration that might help visually: ((1.) What the programmer intended (2.) Concept of our input)**

![mc_7_2.png]({{site.baseurl}}/assets/img/mc_7_2.png)


## Level 6: Reykjavik

The main routine is quite different here compared to previous locks. It moves `#0xf8` to `r14`, `#0x2400` to `r15` and then calls the `enc` routine. It might be of interest to note that `0x2400` onwards is already filled with data, which is unusual compared to previous locks.

![mc_6_1.png]({{site.baseurl}}/assets/img/mc_6_1.png)

The `enc` routine is quite lengthy, comprising of 146 bytes of code, roughly equal to 73 instructions. Following these instructions line by line seemed quite complex, so instead, I stepped out of the function to notice a call to the address `0x2400`. This would indicate that the `enc` routine decrypts instructions stored at `0x2400` and then invokes it immediately after.

Below is the disassembly of a few of the instructions at that location:

![mc_6_2.png]({{site.baseurl}}/assets/img/mc_6_2.png)

These instructions did not really make complete sense until I noticed instruction:  

`b490 2417 dcff : cmp #0x1724, -0x24(r4)`

We run the program, enter our input `AABBCCDDEEFFGGHHIIJJKKLL`, and then step through, one instruction at a time.

After a couple of instructions, we step to the interesting looking `cmp #0x1724, -0x24(r4)`

Upon inspecting `0x24` bytes behind `r4` ( `0x43fe` ):

![mc_6_3.png]({{site.baseurl}}/assets/img/mc_6_3.png)

Simply put, after all the complex decryption of instructions into memory and obfuscation, the firmware compares user input to a hard-coded hex `0x1724`.

`Input (hex): 2417    ; unlock`

## Level 5: Cusco

Fuzzing the input with AABBCCDDEEFFGGHHIIJJKKLL results in a program crash — insn address misaligned. 

Below is a capture of register values at the point of crash. We see that `AABBCCDDEEFFGGHH is accepted, however `II (4949)` overwrites the `pc`

![mc_5_1.png]({{site.baseurl}}/assets/img/mc_5_1.png)

The first 8 characters of our input are accepted, however, the rest begin overwriting the stack.

We find that the 2 bytes after the 16-byte input buffer contains the return address — the location that will be loaded into the program counter after password validation. There are no bounds checking in the code after input, therefore, we can simply overwrite these 2 bytes and direct code execution. We see that the routine `unlock_door` is present at address `0x4464`.

![mc_5_2.png]({{site.baseurl}}/assets/img/mc_5_2.png)

What happens when you enter 16 bytes followed by the above?

`Input (hex): 41414242434344444545464647474848 4644`

Unlocked.
