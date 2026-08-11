---
title: "Static Variables, Linkage, and the C Build Pipeline"
---

## static variables

Static variables live for the entire duration of the program. They are stored in the data segment (not the heap), and they are only visible to the function (or the file if theyre declared outside a function) where they are declared.

a quick example:

```c
#include <stdio.h>

// Recursive function that runs from n to 0
// It will terminate for recursive depth
// greater than 10
void fun(int n){
    
    // Static variable
    static int depth = 0;
    if (n == 0 || depth > 10) return;
    printf("%d ", n);
    
    // Increasing with number of recursive calls
    depth++;
    fun(n - 1);
}
int main(){
    fun(-1);
    return 0;
}
```

output:

`-1 -2 -3 -4 -5 -6 -7 -8 -9 -10 -11`  




---

### The Build Pipeline 

_You need to know how code turns into an executable to understand when variables are processed._

- **The Preprocessor:**
    
    - `#include` essentially copy-pastes code.
    - `#define` macros are _not_ variables and have no memory address. they just get replaced in this stage. 

for example:

main.c
```c
#define BONUS 10

int main() {
    int score = 90;
    return score + BONUS;
}
```

this code turns into:

main.i
```c
int main() {
    int score = 90;
    return score + 10;
}
```



- **Compilation (Compile Time):**
    
    - Translation of C/C++ code into Assembly.
    - The compiler takes the massive `.i` file from the preprocessor and translates it into Assembly language. If you have 50 `.c` files, the compiler runs 50 separate times, creating 50 isolated Translation Units (a Translation Unit is just the preprocessed `.c` file, compiled on its own with no knowledge of the others).


main.s
```assembly
	.globl main       ; Make 'main' visible to the linker
main:                 ; The start of our function
    mov eax, 100      ; Move the number 100 into the CPU's 'eax' register
    ret               ; Return to the operating system
```

(the `100` here is `score + BONUS` already folded into one constant: no actual `add` left, since the compiler did that math at compile time.)





- **The Assembler:**
    
    - Conversion of Assembly into Object files (`.o` or `.obj`).
    - **The Symbol Table:** This is the heart of C/C++ architecture. The object file creates a directory mapping names to binary offsets.
	    - _Defined Symbols:_ "I have a function called `calculate()` starting at byte offset 0x4A."
		- _Undefined Symbols:_ "I need a function called `printf()`, but I don't have it. Linker, please find it."


```
[Raw Machine Code]
B8 64 00 00 00     (This is the exact binary translation of 'mov eax, 100')
C3                 (This is the exact binary translation of 'ret')

[Symbol Table]
00000000 T main    (T means Text/Code. It tells the Linker: "The function 'main' is located right at byte 0")
```



     
- **The Linker (Link Time):**
    
	- **The Two-Pass System:** The linker usually scans files twice. Pass 1 catalogs all available symbols from all `.o` files. Pass 2 goes back into the machine code and overwrites the blank "placeholder" addresses with the actual, final memory addresses of the resolved functions and variables.
    
	- **Static vs. Dynamic Linking:** 
		* _Static Linking (`.a` or `.lib`):_ The linker copies the exact machine code of library functions directly into your final executable. The file gets huge, but it's highly portable.
	    - _Dynamic Linking (`.so` or `.dll`):_ The linker just leaves a note: "When the OS runs this program, go find the `libc.so` file on the user's hard drive and load it."

so: initialized static/global data gets baked into the executable at link time, zero-initialized data is just a reserved-space note (see BSS below), and locals only exist at runtime on the stack.


### Program Memory Layout 

When you run an executable file, the Operating System (OS) creates a Process and hands it a virtual memory space sliced into strict segments.

- **The Stack:** Where local variables live (temporary, fast, destroyed when a function ends).
	- **The Mechanics:** Controlled by the CPU's Stack Pointer (SP) register. When a function is called, the SP moves down to carve out a chunk of memory (a Stack Frame) for the return address, parameters, and local variables.
	- **Destruction:** When the function hits `return`, the SP simply jumps back up. The data isn't "erased": the pointer just abandons it. The next function call will overwrite that exact physical space.


