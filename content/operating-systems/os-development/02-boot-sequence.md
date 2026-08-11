---
title: "2. Boot Sequence & Initialization"
---

part of the [[index|os-development]] series.

### the linker script

The linker script tells the linker (the tool that combines your `.o` files into an executable) how to map your code to physical addresses.

```c
ENTRY(boot)

SECTIONS {
    . = 0x80200000;

    .text :{
        KEEP(*(.text.boot));
        *(.text .text.*);
    }

    .rodata : ALIGN(4) {
        *(.rodata .rodata.*);
    }

    .data : ALIGN(4) {
        *(.data .data.*);
    }

    .bss : ALIGN(4) {
        __bss = .;
        *(.bss .bss.* .sbss .sbss.*);
        __bss_end = .;
    }

    . = ALIGN(4);
    . += 128 * 1024; /* 128KB */
    __stack_top = .;
}
```

this is the order the linker places sections in, following this script.

here is the explanation:

**`ENTRY(boot)`**: this is telling the linker (and the hardware) that the very first instruction the CPU should execute is the label named `boot`.

**`. = 0x80200000`**: This is the "Origin." this tells the linker to start placing the following sections at this specific memory address. This is usually where QEMU expects the kernel to start.

The order (`.text` $\rightarrow$ `.rodata` $\rightarrow$ `.data` $\rightarrow$ `.bss`) is the order they will appear in the final binary.

**`KEEP(*(.text.boot))`**: This is crucial. It tells the linker: "Find the code I marked as `.text.boot` and put it **first**, before any other code." This ensures the CPU hits your boot code immediately.
    
**`*(.text .text.*)`**: The asterisk `*` means "from all input files." So, "Take the `.text` section from every single `.o` file and put them here."



**The Symbols (`__bss`, `__stack_top`)**


The script calculates the addresses for us.

- **`__bss = .`**: This captures the memory address where the BSS section starts.

- In the code, we can now use the name `__bss` as a variable to know exactly where in RAM we need to start zeroing out memory.

| Section   | Description                                                          |
| --------- | -------------------------------------------------------------------- |
| `.text`   | This section contains the code of the program.                       |
| `.rodata` | This section contains constant data that is read-only.               |
| `.data`   | This section contains read/write data.                               |
| `.bss`    | This section contains read/write data with an initial value of zero. |

Unlike the `.text` or `.data` sections, the **Stack** does not originate from the compiled `.o` files. While it is defined within the `SECTIONS` block of the linker script, it serves as a **manual reservation** of raw memory. By incrementing the location counter (`.`), we 'fence off' a specific chunk of RAM to ensure the linker doesn't place other data there. This provides the CPU with a dedicated, empty space to manage function calls and local variables as soon as the stack pointer (`sp`) is initialized in assembly.


The `ALIGN(4)` command is a directive to the linker to ensure that the **Location Counter** (the `.` symbol) is a multiple of 4 bytes before starting the next section, which is known as **Data Alignment**.


### booting


we told the linker (and eventually the debugger) that the program begins execution at the symbol named `boot`. now it's time to write boot:

```c
extern char __bss[], __bss_end[], __stack_top[];
```

we need the addresses to use. thats why we extern them.

```c
__attribute__((section(".text.boot"))) __attribute__((naked)) void boot(void) {
    __asm__ __volatile__(
        "mv sp, %[stack_top]\n" // Set the stack pointer
        "j kernel_main\n"       // Jump to the kernel main function
        :
        : [stack_top] "r"(
            __stack_top) // Pass the stack top address as %[stack_top]
    );
}
```

by `__attribute__((section(".text.boot")))` the linker can easily place this function at the top.

**`__attribute__((naked))`** tells the compiler to leave your function completely alone. It means: _"Do not generate the standard hidden setup and cleanup code for this function. I am writing 100% of the assembly myself."_

### kernel main 

The `kernel_main` function is the very first C code executed by the operating system. 
the initial responsibilities of this function would be:

- **clearing the .bss section:** This is a strict requirement for the C runtime environment, ensuring that any uninitialized global or static variables default to exactly `0` rather than containing leftover garbage memory.

