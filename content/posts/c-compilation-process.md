+++
date = '2026-06-26T18:30:00+05:30'
draft = false
title = 'The C Compilation Process'
+++

# The C Compilation Process

Before a C program becomes an executable binary, it passes through a series of stages collectively known as the **compilation process**. Each stage transforms the source code into a different form, ultimately producing a program that the operating system or microcontroller can execute.

```text
          C FILES COMPILATION PROCESS              
                                                   
┌───────x                                          
│     .c│                                          
│ ┌───────x                        STAGE I         
│ │     .c│      ┌────────┐   ┌────────────┐       
│ │  ┌───────x ─►│COMPILER├──►│PREPROCESSOR├─┐     
└─│  │     .h│   └────────┘   └────────────┘ │     
  │  │       │                    #define    │     
  └──│       │                    #include   │     
     │       │                               │     
     └───────┘                               |     
                                             │     
        ┌───────x                  ┌───────x │     
        │  .asm │     STAGE II     │    .i │ │     
        │       │  ┌───────────┐   │       │ │     
 ┌───── ┤       ├──┤COMPILATION│◄──┤       │─┘     
 │      │       │  └───────────┘   │       │       
 │      └───────┘                  └───────┘       
 │                                                 
 │                                                 
 │       STAGE III      ┌───────x                  
 │     ┌───────────┐    │    .o │                  
 └───► │ ASSEMBLER ├────│  ┌───────x               
       └───────────┘    │  │    .o │               
                        │  │       │               
                        └──│       │─────────┐     
                           │       │         │     
                           └───────┘         │     
     ┌───────x                               │     
     │   .elf│             STAGE IV          │     
     │       │            ┌──────────┐       │     
     │       │ ◄───────── │  LINKER  │◄──────┘     
     │       │            └──────────┘             
     └───────┘                   ▲                 
     executable                  │                 
     linkable        ┌───────x   │                 
     format          │    .ld│   │                 
                     │       │ ──┘                 
                     │       │ linker              
                     │       │ script              
                     └───────┘                     
```


The compilation process consists of four major stages:

1. **Preprocessing**
2. **Compilation**
3. **Assembly**
4. **Linking**

Let's walk through each stage and understand what happens behind the scenes.

---

## Stage I — Preprocessing

The **preprocessor** is responsible for handling all directives that begin with the `#` symbol. It processes the source file and produces an expanded **translation unit** (a C source file after all preprocessing has been completed), typically with the `.i` extension.

During this stage, the following operations are performed:

* **Macro Expansion** – Replaces macros defined using `#define` with their corresponding values or expressions.
* **Header Inclusion** – Inserts the contents of header files (such as `<stdio.h>`) directly into the source file using the `#include` directive.
* **Comment Removal** – Removes all single-line (`//`) and multi-line (`/* ... */`) comments.
* **Conditional Compilation** – Evaluates directives such as `#ifdef`, `#ifndef`, `#if`, `#elif`, and `#endif` to determine which sections of code should be included in the translation unit.

The output of this stage is a preprocessed source file (typically with the .i extension).

Let's look at a simple example.

`main.h`
```c
#ifndef MAIN_H
#define MAIN_H

/* Declare the message array as external
   This tells the compiler that message 
   is defined in another source file */
   
extern char message[];

#endif

```

`main.c`
```c
#include "main.h"

char message[] = "Hello World";

int main(void) {
    char *ptr = message;
// This comments gets stripped out by the preprocessor, 
// so it won't appear in the preprocessed output
    while (*ptr != '\0') {
        ptr++;
    }
/* This is a multi-line comment also gets stripped out 
   by preprocessor.
*/
    return 0;
}

```

Generate the preprocessed output using GCC:

```bash
gcc -E main.c -o main.i
```

The generated main.i file will look similar to this:

```c
# 0 "main.c"
# 0 "<built-in>"
# 0 "<command-line>"
# 1 "main.c"
# 1 "main.h" 1          <-- Entering header file

extern char message[];
# 2 "main.c" 2          <-- Returning to main.c

char message[] = "Hello World";

int main(void) {
    char *ptr = message;

    while (*ptr != '\0') {
        ptr++;
    }

    return 0;
}
```
> The lines beginning with # are called line markers. They help the compiler and debugger keep track of the original source file and line numbers after preprocessing.

---

## Stage II — Compilation

The **compiler** takes the preprocessed source file (`.i`) and converts the C program into **assembly code** for the target processor architecture. The resulting assembly source file is typically saved with the `.s` extension.

> **Note:** Internally, a compiler performs several phases such as lexical analysis, parsing, semantic analysis, optimization, and code generation. Those topics deserve their own discussion, so for now we'll focus on the compiler's overall role in the build process.

At a high level, the compiler performs the following tasks:

* **Checks the source code for errors** such as missing semicolons, undeclared variables, incompatible data types, and incorrect function usage.
* **Code Optimization (optional)** – Improves the generated code by reducing execution time, minimizing code size, or eliminating redundant operations. The level of optimization depends on the compiler options used (for example, -O0, -O1, -O2, -O3, or -Os).
* **Assembly Code Generation** – Produces assembly instructions corresponding to the source code for the target architecture.

For example, if you accidentally write:

```c
int main(void)
{
    int number = "Hello";
    return 0;
}
```

the compiler will report a type mismatch because a string literal cannot be assigned to an `int`.

Or if you forget a semicolon:

```c
int main(void)
{
    int value = 10
    return 0;
}
```

the compiler will report a syntax error.

The output of this stage is an **assembly source file**.