#### Ex: stack walkthrough

lets see whats happening inside stack for this program (drawing it growing upward here as new stuff gets pushed: the SP moving "down" in memory just means down in address value, not down on the page):

```c
int add_numbers(int a, int b) {
    int result = a + b;
    return result;
}

int main() {
    int x = 5;
    int y = 10;
    int sum = add_numbers(x, y);
    return 0;
}
```


Step 1: `int x = 5;`

`main()` stack frame is created. Local variable `x` is pushed.



```
|                 |
|                 |
|=================| <-- main() frame starts here
| int x = 5       | <-- SP
|=================|
```

---

Step 2: `int y = 10;`

Local variable `y` is pushed.



```
|                 |
|                 |
|-----------------|
| int y = 10      | <-- SP
|-----------------|
| int x = 5       | 
|=================| 
```

---

Step 3: Preparing to call `add_numbers(x, y)`

Arguments (`a=5`, `b=10`) and the Return Address (where to resume in `main`) are pushed before jumping.



```
|                 |
|-----------------|
| Return Address  | <-- SP
|-----------------|
| arg: a = 5      | 
|-----------------|
| arg: b = 10     | 
|-----------------|
| int y = 10      | 
|-----------------|
| int x = 5       | 
|=================| 
```

---

Step 4: `int result = a + b;` (Inside the function)

`add_numbers()` stack frame begins. Math is calculated, and local variable `result` (15) is pushed.



```
|=================| <-- add_numbers() frame starts here
| int result = 15 | <-- SP
|=================| 
| Return Address  | 
|-----------------|
| arg: a = 5      | 
|-----------------|
| arg: b = 10     | 
|-----------------|
| int y = 10      | 
|-----------------|
| int x = 5       | 
|=================| 
```

---

Step 5: `return result;` (The Destruction)

Result (15) is saved to a CPU register. The Stack Pointer snaps back down to `main()`. The `add_numbers` frame is abandoned.



```
|                 |
| ( abandoned )   | 
|                 | 
|                 | 
|                 | 
|                 | 
|                 | 
|-----------------|
| int y = 10      | <-- SP is back in main()!
|-----------------|
| int x = 5       | 
|=================| 
```

---

Step 6: `int sum = ...` (Back in main)

Back in `main()`, the saved result (15) is pushed as the new variable `sum`, overwriting old, abandoned stack data.



```
|                 |
|-----------------|
| int sum = 15    | <-- SP
|-----------------|
| int y = 10      | 
|-----------------|
| int x = 5       | 
|=================| 
```

---

Step 7: `return 0;`

Program ends. Stack Pointer drops to the bottom, destroying the `main()` frame.



```
[ Stack is Empty ]
```




- **The Heap:** Where dynamic memory lives (e.g., `malloc`, `new`).
	- **The Mechanics:** Managed by the OS. When you call `malloc()` or `new`, the memory allocator searches for a contiguous block of free RAM large enough to hold your request, marks it as "in use," and returns a pointer to it.
	- **The Danger:** Memory Leaks. Because the CPU doesn't manage the Heap automatically, if you lose the pointer to that memory without calling `free()` or `delete`, that RAM is permanently locked up until your program is fully shut down by the OS.
    
- **The Data Segment:** Where **initialized** global and `static` variables live.
	- **The `.rodata` Segment (Read-Only Data)**
		- _Detail:_ Often lumped in with the Data segment, this is specifically for constants. If you write `char* str = "Hello";`, the pointer `str` lives on the stack, but the literal string `"Hello"` lives in `.rodata`. If you try to modify it (`str[0] = 'h';`), the OS will trigger a segfault because that memory page is physically locked at the hardware level.
	- **The `.data` Segment** 
		- _Detail:_ For **initialized** global and `static` variables (`static int x = 5;`). This data is baked directly into your `.exe` file. When the OS loads your program, it copies these bytes from your hard drive directly into RAM.

so:

```c
char *str = "beyza";
str = "emin"; //legal move
str[1]="a"; //illegal
```
because when you call `char *str = "beyza";` it stores the string "beyza" in rodata. you can change what str points to but you cannot touch the string in rodata.