- **The Infinite Loop:** why `kernel_main` can never actually reach its `ret`. this one's worth walking through in detail, since removing it caused a genuinely confusing bug:

We have this kernel that prints "Hello World!" once and parks the CPU:



```c
void kernel_main(void) {
    memset(__bss, 0, (size_t)__bss_end - (size_t)__bss);
    printf("Hello World!");

    for (;;) {
        __asm__ __volatile__("wfi");
    }
}
```

And it gets called from boot like this:



```c
__attribute__((section(".text.boot"))) __attribute__((naked)) void boot(void) {
    __asm__ __volatile__(
        "mv sp, %[stack_top]\n"
        "j kernel_main\n"
        :
        : [stack_top] "r"(__stack_top)
    );
}
```

Everything works fine. Prints once, parks, done.

But what if we removed the infinite loop? I commented out the `wfi` loop, just to see what happens:



```c
void kernel_main(void) {
    memset(__bss, 0, (size_t)__bss_end - (size_t)__bss);
    printf("Hello World!");

    //for (;;) {
    //    __asm__ __volatile__("wfi");
    //}
}
```

Instead of stopping, "Hello World!" printed infinitely. Something was wrong, so I started debugging to check if QEMU was restarting, OpenSBI was re-entering on return, or if `ra` held a strange value.

First, I captured the return address register by adding this right before `kernel_main` returns:

```c
void kernel_main(void) {
    memset(__bss, 0, (size_t)__bss_end - (size_t)__bss);
    printf("Hello World!");

    uint32_t ra_val;
    __asm__ __volatile__("mv %0, ra" : "=r"(ra_val));
    printf("About to return! The RA register holds: 0x%x\n", ra_val);
}
```

The output showed:



```
Hello World!
About to return! The RA register holds: 0x802002de
```

To see where `0x802002de` pointed, I disassembled the kernel:


```
llvm-objdump -d build/kernel.elf | grep -B5 -A5 "802002d"
```

The address was inside my own code, specifically at the `mv a1, ra` instruction in the middle of the second `printf` setup.

Here is what was actually happening:

`boot` is declared `__attribute__((naked))`, so the compiler emits zero prologue and epilogue. Inside it, `j kernel_main` is the pseudo-instruction `jal x0, kernel_main`. This is an unconditional jump that discards the link, meaning it does not write to `ra`. Compare this with `call` or `jal ra, kernel_main`, which would save a valid return address into `ra`. When `kernel_main` starts executing, `ra` still holds whatever leftover firmware state OpenSBI left in it prior to the `mret` into the kernel.

However, `kernel_main` is a normal C function. The compiler emits a standard prologue and epilogue. At `-O2`, the prologue saves `ra` (the leftover value from OpenSBI) onto the stack. The function runs, prints the debug info, and then the epilogue restores that same `ra` value from the stack and executes `ret` (`jr ra`).

That `ret` execution causes the unraveled behavior. The value in `ra` (`0x802002de`) becomes the program counter. There are two landing spots that can produce this infinite loop:

1. `ra` points to the address of `boot` itself (since the linker script puts `.text.boot` first at `0x80200000`). Re-entering `boot` resets the stack pointer and jumps back into `kernel_main`, creating a cycle.
    
2. `ra` lands somewhere inside `kernel_main`'s own instructions, which happened here. Jumping to `0x802002de` executed the printing pipeline again. The stack remained balanced because each iteration cleanly executed the same prologue and epilogue, bouncing control between `ret` and the kernel image.
    

The `wfi` loop fixed this because `for(;;) __asm__("wfi");` is an infinite loop that prevents the function from ever reaching the broken `ret` path. A bare `for(;;);` would have masked the issue the exact same way; `wfi` simply lets the CPU idle instead of spinning at full power.

Ultimately, two factors collided: `boot` used `j` instead of `call`, leaving `ra` uninitialized, and `kernel_main` was allowed to return assuming `ra` held a valid destination. In bare-metal development, you must always ensure `call` is used if a function returns, or guarantee the entry point contains an explicit execution barrier.
