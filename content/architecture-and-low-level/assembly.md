---
title: "x86-64 Assembly Fundamentals: Registers, Syscalls, and Control Flow"
---

going to use intel syntax with as assembler

some of what's here comes from pwn.college challenges i was working through while writing this. they show up in a few of the examples, but this isn't just copied challenge content, they're a small part of a bigger set of notes.

## 0. Assembler Directives and Sections

Assembler directives are special instructions for the assembler that organize code, set syntax, and manage what part of the code does what.  
Some common directives:
- `.intel_syntax noprefix`: Tells assembler to use Intel-style syntax for instructions.
- `.global _start`: Makes the `_start` label visible to linker, marking your entry point.
- `.section text`: Begins the code section (machine instructions go here).
- `.section data`: Begins the data section (static variables, strings, arrays go here).

> `as` normally uses AT&T syntax (`%rax`, `$42`, source-then-destination) by default. `.intel_syntax noprefix` is what switches it over to the intel style used here (`rax`, `42`, destination-then-source, no `%`/`$`).

**Example:**
```assembly
.intel_syntax noprefix
.global _start
.section text
_start:
    mov rdi, 0
    mov rax, 60
    syscall
.section data
```

## 1. Registers and Memory Access


| 64-bit | 32-bit | 16-bit | 8-bit (high) | 8-bit (low) |
|--------|--------|--------|--------------|-------------|
| RAX    | EAX    | AX     | AH           | AL          |
| RBX    | EBX    | BX     | BH           | BL          |
| RCX    | ECX    | CX     | CH           | CL          |
| RDX    | EDX    | DX     | DH           | DL          |
| RSI    | ESI    | SI     | *(none)*     | SIL         |
| RDI    | EDI    | DI     | *(none)*     | DIL         |
| RBP    | EBP    | BP     | *(none)*     | BPL         |
| RSP    | ESP    | SP     | *(none)*     | SPL         |
| R8     | R8D    | R8W    | *(none)*     | R8B         |
| R9     | R9D    | R9W    | *(none)*     | R9B         |
| R10    | R10D   | R10W   | *(none)*     | R10B        |
| R11    | R11D   | R11W   | *(none)*     | R11B        |
| R12    | R12D   | R12W   | *(none)*     | R12B        |
| R13    | R13D   | R13W   | *(none)*     | R13B        |
| R14    | R14D   | R14W   | *(none)*     | R14B        |
| R15    | R15D   | R15W   | *(none)*     | R15B        |

> registers work like this. for example we can call al for lower 8 bits of rax.





`rdi` keeps the argument for exit syscall and `rax` stores syscall number:

```
hacker@dojo:~$ cat asm.s
.intel_syntax noprefix
.global _start
_start:
mov rdi, 42
mov rax, 60
syscall
hacker@dojo:~$ as -o asm.o asm.s
hacker@dojo:~$ ld -o exe asm.o
hacker@dojo:~$ ./exe
hacker@dojo:~$ echo $?
42
```

`echo $?`  command prints the exit status of the last executed program (`./exe`), which is `42`.

`.global _start` tells the program where to begin to read the code.

## 2. Building and Running Assembly Programs

To turn your assembly source file into an executable program on Linux, you need to use two tools:

1. The **assembler** (`as`) converts your `.s` file to an object file (`.o`).
2. The **linker** (`ld`) takes the object file and produces an executable binary.

**Step by Step:**

```bash
$ as -o server.o server.s         # assemble: creates server.o from server.s
$ ld -o server server.o           # link: creates executable 'server' from object file
$ ./server                        # run your program
```
## 3. Instructions

in previous example we learn how to put some value into a register with `mov` instruction.

```assembly
mov rsi, 0x404000    # moves 0x404000 to rsi register
mov rdi, [rsi]       # moves the value in the address of rsi to the rdi register
```