- **The BSS Segment:** Where **uninitialized** (or zero-initialized) global and `static` variables live.
    - _Detail:_ For **uninitialized** global and `static` variables (`static int y;`). To save space on your hard drive, the compiler doesn't put thousands of zeros in the `.exe` file. It just leaves a note in the header: _"OS, allocate 4,000 bytes for BSS and zero it out before running `main()`."_
        

### The Three Faces of `static`

_The `static` keyword does completely different things depending on where you type it._

- **1. Local Static Variables (Inside a function)**
    
	- **The Hidden Flag:** How does the compiler know to only initialize it once? Behind the scenes, the compiler secretly creates a hidden boolean variable alongside your static variable.
    
	    - _Code:_ `static int count = 5;`
        
	    - _Compiler's reality:_ `if (!count_initialized) { count = 5; count_initialized = true; }`
        
	- **Thread Safety (C++11 onward):** In modern C++, local statics are guaranteed to be thread-safe ("Magic Statics"). If two threads hit the initialization line at the exact same nanosecond, the compiler injects invisible lock mechanisms to ensure it only initializes once.
        
- **2. Global Static Variables (Outside a function / File scope)**
    
	- **Internal Linkage Deep Dive:** When the assembler builds the `.o` file, it marks global static variables with a "Local" flag in the Symbol Table. When the Linker combines files, it is programmed to _completely ignore_ Local-flagged symbols.
        
- **3. Class Static Members (C++ Specific)**
    
	- **Static Member Variables:** They are practically global variables that are scoped inside the class's namespace for organizational purposes.
    
	- **Static Member Functions:** A normal class method (`void drive()`) secretly takes a hidden parameter: the `this` pointer, which points to the specific object instance. A `static` method (`static void getCount()`) **does not have a `this` pointer**. Therefore, a static method can never access non-static class variables, because it doesn't know _which_ object's variables to look at.
        

### `static` vs. `extern`



- **`extern` (External Linkage):** "I promise the compiler this variable exists somewhere else in another file. Let the Linker find it." (Used for sharing state across files).

- **Global `static` (Internal Linkage):** "Hide this variable. Do not let the Linker see it. It is mine alone." (Used for encapsulating state within one file).

- **Why they are opposites:** How `extern` broadens visibility, while `static` restricts it.



lets say we have two files named file A and file B:


fileA:
```c
int global_score = 100;          // Public Data
static int private_score = 50;   // Private Data (Hidden)

void print_score() { ... }       // Public Function
```

fileB:
```c
extern int global_score;         // Promise

int main() {
    global_score += 10;
    print_score();               // Called, but not defined here!
}
```

compiler runs on `fileA.c` and `fileB.c` completely separately. It builds two unconnected boxes (`.o` files), each with its own dictionary.

```
+------------------------------+       +------------------------------+
|           fileA.o            |       |           fileB.o            |
+------------------------------+       +------------------------------+
| SYMBOL TABLE:                |       | SYMBOL TABLE:                |
|                              |       |                              |
| [D] global_score  (Public)   |       | [U] global_score (Wanted!)   | 
| [d] private_score (Hidden)   |       | [U] print_score  (Wanted!)   |
| [T] print_score   (Public)   |       | [T] main         (Public)    |
+------------------------------+       +------------------------------+
```

here i need to explain what those letters mean:

the secret alphabet the compiler uses for the Symbol Table:

- **`D` (Public Data):** A normal global variable.
- **`d` (Private Data):** A `static` global variable (Hidden).
- **`T` (Text/Code):** A function you wrote.
- **`U` (Undefined):** An `extern` promise or a called function. It means _"I need this, Linker please find it."_

then, the Linker steps in. Its entire job is to look at every **`[U]`** (Undefined) symbol and draw a bridge to a matching **`[D]`** or **`[T]`** in another file.

Notice how `private_score` is trapped inside `fileA` because of that lowercase `d`.