Generate the assembly output using GCC:

```bash
gcc -S main.i -o main.s
```

> **Note:** The generated assembly file contains assembly instructions along with assembler directives (such as .text, .data, and .globl). Depending on the compiler, optimization level, target architecture, and operating system, your output may differ from the example shown below.

The generated Assembly file will look similar to this:

```asm
    .file "main.c"
    .text
    .globl message
    .data
    .align 8
message:
    .ascii "Hello World\0"
    .text
    .globl main
    .def main; .scl 2; .type 32; .endef
main:
    pushq %rbp
    movq %rsp, %rbp
    subq $48, %rsp
    call __main
    leaq message(%rip), %rax
    movq %rax, -8(%rbp)
    jmp .L2
.L3:
    addq $1, -8(%rbp)
.L2:
    movq -8(%rbp), %rax
    movzbl (%rax), %eax
    testb %al, %al
    jne .L3
    movl $0, %eax
    addq $48, %rsp
    popq %rbp
    ret
```
> **Note:** The example below was generated for an x86-64 target. If you're compiling for an ARM Cortex-M microcontroller, the generated assembly will use ARM Thumb instructions instead.


---

## Stage III — Assembling

The **assembler** converts the assembly source file (`.s`) into an **object file** (`.o` on Unix-like systems or `.obj` on Windows). This object file contains machine instructions that the processor can execute, along with additional information required by the linker.

During this stage, the assembler performs the following tasks:

* **Machine Code Generation** – Converts assembly instructions into their corresponding machine code (binary instructions) for the target processor.
* **Symbol Processing** – Records symbols (such as functions and global variables) that are either defined in or referenced by the source file.
* **Relocation Information** – Generates relocation entries for symbols whose final addresses are not yet known. These will be resolved during the linking stage.

At this point, the object file is **not yet an executable program**. References to functions or variables defined in other source files or libraries remain unresolved until the linker combines all object files.

Generate the object file using GCC:

```bash
gcc -c main.s -o main.o
```

The generated object file (`main.o`) is a **binary file**, so it cannot be viewed directly in a text editor.

Instead, you can inspect its contents using tools such as:

```bash
objdump -d main.o
```
```bash
❯ objdump -d .\main.o

.\main.o:     file format pe-x86-64


Disassembly of section .text:

0000000000000000 <main>:
   0:   55                      push   %rbp
   1:   48 89 e5                mov    %rsp,%rbp
   4:   48 83 ec 30             sub    $0x30,%rsp
   8:   e8 00 00 00 00          call   d <main+0xd>
   d:   48 8d 05 00 00 00 00    lea    0x0(%rip),%rax        # 14 <main+0x14>
  14:   48 89 45 f8             mov    %rax,-0x8(%rbp)
  18:   eb 05                   jmp    1f <main+0x1f>
  1a:   48 83 45 f8 01          addq   $0x1,-0x8(%rbp)
  1f:   48 8b 45 f8             mov    -0x8(%rbp),%rax
  23:   0f b6 00                movzbl (%rax),%eax
  26:   84 c0                   test   %al,%al
  28:   75 f0                   jne    1a <main+0x1a>
  2a:   b8 00 00 00 00          mov    $0x0,%eax
  2f:   48 83 c4 30             add    $0x30,%rsp
  33:   5d                      pop    %rbp
  34:   c3                      ret
  35:   90                      nop
  36:   90                      nop
  37:   90                      nop
  38:   90                      nop
  39:   90                      nop
  3a:   90                      nop
  3b:   90                      nop
  3c:   90                      nop
  3d:   90                      nop
  3e:   90                      nop
  3f:   90                      nop
```

The disassembly above shows the machine instructions generated by the assembler. Notice that the instructions are now represented as hexadecimal bytes (machine code) alongside their assembly mnemonics.

For example:

```bash
8:   e8 00 00 00 00          call   d <main+0xd>
d:   48 8d 05 00 00 00 00    lea    0x0(%rip),%rax
```

You may notice that the `call` instruction and the address used by `lea` contain placeholder values (`00 00 00 00`). This is because the assembler does **not** know the final memory addresses of functions and global variables.

Instead, it records relocation information inside the object file. During the **linking stage**, the linker resolves these references and replaces the placeholders with the correct addresses.

This is why an object file (`.o`) cannot be executed directly—it is only an intermediate file that still requires linking.

---

## Stage IV — Linking

The **linker** performs the final stage of the build process by combining one or more object files (`.o`) along with the required libraries to produce a single executable program.

During this stage, the linker performs the following tasks:

* **Symbol Resolution** – Resolves references to functions and global variables by matching their declarations with their corresponding definitions across object files and libraries.
* **Address Assignment** – Determines the final memory addresses of code and data sections.
* **Relocation** – Updates machine instructions and data references with the correct addresses determined during linking.
* **Executable Generation** – Produces the final executable file that can be loaded and executed by the operating system.


### Static vs. Dynamic Linking

There are two common ways a program can use libraries:

* **Static Linking** – Library code is copied directly into the executable, making it self-contained but increasing its size.
* **Dynamic Linking** – The executable contains references to shared libraries that are loaded by the operating system at runtime, reducing executable size and allowing multiple programs to share the same library.

> **Note:** Most bare-metal embedded systems use **static linking** because there is no operating system available to load shared libraries at runtime.

Generate the executable using GCC:

```bash id="9tw0rn"
gcc main.o -o main
```

At this point, the build process is complete. The executable now contains all the necessary code and data required to run the program.

_**Author's Note:** Technical content by the author. Edited and reworded with AI assistance._