we can also do some math. for example if we want to set a linear function like  `f(x) = mx + b` and place the result in rax, we can use the instructions [[#imul]] (or [[#mul]] if we're dealing with positive numbers), and [[#add]].

```
.intel_syntax noprefix
.global _start
_start:
imul rdi, rsi
add rdi, rdx
mov rax, rdi
```

---

If we have `x % y`, and `y` is a power of 2, such as `2^n`, the result will be the lower `n` bits of `x`.

- `rax = rdi % 256`
- `rbx = rsi % 65536`


```
.intel_syntax noprefix
.global _start
_start:
mov al, dil
mov bx, si
```

dont forget we cant mov an 8-bit reg to a 64-bit register like:
`mov rax, dil`

mov needs both operands to be the same size. thats why movzx/movsx exist, to zero- or sign-extend a smaller value into a bigger register.

we can use [[#movzx]] for this. 

---


x86 allows you to 'shift' bits around in a register.

You can use this to do special things to the bits you care about.

Shifting has the nice side effect of doing quick multiplication (by 2) or division (by 2), and can also be used to compute modulo.

Here are the important instructions:

- `shl reg1, reg2` <=> Shift `reg1` left by the amount in `reg2`
- `shr reg1, reg2` <=> Shift `reg1` right by the amount in `reg2`

> note: when the shift amount comes from a register rather than an immediate, x86 requires it to specifically be `cl`. you can't use any arbitrary register there.

Please perform the following: Set `rax` to the 5th least significant byte of `rdi`.

For example:

```
rdi = | B7 | B6 | B5 | B4 | B3 | B2 | B1 | B0 |

```

for setting rax to the value of B4 we should shift 32 to the right and then use dil to mov al

```
.intel_syntax noprefix
.global _start
_start:
shr rdi, 32
mov al, dil
```


---


 In assembly language, the way a number literal is interpreted depends on its format:

 - If the number starts with a `0` (like `0101`), it’s treated as **octal** (base-8), not decimal! So `0101` becomes decimal 65.
 - If the number doesn’t start with a zero (like `101`), it’s interpreted as **decimal** (base-10), so `101` stays 101.


_Reference: The summary table shows how number literals are interpreted based on their prefix: leading `0` means octal, no prefix means decimal, `0x` means hexadecimal, `0b` means binary._


---

if we're dealing with addresses in a 64-bit context, we have to specify the operand size as `QWORD PTR` for a 64-bit operation:


```
mov rax, [0x404000]
add qword ptr [0x404000], 0x1337
```

---

In x86_64, you can access each of these sizes when dereferencing an address, just like using bigger or smaller register accesses:

- `mov al, [address]` <=> moves the least significant byte from address to `rax`
- `mov ax, [address]` <=> moves the least significant word from address to `rax`
- `mov eax, [address]` <=> moves the least significant double word from address to `rax`
- `mov rax, [address]` <=> moves the full quad word from address to `rax`

you can take a look at them at [[#registers]]

lets try to
- Set `rax` to the byte at `0x404000`
- Set `rbx` to the word at `0x404000`
- Set `rcx` to the double word at `0x404000`
- Set `rdx` to the quad word at `0x404000`

```
mov al, [0x404000]
mov bx, [0x404000]
mov ecx, [0x404000]
mov rdx, [0x404000]
```

this should work.


---


It is worth noting, as you may have noticed, that values are stored in reverse order of how we represent them.

As an example, say:

```
[0x1330] = 0x00000000deadc0de
```

If you examined how it actually looked in memory, you would see:

```
[0x1330] = 0xde
[0x1331] = 0xc0
[0x1332] = 0xad
[0x1333] = 0xde
[0x1334] = 0x00
[0x1335] = 0x00
[0x1336] = 0x00
[0x1337] = 0x00
```

This format of storing things in 'reverse' is intentional in x86, and it's called "Little Endian".


if we want to set `[rdi] = 0xdeadbeef00001337` we can do it through another register. i used rax:

```
.intel_syntax noprefix
.global _start
_start:
mov rax, 0xdeadbeef00001337
mov [rdi], rax
```

as another example:

```
[0x1337] = 0x00000000deadbeef
```

The real way memory is laid out is byte by byte, little endian:

```
[0x1337] = 0xef
[0x1337 + 1] = 0xbe
[0x1337 + 2] = 0xad
...
[0x1337 + 7] = 0x00
```

What does this do for us?

Well, it means that we can access things next to each other using offsets, similar to what was shown above.

Say you want the 5th _byte_ from an address, you can access it like:

```
mov al, [address+4]
```


Remember, offsets start at 0.

---


#### stack

The stack is a region of memory that can store values for later.

To store a value on the stack, we use the `push` instruction, and to retrieve a value, we use `pop`.

The stack is a last in, first out (LIFO) memory structure, and this means the last value pushed is the first value popped.

also the stack pointer register (`rsp`) points to the stack.

lets try something like take the top value of the stack, subtract `rdi` from it, then put it back.

```
.intel_syntax noprefix
.global _start
_start:
pop rax
sub rax, rdi
push rax
```


its actually fun in some way. we can swap values with the stack:


- If to start `rdi = 2` and `rsi = 5`
- Then to end `rdi = 5` and `rsi = 2`

```
_start:

push rdi
push rsi
pop rdi
pop rsi
```


when we push it just pushes the value of that register.


On x86, the stack pointer is stored in the special register, `rsp`. `rsp` always stores the memory address of the top of the stack, i.e., the memory address of the last value pushed.

Similar to the memory levels, we can use `[rsp]` to access the value at the memory address in `rsp`.

so we can calculate the average of 4 consecutive quad words stored on the stack without using `pop`.

```
.intel_syntax noprefix
.global _start
_start:

mov rax, [rsp]
add rax, [rsp+0x8]
add rax, [rsp+0x10]
add rax, [rsp+0x18]
mov rdi, 4
div rdi
push rax
```

---


### system calls and i/o

a syscall is how a program asks the kernel to do something it can't do on its own: read a file, write output, exit, etc. the `syscall` instruction traps into the kernel, which looks at the number in `rax` to figure out which operation is being requested.

you remember the code we wrote in the beginning?

```
.intel_syntax noprefix
.global _start
_start:
mov rdi, 42
mov rax, 60
syscall
```

60 is the number for exit syscall. we use rax for storing values for syscalls.

there are so many syscalls but i want to go through two of them in more detail.


in the example above, It reads up to 100 bytes of input from standard input (keyboard or piped input), and then immediately writes those same bytes to standard output (screen).
```
mov rdi, 0      # stdin file descriptor (0: standard input)
mov rsi, rsp    # read data into the stack (at rsp address)
mov rdx, 100    # read 100 bytes
mov rax, 0      # read system call number (0)
syscall         # perform read, number of bytes read will be stored in rax

mov rdi, 1      # stdout file descriptor (1: standard output)
mov rsi, rsp    # write data from the stack (at rsp address)
mov rdx, rax    # write as many bytes as were read (rax: return value of read)
mov rax, 1      # write system call number (1)
syscall         # perform write

```

0 is for reading and 1 is for writing.

worth noting these are actually two different things that happen to line up: `0`/`1` in `rdi` are file descriptors (stdin/stdout), while `0`/`1` in `rax` are syscall numbers (read/write). it's not a general "0 means read" rule. it's just a coincidence for these two specific syscalls.

theres another example you can look at it:

```
mov rdi, 0                      # Set rdi register to 0 -> file descriptor 0 (stdin)
mov rsi, 1337000                # Set rsi register to 1337000 -> buffer address for reading data
mov rdx, 8                      # Set rdx register to 8 -> number of bytes to read
mov rax, 0                      # Set rax register to 0 -> syscall number for 'read'
syscall                         # Invoke the system call: read(stdin, buffer, 8 bytes)

mov rdi, 1                      # Set rdi register to 1 -> file descriptor 1 (stdout)
mov rsi, 1337000                # Set rsi register to 1337000 -> buffer address to write data from
mov rdx, rax                    # Set rdx to value of rax -> number of bytes actually read previously
mov rax, 1                      # Set rax register to 1 -> syscall number for 'write'
syscall                         # Invoke the system call: write(stdout, buffer, rax bytes)

mov rdi, 42                     # Set rdi register to 42 -> exit code 42
mov rax, 60                     # Set rax register to 60 -> syscall number for 'exit'
syscall                         # Invoke the system call: exit(42)
```


## 4. Arithmetic and Data Manipulation

we've seen some of the arithmetic instructions like imul or add before. we are gonna learn about division here and its a little bit harder than others.

to remember:
- `add reg1, reg2` <=> `reg1 += reg2`
- `sub reg1, reg2` <=> `reg1 -= reg2`
- `imul reg1, reg2` <=> `reg1 *= reg2`

in assembly we cant use registers like variables in other languages. most of them have a specific role tied to the CPU architecture and calling conventions. some examples: `rdi`/`rsi` for passing function arguments, `rax` for return values, `rcx` as a counter, `rbp` as a base pointer, and `rbx`/`rbp`/`r12-r15` as ones that have to be preserved across function calls.

division leans on specific registers more than the other arithmetic ops. for 64-bit division, the dividend gets split across `rdx` (high bits) and `rax` (low bits) as a 128-bit value, the divisor goes in any general-purpose register, and once `div` runs the quotient lands in `rax` and the remainder in `rdx`.


| Variant | Divisor | Dividend | Quotient | Remainder |
| ------- | ------- | -------- | -------- | --------- |
| 8-bit   | reg8            | AX                   | AL                 | AH                  |
| 16-bit  | reg16           | DX:AX                | AX                 | DX                  |
| 32-bit  | reg32           | EDX:EAX              | EAX                | EDX                 |
| 64-bit  | reg64           | rdx:rax              | rax                | rdx                 |
_The CPU always uses fixed registers for dividend, quotient, and remainder depending on the operand size._

here are some examples:

lets try to calculate `speed = distance / time`


distance must be stored in rdx:rax and speed is going to be stored in rax.

we can choose the register of time. i used rsi for this example and took distance from rdi:

```
.intel_syntax noprefix
.global _start
_start:
mov rax, rdi
div rsi
```



another example for computing the following: `rdi % rsi`

Place the value in `rax`.

```
.intel_syntax noprefix
.global _start
_start:
mov rax, rdi
div rsi
mov rax, rdx
```


## 5. Bitwise Operations

Bitwise operations in assembly manipulate individual bits within registers or memory locations using operators like AND, OR, XOR, and NOT. These operations are fundamental for tasks such as setting, clearing, or toggling specific bits efficiently. Unlike high-level languages, assembly provides direct control over these operations, making them essential for low-level programming, optimization, and hardware interfacing. 


we talked about shifting bits earlier. bit shifts (left and right) are also common bitwise operations used to multiply or divide values by powers of two.

- **AND**
    
    ```
    A | B | X
    ---+---+---
    0 | 0 | 0
    0 | 1 | 0
    1 | 0 | 0
    1 | 1 | 1
    ```
    
- **OR**
    
    ```
    A | B | X
    ---+---+---
    0 | 0 | 0
    0 | 1 | 1
    1 | 0 | 1
    1 | 1 | 1
    ```
    
- **XOR**
    
    ```
    A | B | X
    ---+---+---
    0 | 0 | 0
    0 | 1 | 1
    1 | 0 | 1
    1 | 1 | 0
    ```


i think the coolest part of bitwise operations are that we can do so many things just using those operations. for example if we want to set rax to the value of (rdi AND rsi):

```
and rdi, rsi
mov rax, rdi
```

we do this, right? but what if i say theres a cooler way to do this?

```
and rdi, rsi
and rax, 0
or rax, rdi
```

this isn't actually shorter or faster than the first version. it ends up with the same value in `rax`, just by zeroing it and then `or`-ing the source in instead of a direct `mov`.

we can also swap the value of two registers without pushing them to the stack just using xor.


```
xor a, b 
xor b, a
xor a, b 
```

(`a` and `b` here are placeholders for any two registers, e.g. `rdi`/`rsi`.)

this'd work.

there are so many things that we can do with those operands but im going to give you one more example. 

```plaintext
if x is even then
  y = 1
else
  y = 0
```

Where:
- `x = rdi`
- `y = rax`

```
and rdi, 1
xor rdi, 1
and rax, 00000000 ;we just wanna make sure rax is 0
or rax, rdi
```


in line 3:
`and rax, 00000000 ;we just wanna make sure rax is 0`

theres another common way to do this:
`xor rax`

`xor reg, reg` is the more idiomatic way to zero a register than `and reg, 0`. it's a shorter encoding and doesn't need an immediate operand.

>[!TIP]
branching can affect cpu's performance. when we use bitwise operations instead of conditionals that can improve cpu performance hence its more branchless. this is because a mispredicted conditional branch forces the CPU to flush and refill its pipeline, which costs cycles. branchless code sidesteps that entirely.





---

## 6. Control Flow and Branching

- **`jmp` (Jump):**  
  The `jmp` instruction performs an unconditional jump to a specified address or label in the program. It changes the Instruction Pointer (IP or RIP in 64-bit systems) to redirect code execution to another part of the code, without any conditions.

- **`call`:**  
  The `call` instruction invokes a subroutine (function). It saves the address of the next instruction (the return address) onto the stack and then jumps to the specified procedure or label. After the subroutine is done, a `ret` (return) instruction can be used to continue execution from the saved address.

- **`cmp` (Compare):**  
  The `cmp` instruction is used to compare two operands. It computes the difference between them, updates the CPU flags based on the result, but does not store the result itself. The flags (such as Zero Flag, Sign Flag, etc.) are then typically used by conditional jump instructions (e.g., `je`, `jne`) to determine program flow.

We can't directly `mov` the flags into a register. Instead, x86 provides a family of "set on condition" instructions that write a `0` or `1` to a byte-sized destination based on the current flags. 

The one we'll use here is `setz` ("Set if Zero"):

```asm
setz dil
```

This checks the Zero Flag and:

- If ZF = 1 (the values **were** equal, i.e., the subtraction result was zero), it writes `1` to `dil`.
- If ZF = 0 (the values were **not** equal), it writes `0` to `dil`.

Simple: `1` means "yes, they matched!" and `0` means "no, they didn't." There's also a complementary instruction, `setnz` ("Set if Not Zero"), which does the opposite, but we won't need it here.

- `dil` is just the lowest 8 bits --- the **l**ow byte of r**di**

`cmp` can compare a register with an immediate (`cmp rdi, 42`) or even a memory location with an immediate (`cmp BYTE PTR [rsp], 42`). But it **cannot** compare two memory locations at once --- at most one operand can be a memory dereference. This is a general rule in x86 and, actually, in almost all CPU architectures.
**Memory-to-Memory** operations are forbidden because they would require multiple "load" micro-ops, making the instruction encoding too complex and the execution too slow compared to using a register as a middleman.

assuming a prior `cmp rdi, 3`, the conditional jumps read as:

| Command | Meaning (unsigned)     | Condition |
| ------- | ---------------------- | --------- |
| ja      | Jump if Above          | rdi > 3   |
| jae     | Jump if Above or Equal | rdi >= 3  |
| jb      | Jump if Below          | rdi < 3   |
| jbe     | Jump if Below or Equal | rdi <= 3  |

Earlier, we learned how to manipulate data in a pseudo-control way, but x86 gives us actual instructions to manipulate control flow directly.

There are two major ways to manipulate control flow:

- Through a jump
- Through a call

In this level, you will work with jumps.

There are two types of jumps:

- Unconditional jumps
- Conditional jumps

Unconditional jumps always trigger and are not based on the results of earlier instructions.

As you know, memory locations can store data and instructions. Your code will be stored at `0x400042` (this will change each run).

For all jumps, there are three types:

- Relative jumps: jump + or - the next instruction.
- Absolute jumps: jump to a specific address.
- Indirect jumps: jump to the memory address specified in a register.

In x86, absolute jumps (jump to a specific address) are accomplished by first putting the target address in a register `reg`, then doing `jmp reg`.


Jumping to the absolute address `0x403000`:

```
mov rax, 0x403000
jmp rax
```

---

this one was one of the hardest challenges:


Hint: For the relative jump, look up how to use `labels` in x86.


- Make the first instruction in your code a `jmp`.
- Make that `jmp` a relative jump to 0x51 bytes from the current position.
- At the code location where the relative jump will redirect control flow, set `rax` to 0x1.


```
.intel_syntax noprefix
.global _start
_start:
jmp target
.rept 81
nop
.endr
target:
mov rax, 0x1
```


In fact, the assembler that we're using has a handy `.rept` directive that you can use to repeat assembly instructions some number of times: [GNU Assembler Manual](https://ftp.gnu.org/old-gnu/Manuals/gas-2.9.1/html_chapter/as_7.html)

---


Creating a two jump trampoline:

- Make the first instruction in your code a `jmp`.
- Make that `jmp` a relative jump to 0x51 bytes from its current position.
- At 0x51, write the following code:
    - Place the top value on the stack into register `rdi`.
    - `jmp` to the absolute address 0x403000.




```
.intel_syntax noprefix
.global _start
_start:
jmp target
.rept 81
nop
.endr
target:
pop rdi
mov rax, 0x403000
jmp rax
```

---

implementing the following:

these two constants are real file-format magic numbers: `0x7f454c46` is the ELF magic number (bytes `\x7fELF`), and `0x00005A4D` is the "MZ" magic used by DOS/PE (Windows) executables, so this is really "detect ELF vs PE vs neither."

```plaintext
if [x] is 0x7f454c46:
    y = [x+4] + [x+8] + [x+12]
else if [x] is 0x00005A4D:
    y = [x+4] - [x+8] - [x+12]
else:
    y = [x+4] * [x+8] * [x+12]
```


```
.intel_syntax noprefix
.global _start
_start:
cmp dword ptr [edi], 0x7f454c46
je equal
cmp dword ptr [edi], 0x00005A4D
je equal2
mov eax, [edi+4]
imul eax, [edi+8]
imul eax, [edi+12]
jmp done
equal:
mov eax, [edi+4]
add eax, [edi+8]
add eax, [edi+12]
jmp done
equal2:
mov eax, [edi+4]
sub eax, [edi+8]
sub eax, [edi+12]
jmp done
done:
nop
```


#### switch-case

The last jump type is the indirect jump, often used for switch statements in the real world. Switch statements are a special case of if-statements that use only numbers to determine where the control flow will go.

Here is an example:

```
switch(number):
  0: jmp do_thing_0
  1: jmp do_thing_1
  2: jmp do_thing_2
  default: jmp do_default_thing
```

The switch in this example works on `number`, which can either be 0, 1, or 2. If `number` is not one of those numbers, the default triggers. You can consider this a reduced else-if type structure. In x86, you are already used to using numbers, so it should be no surprise that you can make if statements based on something being an exact number. Additionally, if you know the range of the numbers, a switch statement works very well.

Take, for instance, the existence of a jump table. A jump table is a contiguous section of memory that holds addresses of places to jump.

In the above example, the jump table could look like:

```
[0x1337] = address of do_thing_0
[0x1337+0x8] = address of do_thing_1
[0x1337+0x10] = address of do_thing_2
[0x1337+0x18] = address of do_default_thing
```

Using the jump table, we can greatly reduce the amount of `cmps` we use. Now all we need to check is if `number` is greater than 2. If it is, always do:

```
jmp [0x1337+0x18]
```

Otherwise:

```
jmp [jump_table_address + number * 8]
```

Using the above knowledge, implement the following logic:

```plaintext
if rdi is 0:
  jmp 0x40301e
else if rdi is 1:
  jmp 0x4030da
else if rdi is 2:
  jmp 0x4031d5
else if rdi is 3:
  jmp 0x403268
else:
  jmp 0x40332c
```


---

Please do the above with the following constraints:

- Assume `rdi` will NOT be negative.
- Use no more than 1 `cmp` instruction.
- Use no more than 3 jumps (of any variant).
- We will provide you with the number to 'switch' on in `rdi`.
- We will provide you with a jump table base address in `rsi`.

Here is an example table:

```
[0x40427c] = 0x40301e (addrs will change)
[0x404284] = 0x4030da
[0x40428c] = 0x4031d5
[0x404294] = 0x403268
[0x40429c] = 0x40332c
```


- We use a **jump table** to implement an efficient switch-case structure in assembly, minimizing the number of conditional checks.
- With only one `cmp` and a couple of jumps, we check if the input index (`rdi`) is within the valid case range or not.
- If the index is out of range, we redirect it to a single default entry in the jump table, avoiding unsafe memory access.
- This approach takes full advantage of **indirect jumps** and index arithmetic to achieve both safety and speed, while strictly adhering to the given instruction constraints.

the code block below is what I wrote without any constraints:
```

.intel_syntax noprefix
.global _start
_start:
cmp rdi, 0
jne notzero

zero:
jmp 0x40301e

notzero:
cmp rdi, 3
ja default
jmp [rsi+rdi*8] 


default:
jmp 0x40332c
```

and this is the right one:
```
cmp rdi, 3
ja default
jmp [rsi+rdi*8]
default:
jmp [rsi+0x20]

```

the separate `rdi == 0` check from the first attempt turned out to be unnecessary: index 0 lands correctly through `jmp [rsi+rdi*8]` just like any other in-range index, so folding it into the same jump-table check is what gets this down to one `cmp` and two jumps.

---

#### call example

a quick example tying `call` and `ret` together:

```assembly
.intel_syntax noprefix
.global _start
_start:
    mov rdi, 42           ; we put a number into the rdi register
    call print_number     ; go to the print_number function (return address is pushed onto the stack)
    ; print_number finishes and comes back here with 'ret'
    mov rax, 60           ; exit system call number (on Linux)
    xor rdi, rdi          ; exit code 0
    syscall               ; terminate the program
print_number:
    ; No actual write operation here, just sample code
    nop                   ; (No Operation - for demonstration purposes)
    ret                   ; return to the address before call (just below the call in _start)
```

---

In most programming languages, a structure exists called the for-loop, which allows you to execute a set of instructions for a bounded amount of times. The bounded amount can be either known before or during the program's run, with "during" meaning the value is given to you dynamically.

As an example, a for-loop can be used to compute the sum of the numbers 1 to n:

```plaintext
sum = 0
i = 1
while i <= n:
    sum += i
    i += 1
```

Please compute the average of `n` consecutive quad words, where:

- `rdi` = memory address of the 1st quad word
- `rsi` = `n` (amount to loop for)
- `rax` = average computed


```
.intel_syntax noprefix
.global _start
_start:
mov rbx, 0
mov rax, 0

loop:
add rax, [rdi+rbx*8]
add rbx, 1
cmp rbx, rsi
jb loop
div rsi
```

i had to use rbx because theres no way to solve this challenge without defining an i

---

Count the consecutive non-zero bytes in a contiguous region of memory, where:

- `rdi` = memory address of the 1st byte
- `rax` = number of consecutive non-zero bytes

Additionally, if `rdi = 0`, then set `rax = 0` (we will check)!

```
.intel_syntax noprefix
.global _start
_start:

cmp rdi, 0
je zero
mov rax, 0

loop:
cmp word ptr [rdi+rax], 0
je finish
add rax, 1
jmp loop


zero:
mov rax, 0

finish:
```


---

Please implement the following logic:

```plaintext
str_lower(src_addr):
  i = 0
  if src_addr != 0:
    while [src_addr] != 0x00:
      if [src_addr] <= 0x5a:
        [src_addr] = foo([src_addr])
        i += 1
      src_addr += 1
  return i
```


```
.intel_syntax noprefix
.global _start
_start:
call str_lower
mov rax, rbx
mov rsi, rdi

str_lower:
mov rbx, 0
cmp rsi, 0
je end
cmp byte ptr [rsi], 0
je end
while:
        cmp byte ptr [rsi+rbx*1], 0x00
        je end
        cmp byte ptr [rsi+rbx*1], 0x5a
        ja cont
        movzx rdi, byte ptr [rsi+rbx]
        call [0x403000]
        mov [rsi+rbx], rax

        cont:
        add rbx, 1
        jmp while
end:
mov rax, rbx
ret

```


---

#### functions


A function is a callable segment of code that does not destroy control flow.

Functions use the instructions "call" and "ret".

The "call" instruction pushes the memory address of the next instruction onto the stack and then jumps to the value stored in the first argument.

Let's use the following instructions as an example:

```
0x1021 mov rax, 0x400000
0x1028 call rax
0x102a mov [rsi], rax
```

1. `call` pushes `0x102a`, the address of the next instruction, onto the stack.
2. `call` jumps to `0x400000`, the value stored in `rax`.

The "ret" instruction is the opposite of "call".

`ret` pops the top value off of the stack and jumps to it.

Let's use the following instructions and stack as an example:

```
Stack ADDR  VALUE
0x103f mov rax, rdx         RSP + 0x8   0xdeadbeef
0x1042 ret                  RSP + 0x0   0x0000102a
```

Here, `ret` will jump to `0x102a`.

This is the same `str_lower` problem shown earlier, revisited here using the proper System V calling convention (preserving registers across the `call`, and calling `foo`'s address directly).

Please implement the following logic:

```plaintext
str_lower(src_addr):
  i = 0
  if src_addr != 0:
    while [src_addr] != 0x00:
      if [src_addr] <= 0x5a:
        [src_addr] = foo([src_addr])
        i += 1
      src_addr += 1
  return i
```

`foo` is provided at `0x403000`. `foo` takes a single argument as a value and returns a value.

All functions (`foo` and `str_lower`) must follow the Linux amd64 calling convention (also known as System V AMD64 ABI): [System V AMD64 ABI](https://en.wikipedia.org/wiki/X86_calling_conventions#System_V_AMD64_ABI)

Therefore, your function `str_lower` should look for `src_addr` in `rdi` and place the function return in `rax`.

An important note is that `src_addr` is an address in memory (where the string is located) and `[src_addr]` refers to the byte that exists at `src_addr`.

Therefore, the function `foo` accepts a byte as its first argument and returns a byte.




```
.intel_syntax noprefix
.global _start

str_lower:
mov rbx, 0
cmp rdi, 0
je end
while:
        cmp byte ptr [rdi], 0x00
        je end
        cmp byte ptr [rdi], 0x5a
        ja cont
        push rdi
        push rbx
        movzx rdi, byte ptr [rdi]

        mov rax, 0x403000
        call rax
        pop rbx
        pop rdi
        mov byte ptr [rdi], al
        inc rbx

        cont:
        inc rdi
        jmp while
end:
mov rax, rbx
ret
_start:
call str_lower

```


---

## 7. Number Representations and Literals

the top bit of a signed number is the sign bit: 0 positive, 1 negative. bit 7 for an 8-bit value.

negative numbers use two's complement: invert every bit and add 1. `5` is `00000101` → invert to `11111010` → +1 = `11111011`, which is `-5`.

why not just flip the sign bit and call it a day? because then `add`/`sub` would need separate signed/unsigned logic, and you'd end up with two zeros (`+0` and `-0`). two's complement avoids both.

## 8. Disassembly and Binary Analysis


you can disassemble your program:

`objdump -M intel -d quitter` (`quitter` here is just the name of whatever compiled binary you're analyzing. swap it for your own.)



### Binary Sections & Extracting Hardcoded Strings

when a C program gets compiled (into an ELF on Linux), it's not just a blob of instructions. it gets split into sections, each with its own memory permissions.

the common ones:
- `.text`: the actual instructions. read + execute.
- `.data`: global/static variables that start out non-zero. read + write.
- `.bss`: global/static variables that start out zero (or uninitialized). read + write.
- `.rodata`: read-only data, constants, string literals, that kind of thing.

#### `.rodata` example

say you've got this:

```c
#include <stdio.h>
#include <string.h>

int main(int argc, char *argv[]) {
    if (argc > 1 && strcmp(argv[1], "SuperSecretKey") == 0) {
        printf("Access Granted!\n");
    } else {
        printf("Access Denied.\n");
    }
    return 0;
}
```

the string `"SuperSecretKey"` doesn't get embedded in the `strcmp` instruction itself. it sits in `.rodata` at some address (say `0x402000`), and `.text` just references that address when it calls `strcmp`.

#### pulling it out

since the string just sits there untouched by the CPU logic, you don't need to trace through any assembly to get it. you can read it straight out of the binary.

```
# -s displays the full raw contents
# -j targets the specific section
objdump -s -j .rodata ./binary_name
```

_Example Output:_

```
Contents of section .rodata:
 402000 53757065 72536563 7265744b 657900   SuperSecretKey.
```