```
+------------------------------+       +------------------------------+
|           fileA.o            |       |           fileB.o            |
+------------------------------+       +------------------------------+
| SYMBOL TABLE:                |       | SYMBOL TABLE:                |
|                              |       |                              |
| [D] global_score  <===================== [U] global_score           | (LINKED!)
|                              |       |                              |
| [T] print_score   <===================== [U] print_score            | (LINKED!)
|                              |       |                              |
| [d] private_score (IGNORED)  |       | [T] main                     | 
+------------------------------+       +------------------------------+
```

but what happens if `fileB` tries to use `private_score`?

If you wrote `extern int private_score;` inside `fileB.c`, the Linker tries to draw a bridge. But because `private_score` is marked with a lowercase **`[d]`**, the Linker treats it like a blank wall. The bridge fails, the Linker panics, and the build crashes.


```
+------------------------------+       +------------------------------+
|           fileA.o            |       |           fileB.o            |
+------------------------------+       +------------------------------+
| SYMBOL TABLE:                |       | SYMBOL TABLE:                |
|                              |       |                              |
| [d] private_score (Hidden)   |   X====== [U] private_score          | (ERROR!)
+------------------------------+       +------------------------------+
       Linker Error: "Undefined reference to 'private_score'"
```

and the final result:

Once all the `[U]` symbols are successfully connected to a public `[D]` or `[T]`, the Linker melts the boxes together into one single program. All the `[U]` promises are deleted because the actual memory addresses have been filled in.

```
+---------------------------------------------------------------------+
|                      FINAL EXECUTABLE (.exe)                        |
+---------------------------------------------------------------------+
| UNIFIED SYMBOL TABLE:                                               |
|                                                                     |
| [T] main            (The starting point)                            |
| [T] print_score     (Ready to be run)                               |
| [D] global_score    (Living in the public Data segment)             |
| [d] private_score   (Living in the Data segment, but kept private)  |
|                                                                     |
+---------------------------------------------------------------------+
```


###  Practical Proof



**1. The Build Pipeline**

- _(How to stop at the Preprocessor)_
    
    Run `gcc -E main.c -o main.i`. Open `main.i` to see all `#include` files copy-pasted and `#define` macros replaced with raw numbers.
    
- _(How to stop at the Compiler)_
    
    Run `gcc -S -O2 main.c`. Open `main.s` to see the raw Assembly code and how the compiler pre-calculates your math.
    
- _(How to stop at the Assembler)_
    
    Run `gcc -c main.c`. This generates `main.o`, the raw binary object file with the Symbol Table inside.
    

**2. Program Memory Layout**

- _(How to prove the BSS segment saves hard drive space)_
    
    Run `size a.out` (Linux/Mac) or `size program.exe` (Windows). Create a massive uninitialized global array (`int huge[1000000];`) and run it again. You will see the `bss` memory size jump, but your actual `.exe` file size won't change at all.
    
- _(How to see `.rodata`/`.data`)_
    
    Run `objdump -s -j .rodata ./a.out` or `objdump -s -j .data ./a.out` to dump the raw bytes sitting in each segment.
    
- _(How to watch the Heap)_
    
    Run `gdb ./a.out`, break after a `malloc()` call, and `print` the returned pointer. it'll be a heap address, distinct from anything on the stack.
    

**3. The Stack**

- _(How to watch the Stack Pointer move in real-time)_
    
    Run `gdb ./a.out`.
    
    Type `break main`, then `run`, then type `info registers rsp`. Write down the hexadecimal address. Step into a new function, type `info registers rsp` again, and you will see the memory address physically change as the new frame is pushed.
    

**4. `static`, `extern`, and Linkage**

- _(How to read the compiler's hidden Symbol Table)_
    
    Run `nm file.o` (Linux/Mac) or `dumpbin /symbols file.obj` (Windows).
    
- _(How to decipher the output)_
    
    - See a capital **`D`**? That is a normal global variable. The Linker can see it.
        
    - See a lowercase **`d`**? That is a `static` variable. The Linker is blocked from seeing it.
        
    - See a capital **`U`**? That is an `extern` promise. The Linker must go find it in another file.